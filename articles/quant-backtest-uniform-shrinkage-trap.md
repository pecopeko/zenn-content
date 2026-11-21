---
title: "日本株バックテストの偽陽性を暴く：Uniform-Cap・Day-Block Bagging・OOS MDA剪定"
emoji: "📈"
type: "tech"
topics: ["python", "quant", "machinelearning", "backtest", "finance"]
published: true
---

## はじめに：なぜ「改善したはずのバックテスト」は実運用で崩れるのか

日本株の機械学習トレードモデルを検証していると、次のような「一見すると見事な改善」に何度も出会います。

- マクロ指標やテクニカル指標を90個以上投入したら、学習期間のIC（Information Coefficient）が `0.75` を超えた
- 「予測スコアが悪い日は見送る（Cash Veto）」ゲートを入れたら、取引1回あたりの平均リターン（bp）が上がり、最大ドローダウンも減った
- `sklearn.ensemble.BaggingRegressor` を使ったら OOB スコアが大幅に改善した

しかし、これらの多くは **実運用で再現しない偽陽性（False Positive）** です。原因は大きく4つに集約されます。

1. **高次元・多重共線性による静的ホールドアウト崩壊**（92特徴量で Train IC `0.7792` → OOS IC `-0.0617`）
2. **日次クロスセクション相関を無視した行ブートストラップのOOBリーク**
3. **稼働率の低下を「エッジ」と誤認する比較設計**（対一律縮小対照＝Uniform-Cap Control の欠如）
4. **インサンプル重要度（MDI）に騙されたノイズ特徴量の残存**

本記事では、日本株の引けエントリー〜翌日寄付決済（オーバーナイト）を対象とした実データパネル（2025-04〜2026-10、最大376営業日・実損益照合269営業日）に対して行った AFML（Advances in Financial Machine Learning）準拠の監査結果と、そのまま手元で動かせる Python 実装をまとめて解説します。

---

## 1. 静的ホールドアウト崩壊とデータ監査の前提

まずは「特徴量を増やすほど精度が上がる」という直感が、金融時系列ではどれほど危険かを実測値で確認します。

### 実パネル検証の前提ルール（リーク・欠損の厳格統制）

検証スクリプトでは、以下をハード制約として課しています。

- **`.fillna(0)` の全面禁止**：欠損値のあるターゲット行・特徴量列をゼロ埋めせず、明示的に除外して件数を記録する（欠損を `0.0` で埋めると、リターン `0 bp` という偽の「無風日」を捏造することになる）
- **タイムスタンプ因果監査**：引け前の発注締切である `15:25 JST`（`06:25 UTC`）以降に確定したデータが混入していないか、14本の `audit_*_available_at` 列を全行チェック（違反 `0` 件を確認）
- **非複利・固定資本会計**：日本株は基礎資金 `800,000 JPY`・最大信用枠 `2,400,000 JPY`・**100株単元切り捨て**・往復コスト（Long `5 bp` / Short `35 bp`）で損益を厳密計算

### 92特徴量（`Full92`）vs 厳選22特徴量（`Core22`）の実測比較

同じパネルに対して、ラグやボラティリティを網羅した `Full92`（92特徴量）と、主要マクロ・ギャップに絞った `Core22`（22特徴量）を比較した結果がこちらです（静的ホールドアウトは学習100〜104日 / OOS 262〜272日、Walk-Forward は20日ごとの拡大窓リフィット・`alpha=50`）。

| ターゲット | 特徴量群 | 学習 / OOS日数 | WF `Ridge(50)` IC | WF `DayBlockBag(50)` IC | 静的 Train IC | 静的 OOS IC | Kaiser有効次元 |
|---|---|---:|---:|---:|---:|---:|---:|
| `T3_jp_top20`（日本株上位20） | `Full92` | 104 / 272 | `0.0763` | `0.0730` | **`0.7792`** | **`-0.0617`** | `15 / 92` |
| `T3_jp_top20`（日本株上位20） | `Core22` | 104 / 272 | `0.0499` | `0.0503` | `0.5818` | `-0.0503` | `7 / 22` |
| `T2_us_macro`（米国マクロ連動） | `Full92` | 100 / 262 | `0.0551` | `0.0791` | `0.7549` | `0.0311` | `15 / 92` |
| `T2_us_macro`（米国マクロ連動） | `Core22` | 100 / 262 | **`0.1330`** | **`0.1427`** | `0.5524` | **`0.0886`** | `7 / 22` |

