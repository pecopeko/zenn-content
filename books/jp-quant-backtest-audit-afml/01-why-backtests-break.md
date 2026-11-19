---
title: "第1章：なぜ日本株バックテストは実運用で崩れるのか"
free: true
---

## 本書の目的と対象読者

Python と scikit-learn で日本株や米国株のシステムトレードを研究していると、誰もが一度は「バックテストでは年率シャープ 3.0 超えだったのに、実弾を入れた翌月から右肩下がりになる」という壁にぶつかります。

その原因の多くは、モデルの表現力不足ではありません。**バックテストの評価基盤そのものに、偽陽性（False Positive）を大量生産する構造的な穴が空いている** ことが原因です。

本書では、Marcos López de Prado の『Advances in Financial Machine Learning（AFML）』の核心手法（標本一意性・日次ブロックバギング・PCA直交性・OOS順列重要度 MDA）に、**日本株特有の100株単元丸め・非複利固定資本会計・対一律縮小対照（Uniform-Cap Control）** を組み合わせた実戦的なバックテスト監査パイプラインを、コピペ可能な Python コードとともに一気通貫で構築します。

本書の第1章・第2章は無料公開しています。実際に使用する実パネル検証データ（日本株オーバーナイト 269〜376営業日、米国株クロスマーケット 370〜371営業日、翌朝決済履歴 225日/27日、日中戦略 416日）の具体的な失敗ログと成功ログを隠さず提示します。

---

## 1. 静的ホールドアウト崩壊の実測：Train IC `0.7792` → OOS IC `-0.0617`

まず、高次元特徴量をそのまま線形・ツリー系モデルへ流し込んだときに起きる典型的な崩壊を、実パネル（`2025-04-01 .. 2026-10-15`、376営業日）の測定値で直視しましょう。

引け（15:25 JST 締切）に利用可能なグローバルマクロ・先物・為替・ボラティリティ・ギャップ指標のすべて（92特徴量：`Full92`）と、中核22指標に絞った `Core22` を使い、前半約100日を学習、後半約262〜272日を Out-of-Sample（OOS）とした静的ホールドアウト監査の結果が以下です（正則化 `Ridge(alpha=50)` ベースの `DayBlockBaggingRegressor`）。

| 予測ターゲット | 特徴量セット | 学習 / OOS 日数 | 静的 Train IC | 静的 OOS IC | 20日拡大窓 WF OOS IC |
|---|---|---:|---:|---:|---:|
| `T3_jp_top20`（日本株上位20翌朝リターン） | `Full92`（92特徴） | 104 / 272 | **`0.7792`** | **`-0.0617`** | `0.0730` |
| `T3_jp_top20`（日本株上位20翌朝リターン） | `Core22`（22特徴） | 104 / 272 | `0.5818` | `-0.0503` | `0.0503` |
| `T2_us_macro`（米国マクロ連動リターン） | `Full92`（92特徴） | 100 / 262 | `0.7549` | `0.0311` | `0.0791` |
| `T2_us_macro`（米国マクロ連動リターン） | `Core22`（22特徴） | 100 / 262 | `0.5524` | **`0.0886`** | **`0.1427`** |

ここには金融機械学習の残酷な現実が凝縮されています。

1. **サンプル数 $N \approx 100$ に対して特徴量 $p = 92$ を投入すると、L2正則化（`alpha=50`）を掛けても学習期間は `IC = 0.7792` まで簡単にフィットしてしまう**。しかし OOS へ出た瞬間 `-0.0617` へ反転する。
2. **特徴量を 92 → 22 に削るだけで、同じモデルの Walk-Forward OOS IC が `0.0791 -> 0.1427`（約1.8倍、単体Ridgeでは `0.0551 -> 0.1330` と2.4倍）に跳ね上がる**。金融データでは「情報を足す」より「ノイズと共線性を削る」ほうが遥かにリターンに効く。
3. **静的分割で一度学習した重みを固定し続けると、レジームシフトで容易に符号が反転する**。必ず時系列の拡大窓（Expanding Window）またはローリング窓で Walk-Forward 検証を行わなければならない。

---

## 2. 厳格ルール①：`.fillna(0)` 全面禁止と明示的除外

バックテストのバグで最も静かに検証を汚染するのが、pandas での安易な `df.fillna(0)` です。

### なぜ `.fillna(0)` が危険なのか

- **ターゲット列の欠損を `0.0` で埋めた場合**：米国休場日（例：感謝祭や独立記念日）や上場廃止・値付かずの銘柄を「リターン `0 bp` の取引」として学習・評価してしまう。これは実在しない「分散ゼロの無風日」をデータセットに水増しする行為であり、ボラティリティと IC の双方を歪める。
- **特徴量列の欠損を `0.0` で埋めた場合**：標準化前であっても `0.0` は「前日比ちょうど 0.00%（完全横ばい）」という特定の経済状態を意味する。システム障害や取得失敗の欠損を「市場は動かなかった」と偽装することになる。

### 欠損列・定数列を一切偽装せず除外・記録する抽出関数

監査コードでは、以下のように欠損を1件でも含む特徴量列や分散ゼロの定数列を **無言で埋めず、明示的に `dropped_cols` へ記録して除外** します。

