---
title: "第5章：PCA・Kaiser有効次元による多重共線性監査"
---

## 1. 特徴量を増やすほど壊れる原因：多重共線性と置換効果

金融のマクロ指標やテクニカル指標は、ほとんどが互いに強く相関しています。たとえば、

- `sp500_return_1d`、`nasdaq100_return_1d`、`sox_semiconductor_return_1d`
- 5日リターン、10日リターン、20日リターン、20日ボラティリティ、高値からの乖離率

これらを92個並べて回帰モデルやツリーモデルに投入すると、AFML 第8章で指摘される **置換効果（Substitution Effect）** が発生します。

1. **線形モデル（Ridge / Lasso）での不安定化**：共相関の高い2変数 $x_1, x_2$ に対し、学習期間だけの偶然ノイズを拾って $+10 \cdot x_1 - 9.8 \cdot x_2$ のような巨大な逆符号ペアが形成され、OOS で相関がわずかに崩れた瞬間に予測が爆発する。
2. **重要度指標の希釈**：本来1つの強いファクター（例：米国ハイテク株リスクオン）が5つの派生特徴に分裂しているため、1変数あたりの重要度が $1/5$ に希釈され、単体のノイズ指標より低く見えてしまう。

この構造的な冗長性を診断する最も確実なツールが、**学習期間のみでフィットした PCA（主成分分析）＋ Kaiser 基準（固有値 $\lambda_k \ge 1.0$）＋ 上位/下位主成分 OOS プローブ対決** です。

---

## 2. Kaiser 有効次元と「上位PC vs 下位PC」OOS プローブの実装

標準化された $p$ 次元特徴量行列 $X_{\text{tr}}$ の相関行列に対して固有値分解を行うと、全固有値の合計は特徴量数 $p$ に等しくなります（平均固有値は `1.0`）。

- **Kaiser 基準（$\lambda_k \ge 1.0$）**：元の特徴量1本分以上の分散を説明する独立軸の数 $K$ を「有効次元数（Effective Dimensions）」とみなす。
- **上位 $K$ 主成分プローブ（`Top-K`）**：市場の主要な共通因子（第1〜第 $K$ 主成分）だけで Ridge を学習し、OOS Pearson IC を測定する。
- **下位 $p - K$ 主成分プローブ（`Bottom-K`）**：固有値 $1.0$ 未満の微小分散軸（ノイズ・スプレッド残差）だけで Ridge を学習し、OOS Pearson IC を測定する。

もし真の構造的エッジが存在するなら、**`pca_top_k_oos_ic > pca_bottom_k_oos_ic` かつ `pca_top_k_oos_ic > 0.0`（`edge_on_top_pcs == True`）** になるはずです。逆に下位PC側にしか相関が出ないモデルは、微小な共線性スプレッドに過学習している危険信号です。

```python
from __future__ import annotations

import math
from typing import Any

import numpy as np
from sklearn.decomposition import PCA
from sklearn.linear_model import Ridge
from sklearn.preprocessing import StandardScaler


def _safe_pearson(y_true: np.ndarray, y_pred: np.ndarray) -> float:
    if len(y_true) < 2 or float(np.std(y_true)) < 1e-12 or float(np.std(y_pred)) < 1e-12:
        return 0.0
    val = float(np.corrcoef(y_true, y_pred)[0, 1])
    return 0.0 if math.isnan(val) else round(val, 4)


def run_pca_kaiser_orthogonality_probe(
    X_train: np.ndarray,
    y_train: np.ndarray,
    X_oos: np.ndarray,
    y_oos: np.ndarray,
    probe_alpha: float = 10.0,
) -> dict[str, Any]:
    """学習窓のみで StandardScaler + PCA をフィットし、Kaiser有効次元と上位/下位PCのOOS ICを比較する。"""
    scaler = StandardScaler()
    X_tr_scaled = scaler.fit_transform(np.asarray(X_train, dtype=float))
    X_te_scaled = scaler.transform(np.asarray(X_oos, dtype=float))
    y_tr = np.asarray(y_train, dtype=float)
    y_te = np.asarray(y_oos, dtype=float)
    n_features = X_tr_scaled.shape[1]

    # PCAも必ず学習データのみで fit する（OOS共分散のリーク禁止）
    pca = PCA()
    Z_tr = pca.fit_transform(X_tr_scaled)
    Z_te = pca.transform(X_te_scaled)

    eigenvalues = pca.explained_variance_
    kaiser_dims = int(np.sum(eigenvalues >= 1.0))

    if n_features >= 2:
        top_k = max(1, min(kaiser_dims if kaiser_dims > 0 else 1, n_features - 1))
        top_probe = Ridge(alpha=probe_alpha).fit(Z_tr[:, :top_k], y_tr)
        top_oos_ic = _safe_pearson(y_te, top_probe.predict(Z_te[:, :top_k]))

        bot_probe = Ridge(alpha=probe_alpha).fit(Z_tr[:, top_k:], y_tr)
        bot_oos_ic = _safe_pearson(y_te, bot_probe.predict(Z_te[:, top_k:]))
    else:
        top_k = 1
        top_oos_ic = 0.0
        bot_oos_ic = 0.0

    return {
        "total_features": n_features,
        "kaiser_effective_dims": kaiser_dims,
        "effective_dim_ratio": round(kaiser_dims / max(n_features, 1), 4),
        "top_k_count": top_k,
        "pca_top_k_oos_ic": top_oos_ic,
        "pca_bottom_k_oos_ic": bot_oos_ic,
        "edge_on_top_pcs": bool(top_oos_ic > bot_oos_ic and top_oos_ic > 0.0),
        "top5_explained_variance": [
            round(float(v), 4) for v in pca.explained_variance_ratio_[:5]
        ],
    }
```