ここで注目すべきポイントは3つあります。

1. **`Full92` の静的学習ICは完全に幻影**：`T3_jp_top20` では Train IC `0.7792` という夢のような数字が出ながら、静的 OOS では `-0.0617` へ転落しています。92特徴量あっても Kaiser 基準（固有値 $\ge 1.0$）の有効次元は `15` しかなく、残りは多重共線性の塊です。
2. **特徴量削減だけで Walk-Forward OOS IC が倍増**：`T2_us_macro` では `Full92` → `Core22` に絞っただけで、Walk-Forward OOS Pearson IC が `0.0551 -> 0.1330`（Ridge）、`0.0791 -> 0.1427`（Day-Block Bagging）と **2倍以上** に跳ね上がりました。
3. **静的分割だけではレジーム変化に耐えられない**：`T3_jp_top20` は静的ホールドアウトで負のICですが、20日ごとに拡大窓でリフィットする Walk-Forward にするとプラス（`0.050〜0.076`）に回復します。

---

## 2. 行ブートストラップのOOBリークと `DayBlockBaggingRegressor`

金融の多銘柄パネルデータ（1営業日に複数銘柄の行が並ぶデータ）や、レジームが連続する日次系列に対して、通常の `BaggingRegressor`（行単位の独立サンプリング）を使うと深刻なリークが起きます。

### なぜ通常の行ブートストラップはダメなのか

ある営業日 $t$ に20銘柄の観測行があるとき、通常の行ブートストラップでは、同じ日 $t$ のうち約63.2%の銘柄が学習袋（In-Bag）に入り、残りの36.8%が OOB（Out-of-Bag）に残ります。しかし同一営業日の銘柄は、その日のマクロショック（米金利・為替・指数先物・VIX）を強く共有しています。結果として、

- 各弱学習器が「同じ日の別銘柄」を実質的にカンニングしてしまい、OOB 誤差が過小評価される
- バギング本来の目的である「日次レジーム分散の低減」が十分に働かない

### 日付ブロック単位でリサンプリングする完全実装

解決策はシンプルで、**「行（row）」ではなく「一意な日付（`signal_date`）ブロック」単位で復元抽出する** ことです。以下は scikit-learn 互換でそのまま使える `DayBlockBaggingRegressor` の完全コードです。

```python
from __future__ import annotations

from typing import Any, Sequence
import numpy as np
from sklearn.base import BaseEstimator, RegressorMixin, clone
from sklearn.linear_model import Ridge


class DayBlockBaggingRegressor(BaseEstimator, RegressorMixin):
    """AFML Ch.4/6 Day-Block Bootstrap Bagging Regressor.

    行単位ではなく一意な日付ブロック単位で復元抽出することで、
    同一営業日のクロスセクション相関がバッグ間でリークするのを防ぐ。
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
                raise ValueError("dates length must match X row count.")

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
            raise RuntimeError("fit() を先に呼び出してください。")
        X_arr = np.asarray(X, dtype=float)
        preds = [np.asarray(est.predict(X_arr), dtype=float) for est in self.estimators_]
        return np.mean(preds, axis=0)
```

さらに Walk-Forward 検証では、`StandardScaler` の `fit` や分位点閾値（`Q20` / `Q80`）の算出も **必ず学習窓 `X[:start]` の内部だけに閉じる** 必要があります。

