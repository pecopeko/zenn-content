---
title: "第4章：多銘柄日次パネルとDayBlockBaggingRegressor"
---

## 1. 金融パネルデータで「行ブートストラップ」が崩壊する数学的理由

AFML（第4章・第6章）が強調する最も重要な論点の1つが、**「金融の観測行は i.i.d.（独立同分布）ではない」** という事実です。

たとえば毎日20銘柄をスクリーニングして1年間（250営業日）の学習パネルを作ると、データフレームの行数は $N = 5,000$ 行になります。ここに標準の `sklearn.ensemble.BaggingRegressor` や `RandomForestRegressor` をそのまま適用すると、内部では **5,000行から独立に復元抽出（Row Bootstrap）** が行われます。

ここで何が起きるかを具体的に見ましょう。

- ある営業日 $t$ の20銘柄は、すべて同じ日の日経先物・米金利・ドル円・VIXショックを同時に浴びている（高い日次クロスセクション相関 $\rho > 0$）。
- 行ブートストラップでサイズ $N$ の復元抽出を行うと、ある1行が選ばれる確率は $1 - (1 - 1/N)^N \approx 63.2\%$ である。
- したがって、同一営業日 $t$ の20銘柄のうち **平均12.6銘柄が学習袋（In-Bag）に入り、残り7.4銘柄が OOB（Out-of-Bag）に残る**。
- 結果として、弱学習器は「営業日 $t$ の12.6銘柄」からその日の市場全体の方向性を覚えてしまい、OOB の7.4銘柄を実質的にカンニングして当ててしまう。

さらに深刻なのは、**各弱学習器（決定木や線形モデル）の予測誤差間の相関が下がらない** ことです。バギングの分散低減効果は、各推定器の平均相関係数 $\bar{\rho}$ に強く支配されます。

$$
\text{Var}\left(\frac{1}{B}\sum_{b=1}^{B} \hat{f}_b(x)\right) = \bar{\rho}\,\sigma^2 + \frac{1 - \bar{\rho}}{B}\,\sigma^2
$$

行ブートストラップでは、どのバッグにも全250営業日のデータがほぼ万遍なく含まれてしまうため、バッグ間の多様性が死に、$\bar{\rho} \to 1$ に近づいてバギングの分散低減効果が失われます。

### 解決策：日付ブロック単位の復元抽出（Day-Block Bootstrap）

行ではなく **一意な日付（`signal_date`）の集合 $\{d_1, \dots, d_T\}$ から $T$ 個を復元抽出** し、選ばれた日付に属する全銘柄行をまとめて1つのバッグへ投入します。

- 各バッグには約 $63.2\%$ の営業日だけが含まれ、残り $36.8\%$ の営業日は **1銘柄も漏らさず完全に OOB 側へ隔離** される。
- 特定の暴落日や異常レジーム日がバッグから完全に抜けるケースが生まれるため、各弱学習器間の多様性（$1 - \bar{\rho}$）が回復し、外れ値日に引きずられない頑健な重みが得られる。

---

## 2. `DayBlockBaggingRegressor` の完全実装（scikit-learn 互換）

線形モデル（`Ridge`）だけでなく、任意の scikit-learn 回帰器（`HuberRegressor`, `LGBMRegressor` など）を `base_estimator` として差し替えられる設計にします。さらに第6章の OOS MDA 監査でそのまま参照できるよう、全バッグの係数平均 `coef_` と正規化インサンプル重要度 `feature_importances_` を自動的に集計します。

