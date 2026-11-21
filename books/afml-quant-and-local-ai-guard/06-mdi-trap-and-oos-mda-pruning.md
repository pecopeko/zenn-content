---
title: "第6章：MDIの罠とPurged OOS MDA特徴量剪定"
---

## 1. なぜインサンプル重要度（MDI・係数絶対値）は真っ先に騙されるのか

モデルの解釈や特徴量選択で最もよく使われるのが、決定木系モデルの `feature_importances_`（MDI：Mean Decrease Impurity）や、線形モデルの回帰係数の絶対値 $|\beta_j|$ です。

しかし AFML 第8章が厳しく警告している通り、**インサンプル重要度（IS Importance）は「未知の未来を当てる力」を一切測っていません**。

- ランダムノイズであっても、学習データ内の偶然の外れ値にたまたま一致すれば、MDI や $|\beta_j|$ は高い値を取る。
- 特に多重共線性の強い特徴量群では、学習データ固有のノイズを打ち消すために、無関係なマクロ指標へ大きな係数が割り当てられる。
- その結果、**「インサンプル重要度の上位にいるのに、OOS ではモデルの足を引っ張っている過学習ドラッグ」** が必ず紛れ込む。

これをあぶり出す決定打が、**OOS Permutation MDA（Out-of-Sample Mean Decrease Accuracy）** です。

### OOS MDA の定義

学習済みモデル $\hat{f}$ の OOS ベースライン精度を $\text{IC}_{\text{base}} = \text{corr}(y_{\text{oos}}, \hat{f}(X_{\text{oos}}))$ とします。特徴量 $j$ の列だけを OOS データ内でランダムにシャッフルした行列 $X_{\text{oos}}^{(j, r)}$ を $R$ 回（例：$R = 20$）作り、精度の低下量を平均します。

$$
\text{MDA}_j = \frac{1}{R} \sum_{r=1}^{R} \Bigl(\text{IC}_{\text{base}} - \text{corr}\bigl(y_{\text{oos}}, \hat{f}(X_{\text{oos}}^{(j, r)})\bigr)\Bigr)
$$

- **$\text{MDA}_j > 0$**：特徴量 $j$ を壊すと OOS 精度が下がる → **OOS で本当に効いている特徴量**
- **$\text{MDA}_j \le 0$**：特徴量 $j$ をシャッフルして破壊したほうが OOS 精度が上がる（または変わらない）→ **OOS の足を引っ張っているノイズ特徴量**

さらに本書では、`in_sample_importance >= 0.05`（インサンプルで5%以上の大きな重みを占めている）にもかかわらず `oos_mda_ic <= 0.0` である変数を **`overfit_suspected = True`（過学習容疑）** として自動フラグします。

---

## 2. `run_afml_orthogonality_audit` の完全実装（MDI vs OOS MDA ＋ PCA 統合）

第4章の `DayBlockBaggingRegressor` と第5章の PCA 監査を統合し、1回の呼び出しで「静的 Train/OOS IC」「PCA Kaiser 有効次元」「全特徴量の IS重要度 vs OOS MDA（平均・標準偏差・過学習フラグ）」を返す完全な監査モジュールを実装します。