```python
from sklearn.preprocessing import StandardScaler


def run_walk_forward_with_quantiles(
    X: np.ndarray,
    y: np.ndarray,
    dates: list[str],
    oos_start_idx: int,
    alpha: float = 50.0,
    refit_step: int = 20,
    n_estimators: int = 50,
    random_state: int = 42,
) -> dict[str, dict[str, float]]:
    """拡大窓Walk-Forwardでリークゼロの予測値と学習窓Q20/Q80閾値を返す。"""
    preds: dict[str, float] = {}
    q20_map: dict[str, float] = {}
    q80_map: dict[str, float] = {}

    for start in range(oos_start_idx, len(dates), refit_step):
        end = min(start + refit_step, len(dates))
        scaler = StandardScaler()
        X_tr = scaler.fit_transform(X[:start])
        X_te = scaler.transform(X[start:end])

        bagger = DayBlockBaggingRegressor(
            n_estimators=n_estimators,
            alpha=alpha,
            random_state=random_state,
        ).fit(X_tr, y[:start], dates=np.asarray(dates[:start]))

        tr_pred = bagger.predict(X_tr)
        q20 = float(np.quantile(tr_pred, 0.20))
        q80 = float(np.quantile(tr_pred, 0.80))
        te_pred = bagger.predict(X_te)

        for d_val, p_val in zip(dates[start:end], te_pred.tolist()):
            preds[d_val] = float(p_val)
            q20_map[d_val] = q20
            q80_map[d_val] = q80

    return {"preds": preds, "q20": q20_map, "q80": q80_map}
```

この単純な日付ブロック・バギングだけで、単体の `Ridge(alpha=50)` と比べて OOS IC は安定して向上しました。

- `T2_us_macro Full92`: `0.0551` → **`0.0791`**
- `T2_us_macro Core22`: `0.1330` → **`0.1427`**
- 米国クロスマーケット Long+Short（370日）: `0.0374` → **`0.0703`**

---

## 3. Uniform-Cap Control で「優秀に見える Cash Veto」の敗北を暴く

ここからが本記事の核心です。モデルに **本物の予測IC（`+0.1427`）があるにもかかわらず、ナイーブな売買ゲートにすると資産が増えない** という現象が実パネルで起きました。

### なぜ「対一律縮小対照（Uniform-Cap Control）」が必須なのか

たとえば「予測スコアが下位20%以下の日は現金待機（Cash Veto）」というゲートを導入したとします。269営業日のうち230日だけエントリー（稼働率 `230 / 269 = 85.5%`）した場合、単に全営業日投資のベースライン（`AlwaysLong`）と並べるだけでは不十分です。

- 稼働率を `85.5%` に落とせば、予測力ゼロのランダムな間引きでも **損益の絶対額もドローダウンも機械的に縮む**
- しかも日本株には **100株単元の整数丸め（`floor(cap / (price * 100)) * 100`）** があるため、資金上限を85.5%に一律縮小したときの損益は、単なる線形スケーリング（`AlwaysLong * 0.855`）とはズレる

したがって、フィルタリングやサイジングを評価する際は、**次の2種類の対照群** を必ず併記します。

1. **Uniform-Linear Control**：ベースライン損益 $\times$ 稼働率（理論上の線形縮小）
2. **Uniform-Cap Control**：モデルの平均配分上限（例：`2,400,000 * active_rate` やサイジング時の平均キャップ）を **全営業日に均等適用し、100株単元丸めまで再計算した実測損益**

### 日本株100株単元・固定資本評価と Uniform-Cap Control の実装