```python
from __future__ import annotations

from typing import Any, Sequence
import numpy as np
from sklearn.base import BaseEstimator, RegressorMixin, clone
from sklearn.linear_model import Ridge


class DayBlockBaggingRegressor(BaseEstimator, RegressorMixin):
    """AFML Ch.4/6 準拠の日付ブロック・ブートストラップ・バギング回帰器。

    行単位ではなく一意な日付ブロック単位で復元抽出を行うことで、
    同一営業日のクロスセクション相関によるバッグ間リークを遮断する。
    """

    def __init__(
        self,
        base_estimator: Any | None = None,
        n_estimators: int = 50,
        alpha: float = 50.0,
        non_negative: bool = False,
        random_state: int = 42,
    ) -> None:
        self.base_estimator = base_estimator
        self.n_estimators = n_estimators
        self.alpha = alpha
        self.non_negative = non_negative
        self.random_state = random_state

    def _make_estimator(self) -> Any:
        if self.base_estimator is not None:
            return clone(self.base_estimator)
        return Ridge(alpha=self.alpha, positive=self.non_negative)

    def fit(
        self,
        X: np.ndarray,
        y: np.ndarray,
        dates: Sequence[Any] | np.ndarray | None = None,
    ) -> DayBlockBaggingRegressor:
        X_arr = np.asarray(X, dtype=float)
        y_arr = np.asarray(y, dtype=float)
        n_rows, n_features = X_arr.shape

        if dates is None:
            date_arr = np.arange(n_rows)
        else:
            date_arr = np.asarray(dates)
            if len(date_arr) != n_rows:
                raise ValueError("dates の長さは X の行数と一致する必要があります。")

        unique_dates = np.unique(date_arr)
        date_to_indices = {d: np.flatnonzero(date_arr == d) for d in unique_dates}

        rng = np.random.default_rng(self.random_state)
        self.estimators_: list[Any] = []
        importances_acc = np.zeros(n_features, dtype=float)
        coef_acc = np.zeros(n_features, dtype=float)
        coef_count = 0

        for _ in range(self.n_estimators):
            sampled_dates = rng.choice(
                unique_dates, size=len(unique_dates), replace=True
            )
            row_idx = np.concatenate([date_to_indices[d] for d in sampled_dates])
            est = self._make_estimator()
            est.fit(X_arr[row_idx], y_arr[row_idx])
            self.estimators_.append(est)

            if hasattr(est, "coef_"):
                c = np.ravel(np.asarray(est.coef_, dtype=float))
                if len(c) == n_features:
                    coef_acc += c
                    importances_acc += np.abs(c)
                    coef_count += 1
            elif hasattr(est, "feature_importances_"):
                fi = np.ravel(np.asarray(est.feature_importances_, dtype=float))
                if len(fi) == n_features:
                    importances_acc += np.abs(fi)
                    coef_count += 1

        if coef_count > 0:
            self.coef_ = coef_acc / coef_count
            imp_sum = float(np.sum(importances_acc))
            self.feature_importances_ = (
                importances_acc / imp_sum
                if imp_sum > 0
                else np.zeros(n_features, dtype=float)
            )
        else:
            self.feature_importances_ = np.full(
                n_features, 1.0 / max(n_features, 1), dtype=float
            )
        return self

    def predict(self, X: np.ndarray) -> np.ndarray:
        if not hasattr(self, "estimators_") or not self.estimators_:
            raise RuntimeError("predict() の前に fit() を実行してください。")
        X_arr = np.asarray(X, dtype=float)
        preds = [np.asarray(est.predict(X_arr), dtype=float) for est in self.estimators_]
        return np.mean(preds, axis=0)
```

---

## 3. リークゼロの拡大窓 Walk-Forward 比較ハーネス

バギングの効果を単体 `Ridge` と厳密に比べるには、すべての前処理（特に `StandardScaler`）を各フォールドの学習窓 `X[:start]` の内部だけに閉じ込める必要があります。データ全体に `StandardScaler().fit_transform(X)` を掛けてから時系列分割するのは、未来の平均・分散を学習窓へ漏らす典型的なリークです。