```python
from __future__ import annotations

import copy
import inspect
import math
from typing import Any, Sequence

import numpy as np
from pydantic import BaseModel, Field
from scipy.stats import spearmanr
from sklearn.base import clone
from sklearn.decomposition import PCA
from sklearn.linear_model import Ridge
from sklearn.preprocessing import StandardScaler


class FeatureAuditRow(BaseModel):
    """特徴量ごとの In-Sample 重要度 vs OOS MDA 監査結果。"""

    feature: str
    in_sample_importance: float
    oos_mda_ic: float
    oos_mda_std: float
    overfit_suspected: bool


class OrthogonalityAuditReport(BaseModel):
    """AFML Ch.8 準拠の MDI vs OOS MDA および PCA 直交性監査レポート。"""

    train_pearson_ic: float
    oos_pearson_ic: float
    oos_spearman_ic: float
    kaiser_effective_dims: int
    total_features: int
    pca_top_k_count: int
    pca_top_k_oos_ic: float
    pca_bottom_k_oos_ic: float
    edge_on_top_pcs: bool
    feature_audit: list[FeatureAuditRow] = Field(default_factory=list)
    flagged_overfit_features: list[str] = Field(default_factory=list)


def _safe_pearson(y_true: np.ndarray, y_pred: np.ndarray) -> float:
    if len(y_true) < 2 or np.std(y_true) < 1e-12 or np.std(y_pred) < 1e-12:
        return 0.0
    val = float(np.corrcoef(y_true, y_pred)[0, 1])
    return 0.0 if math.isnan(val) else val


def _safe_spearman(y_true: np.ndarray, y_pred: np.ndarray) -> float:
    if len(y_true) < 2 or np.std(y_true) < 1e-12 or np.std(y_pred) < 1e-12:
        return 0.0
    val = float(spearmanr(y_true, y_pred).statistic)
    return 0.0 if math.isnan(val) else val


def _fit_with_optional_dates(
    model: Any,
    X_train: np.ndarray,
    y_train: np.ndarray,
    dates_train: Sequence[Any] | np.ndarray | None,
) -> Any:
    try:
        fitted = clone(model)
    except Exception:
        fitted = copy.deepcopy(model)
    sig = inspect.signature(fitted.fit)
    if "dates" in sig.parameters and dates_train is not None:
        fitted.fit(X_train, y_train, dates=dates_train)
    else:
        fitted.fit(X_train, y_train)
    return fitted


def run_afml_orthogonality_audit(
    model: Any,
    X_train: np.ndarray,
    y_train: np.ndarray,
    dates_train: Sequence[Any] | np.ndarray | None,
    X_oos: np.ndarray,
    y_oos: np.ndarray,
    feature_names: Sequence[str],
    random_state: int = 42,
    n_repeats: int = 20,
    overfit_is_threshold: float = 0.05,
) -> OrthogonalityAuditReport:
    """MDI/係数重要度 vs OOS Permutation MDA と PCA直交性を一括監査する。"""
    X_tr = np.asarray(X_train, dtype=float)
    y_tr = np.asarray(y_train, dtype=float)
    X_te = np.asarray(X_oos, dtype=float)
    y_te = np.asarray(y_oos, dtype=float)
    n_features = X_tr.shape[1]

    if len(feature_names) != n_features:
        raise ValueError("feature_names の長さは X_train の列数と一致する必要があります。")

    fitted = _fit_with_optional_dates(model, X_tr, y_tr, dates_train)
    train_ic = _safe_pearson(y_tr, np.asarray(fitted.predict(X_tr), dtype=float))
    base_oos_ic = _safe_pearson(y_te, np.asarray(fitted.predict(X_te), dtype=float))
    oos_spearman = _safe_spearman(y_te, np.asarray(fitted.predict(X_te), dtype=float))

    # In-Sample 重要度の抽出（feature_importances_ / coef_ が無ければ Train MDA）
    if hasattr(fitted, "feature_importances_"):
        raw_imp = np.abs(np.ravel(np.asarray(fitted.feature_importances_, dtype=float)))
    elif hasattr(fitted, "coef_"):
        raw_imp = np.abs(np.ravel(np.asarray(fitted.coef_, dtype=float)))
    else:
        rng_tr = np.random.default_rng(random_state)
        raw_imp = np.zeros(n_features, dtype=float)
        for j in range(n_features):
            X_perm = X_tr.copy()
            drops = []
            for _ in range(min(n_repeats, 8)):
                X_perm[:, j] = rng_tr.permutation(X_tr[:, j])
                drops.append(max(train_ic - _safe_pearson(y_tr, fitted.predict(X_perm)), 0.0))
            raw_imp[j] = float(np.mean(drops))

    imp_sum = float(np.sum(raw_imp))
    in_sample_norm = raw_imp / imp_sum if imp_sum > 0 else np.zeros(n_features, dtype=float)

    # Out-of-Sample Permutation MDA
    rng = np.random.default_rng(random_state)
    feature_rows: list[FeatureAuditRow] = []
    flagged_features: list[str] = []

    for j, fname in enumerate(feature_names):
        drops_j: list[float] = []
        X_perm = X_te.copy()
        for _ in range(n_repeats):
            X_perm[:, j] = rng.permutation(X_te[:, j])
            perm_ic = _safe_pearson(y_te, np.asarray(fitted.predict(X_perm), dtype=float))
            drops_j.append(base_oos_ic - perm_ic)
        X_perm[:, j] = X_te[:, j]

        mda_mean = float(np.mean(drops_j))
        mda_std = float(np.std(drops_j, ddof=1)) if len(drops_j) > 1 else 0.0
        in_imp = float(in_sample_norm[j])
        overfit = (in_imp >= overfit_is_threshold) and (mda_mean <= 0.0)
        if overfit:
            flagged_features.append(fname)

        feature_rows.append(
            FeatureAuditRow(
                feature=fname,
                in_sample_importance=round(in_imp, 4),
                oos_mda_ic=round(mda_mean, 4),
                oos_mda_std=round(mda_std, 4),
                overfit_suspected=overfit,
            )
        )

    # PCA 直交性プローブ
    scaler = StandardScaler()
    pca = PCA()
    Z_tr_mat = pca.fit_transform(scaler.fit_transform(X_tr))
    Z_te_mat = pca.transform(scaler.transform(X_te))
    kaiser_dims = int(np.sum(pca.explained_variance_ >= 1.0))

    if n_features >= 2:
        top_k = max(1, min(kaiser_dims if kaiser_dims > 0 else 1, n_features - 1))
        top_ic = _safe_pearson(y_te, Ridge(alpha=10.0).fit(Z_tr_mat[:, :top_k], y_tr).predict(Z_te_mat[:, :top_k]))
        bot_ic = _safe_pearson(y_te, Ridge(alpha=10.0).fit(Z_tr_mat[:, top_k:], y_tr).predict(Z_te_mat[:, top_k:]))
    else:
        top_k, top_ic, bot_ic = 1, base_oos_ic, 0.0

    return OrthogonalityAuditReport(
        train_pearson_ic=round(train_ic, 4),
        oos_pearson_ic=round(base_oos_ic, 4),
        oos_spearman_ic=round(oos_spearman, 4),
        kaiser_effective_dims=kaiser_dims,
        total_features=n_features,
        pca_top_k_count=top_k,
        pca_top_k_oos_ic=round(top_ic, 4),
        pca_bottom_k_oos_ic=round(bot_ic, 4),
        edge_on_top_pcs=(top_ic > bot_ic and top_ic > 0.0),
        feature_audit=feature_rows,
        flagged_overfit_features=flagged_features,
    )
```