```python
import math
from typing import Any
import numpy as np


def evaluate_jp_fixed_capital(
    trades: list[dict[str, Any]],
    base_capital_jpy: float = 800_000.0,
    margin_cap_jpy: float = 2_400_000.0,
    lot_size: int = 100,
    long_cost_bp: float = 5.0,
    short_cost_bp: float = 35.0,
    total_business_days: int | None = None,
) -> dict[str, float]:
    """非複利・100株単元丸めありの日本株固定資本評価。"""
    daily_pnl: dict[str, float] = {}
    trade_bps: list[float] = []

    for rec in trades:
        price = float(rec["entry_price"])
        if price <= 0.0:
            continue
        cap = float(rec.get("allocation_cap_jpy", margin_cap_jpy))
        shares = math.floor(cap / (price * lot_size)) * lot_size
        if shares < lot_size:
            continue

        side = str(rec.get("side", "LONG")).upper()
        exit_px = float(rec["exit_price"])
        gross_ret = (
            (price - exit_px) / price if side == "SHORT" else (exit_px - price) / price
        )
        cost_bp = short_cost_bp if side == "SHORT" else long_cost_bp
        net_ret = gross_ret - (cost_bp / 10_000.0)
        pnl = (shares * price) * net_ret

        d = str(rec["date"])
        daily_pnl[d] = daily_pnl.get(d, 0.0) + pnl
        trade_bps.append(net_ret * 10_000.0)

    b_days = max(len(daily_pnl), int(total_business_days or 0))
    all_pnls = list(daily_pnl.values()) + [0.0] * (b_days - len(daily_pnl))
    total_pnl = float(sum(all_pnls))
    mean_pnl = total_pnl / b_days if b_days > 0 else 0.0
    std_pnl = float(np.std(all_pnls, ddof=1)) if b_days >= 2 else 0.0
    sharpe = (mean_pnl / std_pnl) * math.sqrt(252.0) if std_pnl > 1e-12 else 0.0

    return {
        "trades": len(trade_bps),
        "business_days": b_days,
        "total_net_pnl": round(total_pnl, 2),
        "avg_net_bp": round(float(np.mean(trade_bps)) if trade_bps else 0.0, 2),
        "worst_day_pnl": round(min(all_pnls) if all_pnls else 0.0, 2),
        "annualized_sharpe": round(sharpe, 3),
    }


def compare_gate_with_uniform_control(
    selected_trades: list[dict[str, Any]],
    all_long_trades: list[dict[str, Any]],
    total_business_days: int,
    margin_cap_jpy: float = 2_400_000.0,
) -> dict[str, Any]:
    """Cash Veto ゲートを AlwaysLong・Uniform-Linear・Uniform-Cap の3基準で監査する。"""
    base = evaluate_jp_fixed_capital(
        all_long_trades, margin_cap_jpy=margin_cap_jpy, total_business_days=total_business_days
    )
    gate = evaluate_jp_fixed_capital(
        selected_trades, margin_cap_jpy=margin_cap_jpy, total_business_days=total_business_days
    )
    active_rate = gate["trades"] / max(base["trades"], 1)
    uniform_linear_pnl = round(base["total_net_pnl"] * active_rate, 2)
    uniform_cap = evaluate_jp_fixed_capital(
        all_long_trades,
        margin_cap_jpy=margin_cap_jpy * active_rate,
        total_business_days=total_business_days,
    )
    return {
        "gate": gate,
        "active_rate": round(active_rate, 4),
        "uniform_linear_pnl": uniform_linear_pnl,
        "uniform_cap": uniform_cap,
        "beats_uniform_cap_pnl": gate["total_net_pnl"] > uniform_cap["total_net_pnl"],
        "beats_always_long_sharpe": gate["annualized_sharpe"] > base["annualized_sharpe"],
    }
```

### 実測結果（日本株269営業日）：Cash Veto も 35bp ショート反転も全敗

この監査を日本株269営業日（`2025-09-01 .. 2026-10-15`）の `Core22` Walk-Forward モデルに適用した結果がこちらです。

| 戦略バリアント | 稼働日数 | 純損益 (JPY) | Uniform-Cap対照 (JPY) | Uniform-Linear対照 (JPY) | 平均純利 (bp) | 最悪日 (JPY) | 年率Sharpe | 判定 |
|---|---:|---:|---:|---:|---:|---:|---:|---|
| `AlwaysLong_Open_5bp` | 269/269 | **3,144,591** | - | - | 57.61 | -226,114 | **2.375** | BASELINE |
| `AlwaysShort_Open_35bp` | 269/269 | -5,434,431 | - | - | -97.29 | -457,979 | -4.108 | BASELINE |
| `T3 Ridge Pred>0 CashVeto` | 169/269 | 1,821,435 | 1,488,921 | 1,975,598 | 51.33 | -226,114 | 1.610 | REJECT |
| `T3 DayBlockBag Pred>0 CashVeto` | 179/269 | 2,430,502 | 1,823,275 | 2,092,498 | 64.84 | -226,114 | 2.056 | REJECT |
| `T2 Ridge Pred>0 CashVeto` | 210/269 | 2,431,470 | 2,603,001 | 2,454,885 | 57.32 | -226,114 | 1.980 | REJECT |
| `T2 DayBlockBag Pred>TrainQ20 CashVeto` | 230/269 | 2,963,503 | 2,645,534 | 2,688,684 | 64.01 | -226,114 | 2.336 | REJECT |
| `T2 DayBlockBag Pred>TrainQ20 FlipShort35bp` | 269/269 | 2,449,521 | - | - | 46.04 | -233,242 | 1.838 | REJECT |