---

## 3. 実パネル6系統での PCA・Kaiser 監査結果

日本株マクロパネル4通りと米国株クロスマーケットパネル2通りの計6系統に対し、上記監査を実行した実測結果がこちらです。

| パネル・ターゲット | 特徴量数 | Kaiser有効次元 | 有効次元率 | PCA Top-K OOS IC | PCA Bottom-K OOS IC | `edge_on_top_pcs` | 診断 |
|---|---:|---:|---:|---:|---:|---:|---|
| 日本株 `T3_jp_top20` (`Full92`) | 92 | **`15 / 92`** | `16.3%` | `-0.0588` | `-0.0480` | `False` | 静的構造なし（92本中77本が冗長） |
| 日本株 `T3_jp_top20` (`Core22`) | 22 | **`7 / 22`** | `31.8%` | `-0.1050` | `+0.0651` | `False` | 上位PC負・下位PC正（静的過学習） |
| 日本株 `T2_us_macro` (`Full92`) | 92 | **`15 / 92`** | `16.3%` | **`+0.0835`** | `-0.0491` | **`True`** | 上位PCに本物のエッジ・下位77本はノイズ |
| 日本株 `T2_us_macro` (`Core22`) | 22 | **`7 / 22`** | `31.8%` | **`+0.0602`** | `+0.0256` | **`True`** | 圧縮後も上位PC優位を維持 |
| 米国株 `Long-Only` | 39 | **`11 / 39`** | `28.2%` | `+0.0385` | `+0.0790` | `False` | 下位PC依存（レジーム劣化リスク高） |
| 米国株 `Long+Short` | 40 | **`11 / 40`** | `27.5%` | `-0.0229` | `-0.0225` | `False` | 静的分割では崩壊（WF再学習必須） |

この表から、実務で極めて再現性の高い3つの法則が見えてきます。

### 法則 1：金融特徴量の約70〜84%は共線性の重複である

`Full92`（92特徴量）を投入しても独立な主成分は **`15` 次元（16.3%）** しかなく、米国株の39〜40特徴量でも **`11` 次元（約28%）**、厳選したはずの `Core22` でさえ **`7` 次元（31.8%）** です。この状態のまま正則化の弱いモデルや深い決定木を回せば、残りの7割以上の冗長次元がノイズを吸収し、学習期間だけの偽の相関を量産します。

### 法則 2：`T2_us_macro Full92` が見せた「上位 `+0.0835` vs 下位 `-0.0491`」の鮮明な分離

`T2_us_macro Full92` では、92特徴量そのままの静的 OOS IC は `+0.0311` まで落ち込んでいました。ところが PCA で上位15主成分と下位77主成分に割ってみると、

- **上位15主成分（`Top-K`）**：OOS IC **`+0.0835`**
- **下位77主成分（`Bottom-K`）**：OOS IC **`-0.0491`**

と完全に分離したのです。つまり、**本物の予測シグナルは上位の共通マクロ因子に集中しており、下位77次元の細かなラグ・ボラティリティ派生指標が OOS の予測力を `-0.0491` ぶん足を引っ張っていた** ことが数学的に証明されました。実際、第1章で見た通り `Full92` を `Core22` に落とすだけで、Walk-Forward OOS IC は `0.0791 -> 0.1427` へ劇的に回復しています。

### 法則 3：`edge_on_top_pcs == False` を無視して本番投入してはいけない

`T3_jp_top20 Core22` や米国株 `Long-Only` のように、`Bottom-K` のほうが `Top-K` より OOS IC が高く見える（`edge_on_top_pcs = False`）ケースでは、モデルが「主要な市場ファクター」ではなく「学習期間特有の細かい変数間スプレッド」に賭けています。こうしたモデルは、第3章で見た通り実損益の Uniform-Cap 監査にかけると次々と REJECT になります。

---

## 4. PCA で全体像を掴んだら、次は「個別の犯人（特徴量）」を特定する

PCA・Kaiser 監査は「この特徴量セットに冗長性がどれだけあり、エッジが上位の共通因子に乗っているか」をマクロ視点で診断する優れたX線検査です。

ただし PCA だけでは、22個の特徴量のうち **「具体的にどの変数を残し、どの変数を削除すべきか」** という外科手術のメスまでは下ろせません（主成分は全特徴量の線形結合であり、そのまま削ると経済的解釈が失われやすいため）。

次章では、AFML 第8章の中核である **MDI（インサンプル重要度）vs OOS Permutation MDA（アウトオブサンプル順列重要度）** を実装し、`Core22` の中で実際に予測を壊していた **4つの過学習ノイズ特徴量** を名指しで特定・切除していきます。