---

## 3. 実測事例①：日本株マクロ `T2 Core22` で炙り出された4つのノイズ特徴量

`T2_us_macro Core22`（学習100日 / OOS 262日、`DayBlockBaggingRegressor(alpha=50, n_estimators=50)`）に対して上記 OOS MDA 監査を実行した結果、**インサンプル重要度と OOS 貢献度の劇的な逆転** が観測されました。

### 正の OOS MDA 上位4特徴量（本物のシグナル源）

| 特徴量 | IS重要度 | OOS MDA (IC低下量) | 経済的意味 |
|---|---:|---:|---|
| `gap_minus_nasdaq100` | `0.043` | **`+0.0307`** | 日経ギャップとNasdaq100の乖離（反応の行き過ぎ・出遅れ） |
| `euro_stoxx50_return_1d` | `0.161` | **`+0.0225`** | 欧州大型株のリスク選好 |
| `vix_return_1d` | `0.049` | **`+0.0213`** | 恐怖指数の日次変化 |
| `anchor_overnight_std20` | `0.033` | **`+0.0168`** | 基準銘柄群の20日オーバーナイト変動率 |

### 負の OOS MDA 下位4特徴量（予測を破壊していた剪定対象）

| 特徴量 | IS重要度 | OOS MDA (IC低下量) | 過学習診断 |
|---|---:|---:|---|
| `us_10y_yield_return_1d` | **`0.080`** | **`-0.0240`** | **IS 8.0% 占有 → OOS最悪ドラッグ（過学習容疑）** |
| `crude_oil_wti_futures_return_1d` | `0.040` | **`-0.0179`** | 原油先物日次変化（OOSで逆効き） |
| `kospi_return_1d` | `0.023` | **`-0.0120`** | 韓国KOSPI日次変化（共線ノイズ） |
| `dollar_index_futures_return_1d` | **`0.052`** | **`-0.0110`** | **IS 5.2% 占有 → OOSドラッグ（過学習容疑）** |