この表には、実務で極めて重要な2つの教訓が詰まっています。

1. **`T2 Ridge Pred>0 CashVeto` は「見かけの勝率」に騙される典型**：一見すると `+2,431,470 JPY` を稼いでいますが、同じ稼働率（`210/269`）で一律に枠を縮めただけの **Uniform-Cap 対照（`2,603,001 JPY`）にすら負けています**。ゲートを入れるくらいなら、単に全営業日サイズを縮めたほうが儲かるのです。
2. **最良の `CashVeto` でも `AlwaysLong` に勝てない理由**：`T2 DayBlockBag Pred>TrainQ20 CashVeto` は平均純利を `57.61 bp -> 64.01 bp` に引き上げ、Uniform-Cap 対照（`2,645,534 JPY`）も `31.8万円` 上回りました。それでも **総損益（`2,963,503` vs `3,144,591 JPY`）と年率 Sharpe（`2.336` vs `2.375`）の両方で `AlwaysLong` に敗北** しています。原因は明快で、ベース戦略自体に `+57.61 bp/日` の強い正ドリフトがあるため、弱気シグナルでポジションを **完全に `0` にしてしまう（バイナリ退避）と、期待値プラスの取りこぼしコストが防御益を上回る** からです。まして `35 bp` の信用売りコストを払ってショート反転（`FlipShort35bp`）すれば、`2,449,521 JPY`（Sharpe `1.838`）へ大きく沈みます。

---

## 4. OOS MDA特徴量剪定（`Pruned18`）× 3段階確信度サイジングによる突破

ここで戦略の発想を2点転換します。

1. **インサンプル重要度（MDI）を捨て、OOS MDA（Out-of-Sample Mean Decrease Accuracy）でノイズ特徴量を外科的に切除する**
2. **「0 か 100 か」のバイナリ Cash Veto をやめ、正ドリフトを保ちながら確信度に応じて `80万 / 160万 / 240万円` の3段階でロット配分する**

### PCA有効次元と OOS MDA 監査の実装

`Core22` の静的ホールドアウト（学習100日 / OOS 262日）を PCA 直交性監査と OOS 順列重要度（Permutation MDA）にかけると、次の構造が浮かび上がります。