```python
from __future__ import annotations

from typing import Any
import numpy as np


def extract_non_null_matrix(
    table: Any,
    row_indices: list[int],
    candidate_cols: list[str],
) -> tuple[np.ndarray, list[str], list[str]]:
    """PyArrow Table等から、欠損ゼロ・非定数の特徴量行列だけを厳密に抽出する。

    .fillna(0) を一切行わず、欠損を含む列や分散ゼロの列は dropped_cols に明示記録する。
    """
    kept_cols: list[str] = []
    dropped_cols: list[str] = []
    arrays: list[np.ndarray] = []

    for col in candidate_cols:
        if col not in table.column_names:
            dropped_cols.append(col)
            continue

        raw_vals = [table[col][i].as_py() for i in row_indices]
        has_missing = any(
            v is None or (isinstance(v, float) and np.isnan(v)) for v in raw_vals
        )
        if has_missing:
            dropped_cols.append(col)
            continue

        arr = np.asarray(raw_vals, dtype=float)
        if float(np.std(arr)) <= 1e-12:
            dropped_cols.append(col)
            continue

        kept_cols.append(col)
        arrays.append(arr)

    matrix = (
        np.column_stack(arrays)
        if arrays
        else np.empty((len(row_indices), 0), dtype=float)
    )
    return matrix, kept_cols, dropped_cols
```

ターゲット側（`label_T2_bp` 等）についても同様に、`None` / `NaN` の行を事前に除外し、**除外した行数（例：米国休場に伴う14行の欠損）を監査レポートに必ず明記** します。実際、本書の米国株パネルでは `2025-04-23` の1行で翌朝決済価格が欠損していたため、ゼロ埋めせず明示的に `371日 -> 370日` として除外処理を行っています。

---

## 3. 厳格ルール②：15:25 JST 因果タイムスタンプ監査

日本株の引けエントリー戦略（大引け 15:30 JST、発注締切 15:25 JST = `06:25 UTC`）で致命的なのが、**欧州・米国市場の引け値や日本の大引け確定値を誤って特徴量に混ぜてしまう未来リーク（Look-ahead Bias）** です。

これを人手の目視ではなく機械的に保証するため、データ生成パイプライン側で各データソースの確定時刻を `audit_<source>_available_at` 列（タイムゾーン付き `datetime`）として残し、バックテスト実行前に毎回アサートします。

```python
from datetime import datetime
from typing import Any


def audit_causality_cutoff_utc(
    table: Any,
    date_col: str = "date",
    cutoff_hour_utc: int = 6,
    cutoff_minute_utc: int = 25,
) -> dict[str, int]:
    """全 audit_*_available_at 列が各営業日の 15:25 JST (06:25 UTC) 以内であることを検証する。"""
    checked_cols = 0
    violations = 0

    raw_dates = table[date_col].to_pylist()
    for col in table.column_names:
        if not (col.startswith("audit_") and col.endswith("_available_at")):
            continue
        checked_cols += 1
        for d_raw, avail_ts in zip(raw_dates, table[col].to_pylist()):
            if avail_ts is None:
                continue
            if not isinstance(d_raw, datetime) or not isinstance(avail_ts, datetime):
                raise TypeError(f"{col}: datetime型が必要です。")
            cutoff = d_raw.replace(
                hour=cutoff_hour_utc,
                minute=cutoff_minute_utc,
                second=0,
                microsecond=0,
                tzinfo=avail_ts.tzinfo,
            )
            if avail_ts > cutoff:
                violations += 1

    return {"checked_audit_columns": checked_cols, "causality_violations": violations}
```

本書の日本株マクロパネル（376行・145列）では、14本の `audit_*_available_at` 列に対して上記チェックを実行し、**`causality_violations = 0`** を確認したうえで以降の全検証へ進んでいます。もし1件でも違反が出れば、ICがいくら高くても即座にパイプラインを停止させるのが鉄則です。

---

## 4. 本書で構築する6段バックテスト監査アーキテクチャ

本書では、以降の章を通じて次の6段ゲートをすべて Python コードで実装していきます。

| 章 | 監査ゲート | 防げる偽陽性・実務バグ |
|---|---|---|
| **第2章（無料）** | 非複利・100株/1株単元丸め固定資本会計 | 複利水増し・売買不能な高株価銘柄の混入・休業日除外によるSharpe水増し |
| **第3章** | Uniform-Cap / Uniform-Linear Control | 稼働率低下（見送り増）を「エッジ改善」と誤認する偽陽性 |
| **第4章** | `DayBlockBaggingRegressor` ＋拡大窓WF | 多銘柄パネルの行ブートストラップOOBリーク・スケーラー未来リーク |
| **第5章** | PCA・Kaiser有効次元・上位/下位PC対決 | 多重共線性とノイズ固有ベクトル依存の過学習 |
| **第6章** | MDI vs OOS Permutation MDA 特徴量剪定 | インサンプル重要度が高いのにOOSを破壊するノイズ指標の残存 |
| **第7章** | 3段階確信度サイジング（学習窓分位点） | 正ドリフト戦略におけるバイナリCash Vetoの機会損失 |
| **第8章** | 決済時刻の小標本過学習検証＆統合ハーネス | 直近1ヶ月（27日）の幸運に騙されたテールリスク悪化 |

次章（無料公開）では、すべての損益評価の土台となる **日本株100株単元・米国株1株単元の厳密な固定資本会計モジュール（`FixedCapitalReport`）** を完全実装します。