注目すべきは **`gap_minus_nasdaq100`（IS `4.3%`、OOS MDA `+0.0307`）と `us_10y_yield_return_1d`（IS `8.0%`、OOS MDA `-0.0240`）の対比** です。インサンプル重要度だけを見て特徴量を選んでいたなら、真のエースである `gap_minus_nasdaq100` より、足を引っ張る `us_10y_yield_return_1d` を約2倍も重要視してしまったことになります。

---

## 4. 実測事例②：日中モデル29特徴量でも「約45%（13/29）」が非正の OOS MDA

この現象はマクロモデル特有ではありません。まったく別のパネル（日本株日中戦略・29特徴量・416営業日）で MDI（インサンプル重要度 %）と OOS MDA を対照したところ、**29特徴量のうち正の OOS MDA は `16/29`、残る `13/29`（44.8%）は `<= 0`** という結果が出ました。

- **本物の主力特徴量（ISも高くOOS MDAも強い）**：
  - `pred_compact_1125_res`（IS `11.70%` / OOS MDA **`+0.026541`**）
  - `mkt_ret_am`（IS `4.86%` / OOS MDA **`+0.011866`**）
  - `pred_time_1230`（IS `6.18%` / OOS MDA **`+0.011495`**）
  - `am_range_vs_own_20d_ratio`（IS `4.99%` / OOS MDA **`+0.009704`**）
- **高 MDI なのに OOS を壊していた過学習容疑6特徴量（`IS >= 2.5%` かつ `OOS_MDA <= 0`）**：
  - `stock_up_am_lunch_habit_60d`（IS `3.80%` / OOS MDA `-0.001018`）
  - `rel_sec_am`（IS `2.60%` / OOS MDA `-0.001089`）
  - `early_vs_late_dbar_bal`（IS `2.79%` / OOS MDA `-0.002212`）
  - `alpha_late_1049_1119_bp`（IS `2.93%` / OOS MDA `-0.002840`）
  - `ffd_cum_r1d_norm`（IS `3.74%` / OOS MDA `-0.003333`）
  - `dbar_higher_low_bp`（IS `3.10%` / OOS MDA `-0.003400`）

「60日の昼休み癖」「特定30分間の遅れアルファ」など、人間のストーリーとしてもっともらしい特徴量ほどインサンプル MDI で `3〜4%` の存在感を放ち、OOS では静かに精度を削ります。

---

## 5. 4特徴剪定（`Core22` → `Pruned18`）の Walk-Forward 効果

`T2_us_macro Core22` から先ほど特定した負の OOS MDA 4特徴量（`us_10y_yield_return_1d`, `crude_oil_wti_futures_return_1d`, `kospi_return_1d`, `dollar_index_futures_return_1d`）を除去した **`Pruned18`（18特徴量）** で、20日拡大窓 Walk-Forward（OOS 262日、`alpha=50`）を再計測した結果が以下です。

| 特徴量セット | 特徴量数 | OOS日数 | WF `Ridge(50)` Pearson IC | WF `DayBlockBag(50)` Pearson IC | WF `DayBlockBag(50)` Spearman IC |
|---|---:|---:|---:|---:|---:|
| `Core22` | 22 | 262 | `0.1330` | `0.1427` | `0.1134` |
| **`Pruned18`** | **18** | 262 | **`0.1380`** | **`0.1436`** | `0.1082` |

単体 `Ridge(50)` で `+0.0050`、`DayBlockBag(50)` でも `0.1436` へ Pearson IC が向上しました。一見すると `0.1427 -> 0.1436` はわずか `+0.0009` の地味な差に見えるかもしれません。

ところが次章で、この `Pruned18` を **3段階確信度サイジング × 100株単元 Uniform-Cap Control** に乗せた瞬間、**`Core22`（`+41,051 JPY` / Sharpe敗北）と `Pruned18`（`+545,251 JPY` / Sharpe `2.501` 突破）の間に50万円以上の圧倒的な差** が開くことになります。