```python
from sklearn.decomposition import PCA


def run_oos_mda_and_pca_audit(
    fitted_model: Any,
    X_tr_scaled: np.ndarray,
    y_tr: np.ndarray,
    X_te_scaled: np.ndarray,
    y_te: np.ndarray,
    feature_names: list[str],
    n_repeats: int = 20,
    random_state: int = 42,
) -> dict[str, Any]:
    """OOS Permutation MDA と PCA 上位/下位主成分のOOS IC比較を行う。"""
    def pearson(a: np.ndarray, b: np.ndarray) -> float:
        if len(a) < 2 or np.std(a) < 1e-12 or np.std(b) < 1e-12:
            return 0.0
        return float(np.corrcoef(a, b)[0, 1])

    base_oos_ic = pearson(y_te, fitted_model.predict(X_te_scaled))
    rng = np.random.default_rng(random_state)
    mda_rows = []

    for j, name in enumerate(feature_names):
        drops = []
        X_perm = X_te_scaled.copy()
        for _ in range(n_repeats):
            X_perm[:, j] = rng.permutation(X_te_scaled[:, j])
            perm_ic = pearson(y_te, fitted_model.predict(X_perm))
            drops.append(base_oos_ic - perm_ic)
        X_perm[:, j] = X_te_scaled[:, j]
        mda_rows.append((name, round(float(np.mean(drops)), 4)))

    # Kaiser基準（固有値 >= 1.0）による上位PC vs 下位PCのOOS IC対決
    pca = PCA()
    Z_tr = pca.fit_transform(X_tr_scaled)
    Z_te = pca.transform(X_te_scaled)
    kaiser_k = int(np.sum(pca.explained_variance_ >= 1.0))
    top_k = max(1, min(kaiser_k, len(feature_names) - 1))

    top_ic = pearson(y_te, Ridge(alpha=10.0).fit(Z_tr[:, :top_k], y_tr).predict(Z_te[:, :top_k]))
    bot_ic = pearson(y_te, Ridge(alpha=10.0).fit(Z_tr[:, top_k:], y_tr).predict(Z_te[:, top_k:]))

    return {
        "base_oos_ic": round(base_oos_ic, 4),
        "kaiser_dims": kaiser_k,
        "top_k_oos_ic": round(top_ic, 4),
        "bottom_k_oos_ic": round(bot_ic, 4),
        "edge_on_top_pcs": top_ic > bot_ic and top_ic > 0.0,
        "mda_sorted": sorted(mda_rows, key=lambda x: x[1], reverse=True),
    }
```

`T2_us_macro Core22` に対する実測結果は極めて鮮明でした。

- **PCA直交性**：Kaiser 有効次元 `7 / 22`、上位7主成分の OOS IC = **`0.0602`** vs 下位15主成分の OOS IC = **`0.0256`**（`edge_on_top_pcs = True`：主要な共通因子にエッジが存在）
- **正の OOS MDA 上位4特徴量（本物のシグナル）**：
  - `gap_minus_nasdaq100`（IS重要度 `0.043` / OOS MDA **`+0.0307`**）
  - `euro_stoxx50_return_1d`（IS重要度 `0.161` / OOS MDA **`+0.0225`**）
  - `vix_return_1d`（IS重要度 `0.049` / OOS MDA **`+0.0213`**）
  - `anchor_overnight_std20`（IS重要度 `0.033` / OOS MDA **`+0.0168`**）
- **負の OOS MDA 下位4特徴量（予測を破壊していたノイズ）**：
  - `us_10y_yield_return_1d`（IS重要度 `0.080` / OOS MDA **`-0.0240`**）
  - `crude_oil_wti_futures_return_1d`（IS重要度 `0.040` / OOS MDA **`-0.0179`**）
  - `kospi_return_1d`（IS重要度 `0.023` / OOS MDA **`-0.0120`**）
  - `dollar_index_futures_return_1d`（IS重要度 `0.052` / OOS MDA **`-0.0110`**）

特に米10年債利回り（`us_10y_yield_return_1d`）とドル指数（`dollar_index_futures_return_1d`）は、インサンプルでは堂々の `8.0%`・`5.2%` の重みを占めていたにもかかわらず、OOS ではシャッフルして壊したほうが精度が上がる **過学習ドラッグ** でした。

### `Pruned18` × 3段階確信度サイジング（`80万 / 160万 / 240万円`）の実測結果

上記4つの負MDA特徴量を落とした **`Pruned18`（18特徴量）** で Walk-Forward を再実行すると、OOS Pearson IC はさらに改善しました。

- `Core22` → `Pruned18`（OOS 262日、`alpha=50`）：
  - `Ridge(50)` Pearson IC: `0.1330` → **`0.1380`**
  - `DayBlockBag(50)` Pearson IC: `0.1427` → **`0.1436`**

そして各営業日のリークゼロ予測値 `pred` と、その学習窓だけで決めた `Q20`・`Q80` を使い、以下の3段階サイジングを適用して **同一平均資本の Uniform-Cap Control** と対決させます。