```python
import math
from scipy.stats import spearmanr
from sklearn.preprocessing import StandardScaler


def safe_pearson(y_true: np.ndarray, y_pred: np.ndarray) -> float:
    if len(y_true) < 2 or float(np.std(y_true)) < 1e-12 or float(np.std(y_pred)) < 1e-12:
        return 0.0
    val = float(np.corrcoef(y_true, y_pred)[0, 1])
    return 0.0 if math.isnan(val) else round(val, 4)


def safe_spearman(y_true: np.ndarray, y_pred: np.ndarray) -> float:
    if len(y_true) < 2 or float(np.std(y_true)) < 1e-12 or float(np.std(y_pred)) < 1e-12:
        return 0.0
    val = float(spearmanr(y_true, y_pred).statistic)
    return 0.0 if math.isnan(val) else round(val, 4)


def run_walk_forward_ridge_vs_bag(
    X: np.ndarray,
    y: np.ndarray,
    dates: list[str],
    oos_start_idx: int,
    alphas: tuple[float, ...] = (20.0, 50.0, 100.0),
    refit_step: int = 20,
    n_estimators: int = 50,
    random_state: int = 42,
) -> dict[str, dict[str, Any]]:
    """20営業日ごとの拡大窓リフィットで単体RidgeとDayBlockBaggingを比較する。"""
    results: dict[str, dict[str, Any]] = {}
    y_oos = y[oos_start_idx:]

    for alpha in alphas:
        ridge_preds: list[float] = []
        bag_preds: list[float] = []
        folds = 0

        for start in range(oos_start_idx, len(dates), refit_step):
            end = min(start + refit_step, len(dates))
            folds += 1

            # StandardScalerは必ず学習窓 [:start] だけでfitする
            scaler = StandardScaler()
            X_tr = scaler.fit_transform(X[:start])
            X_te = scaler.transform(X[start:end])
            y_tr = y[:start]
            d_tr = np.asarray(dates[:start])

            ridge = Ridge(alpha=alpha).fit(X_tr, y_tr)
            bagger = DayBlockBaggingRegressor(
                n_estimators=n_estimators,
                alpha=alpha,
                random_state=random_state,
            ).fit(X_tr, y_tr, dates=d_tr)

            ridge_preds.extend(ridge.predict(X_te).tolist())
            bag_preds.extend(bagger.predict(X_te).tolist())

        r_arr = np.asarray(ridge_preds, dtype=float)
        b_arr = np.asarray(bag_preds, dtype=float)
        results[str(int(alpha))] = {
            "alpha": alpha,
            "folds": folds,
            "oos_n": len(y_oos),
            "ridge_pearson_ic": safe_pearson(y_oos, r_arr),
            "ridge_spearman_ic": safe_spearman(y_oos, r_arr),
            "bag_pearson_ic": safe_pearson(y_oos, b_arr),
            "bag_spearman_ic": safe_spearman(y_oos, b_arr),
        }
    return results
```

---

## 4. 実パネル4系統での実測比較：単体 `Ridge` vs `DayBlockBaggingRegressor`

実際に日本株マクロパネルと米国株クロスマーケットパネルの双方で、同一正則化強度（`alpha = 50`、`n_estimators = 50`、20日拡大窓リフィット）での比較を行いました。

| パネル・ターゲット | 特徴量数 | OOS日数 | WF `Ridge(50)` Pearson IC | WF `DayBlockBag(50)` Pearson IC | 改善幅 |
|---|---:|---:|---:|---:|---:|
| 日本株 `T2_us_macro`（`Full92`） | 92 | 262 | `0.0551` | **`0.0791`** | `+0.0240` |
| 日本株 `T2_us_macro`（`Core22`） | 22 | 262 | `0.1330` | **`0.1427`** | `+0.0097` |
| 日本株 `T2_us_macro`（`Pruned18`） | 18 | 262 | `0.1380` | **`0.1436`** | `+0.0056` |
| 米国株 `Long-Only`（39特徴） | 39 | 245 | `0.0259` | **`0.0272`** | `+0.0013` |
| 米国株 `Long+Short`（40特徴） | 40 | 245 | `0.0374` | **`0.0703`** | **`+0.0329`** |

さらに実損益（米国株 OOS 245日、Webull `$221.55` 固定資本、方向一致ゲート `DirectionalAgree`）へ落としたときの差も顕著です。

- **米国 Long-Only OOS**：`Ridge` = `$85.97`（Sharpe `0.868`）→ `DayBlockBag` = **`$118.26`**（Sharpe **`1.180`**）
- **米国 Long+Short OOS**：`Ridge` = `$75.55`（Sharpe `0.676`）→ `DayBlockBag` = **`$146.60`**（Sharpe **`1.215`**）

（なお両者とも `OOS_AlwaysTrade_5bp` の `$167.67 / $181.74` には届かず、バイナリ見送りゲート自体は第3章と同じく REJECT 判定となっています。）

特に注目したいのは、`Full92` や米国株 `Long+Short` のように **特徴量が多く外れ値日の影響を受けやすいパネルほど、`DayBlockBaggingRegressor` の優位幅（`+0.024〜+0.033`）が大きい** という点です。さらに `T2_us_macro` では正則化係数 `alpha = 20, 50, 100` のすべての設定で一貫して単体 `Ridge` を上回りました。

次章では、なぜ `Full92`（92特徴）より `Core22`（22特徴）のほうが圧倒的に強いのかを、**PCA（主成分分析）と Kaiser 有効次元数** によって数学的に解剖します。