```python
def apply_3tier_conviction_sizing(
    payoff_dates: list[str],
    long_by_date: dict[str, dict[str, Any]],
    wf_q: dict[str, dict[str, float]],
) -> tuple[list[dict[str, Any]], float]:
    """学習窓のQ20/Q80に基づき 80万 / 160万 / 240万円 の3段階上限を割り当てる。"""
    scaled_trades: list[dict[str, Any]] = []
    caps: list[float] = []

    for d in payoff_dates:
        rec = dict(long_by_date[d])
        if d not in wf_q["preds"]:
            cap = 1_600_000.0  # 米国休場等で予測未更新の日は中立枠
        else:
            pred = wf_q["preds"][d]
            if pred <= wf_q["q20"][d]:
                cap = 800_000.0
            elif pred >= wf_q["q80"][d]:
                cap = 2_400_000.0
            else:
                cap = 1_600_000.0
        rec["allocation_cap_jpy"] = cap
        caps.append(cap)
        scaled_trades.append(rec)

    return scaled_trades, float(np.mean(caps))
```

日本株269営業日での対決結果がこちらです（参考：フル枠 `2,400,000 JPY` の `AlwaysLong` は `3,144,591 JPY` / Sharpe `2.375` / 最悪日 `-226,114 JPY`）。

| モデル（3段階サイジング） | 平均投入枠 (JPY) | 戦略純損益 (JPY) | 戦略Sharpe | Uniform-Cap対照損益 (JPY) | 対照Sharpe | 対一律縮小エッジ | 判定 |
|---|---:|---:|---:|---:|---:|---:|---|
| `Core22_DayBlockBag50` | 1,686,245 | 2,187,226 | 2.269 | 2,146,176 | 2.362 | `+41,051 JPY` | **REJECT（Sharpe敗北）** |
| **`Pruned18_DayBlockBag50`** | **1,683,271** | **`2,646,201`** | **`2.501`** | `2,100,950` | `2.328` | **`+545,251 JPY`** | **PASS（損益・Sharpe両勝）** |

このコントラストは圧巻です。

- **剪定前の `Core22`** で3段階サイジングを回しても、Uniform-Cap 対照に対する超過収益はわずか `+41,051 JPY` で、**年率 Sharpe は `2.269 < 2.362` と一律縮小に敗北** しました。
- **負の OOS MDA 4特徴量を除去した `Pruned18`** にした瞬間、ほぼ同一の平均投入枠（`1,683,271 JPY`、フル枠の約70%）でありながら、Uniform-Cap 対照（`2,100,950 JPY` / Sharpe `2.328`）を **`+545,251 JPY` 引き離し、年率 Sharpe `2.501` を達成** しました。
- さらにフル枠 `AlwaysLong`（`2,400,000 JPY`、Sharpe `2.375`）と比べても、**約70%の平均資金効率で Sharpe を `2.375 -> 2.501` へ押し上げる** ことに成功しています。

---

## まとめ：実運用前に必ず通すべき4つの関所

日本株バックテストの偽陽性を構造的に潰すには、モデルのアルゴリズムを凝る前に、次の4つの監査パイプラインを固定化するのが最も効果的です。

1. **`.fillna(0)` 禁止＋タイムスタンプ監査**でデータリークと欠損捏造をゼロにする
2. **行ブートストラップを捨て、`DayBlockBaggingRegressor`＋拡大窓 Walk-Forward** で評価する
3. **すべてのゲート・サイジングを 100株単元丸め込みの `Uniform-Cap Control` と比較** し、稼働率低下による偽エッジを棄却する
4. **インサンプル重要度ではなく OOS MDA でノイズ特徴量を剪定** し、正ドリフト戦略ではバイナリ退避ではなく連続・段階サイジングを検証する

今回解説した全監査モジュール（`FixedCapitalReport`、米国株 `$221.55` 1株単元丸め比較、PCA固有値監査、決済待機時間の小標本過学習 `225日 vs 27日` の実証を含む全8章・再現コード一式）は、以下の Zenn Book に体系的にまとめています。第1章・第2章は無料公開しているので、まずはコード品質を確認してみてください。

https://zenn.dev/pecopeko/books/jp-quant-backtest-audit-afml

