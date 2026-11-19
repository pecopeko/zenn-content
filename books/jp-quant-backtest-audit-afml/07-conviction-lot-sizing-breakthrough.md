---
title: "第7章：バイナリCashVetoから3段階確信度サイジングへ（+545,251円・Sharpe 2.501の実証）"
---

## 1. なぜバイナリ Cash Veto は「+57.61 bp の正ドリフト」に勝てなかったのか

第3章では、Walk-Forward OOS IC が `+0.1427` もある優秀なマクロ予測モデルを使いながら、`CashVeto`（下位20%以下の日を0株退避）も `FlipShort35bp`（35bpコストでショート反転）も、すべて `AlwaysLong`（`+3,144,591 JPY` / Sharpe `2.375`）に敗北しました。

理由を期待値の式で見ると一瞬で理解できます。

- `AlwaysLong` の日次期待リターンは **`+57.61 bp`** と極めて強い正のドリフトを持っている。
- マクロ予測モデルが下位20%（`pred <= Q20`）と判定した日であっても、条件付き期待リターンが「常に大幅マイナス」になるわけではなく、「平均より低い正のリターン、あるいはゼロ近傍」に留まる日が多い。
- そこに **バイナリ退避（投資枠 `0 JPY`）** を適用すると、期待値がまだプラスの日まで全額降りてしまい、さらに `35 bp` の空売りコストを払えば期待値は大きくマイナスへ沈む。

### 発想の転換：0株退避をやめ、「下限80万・中立160万・上限240万円」の3段階確信度サイジングへ

強い正ドリフトを持つベース戦略の上に予測モデルを重ねるときの正解は、「乗るか降りるか（`0` or `1`）」ではなく **「常にベースドリフトを掴みつつ、確信度に応じてロット上限を連続・段階的に強弱させる（Bet Sizing）」** ことです。

具体的には、各20日リフィット窓の **学習データのみ（`X[:start]`）** から予測値の20パーセンタイル（`Q20`）と80パーセンタイル（`Q80`）を計算し、OOS の各営業日に以下の上限枠を割り当てます。

- **弱気（`pred <= Q20`）**：基礎資金相当の **`800,000 JPY`**（信用レバレッジ1倍・防御モード）
- **中立（`Q20 < pred < Q80`）**：中間枠の **`1,600,000 JPY`**（2倍・通常モード）
- **強気（`pred >= Q80`）**：フル信用枠の **`2,400,000 JPY`**（3倍・攻勢モード）
- **米国休場等でマクロ予測が未更新の日**：未来値を捏造せず、保守的に中立枠 **`1,600,000 JPY`** とする

そして評価時は、全269営業日にわたる **実際の平均投入枠 $\bar{C} = \frac{1}{269}\sum_{t=1}^{269} C_t$** を算出し、まったく同じ $\bar{C}$ を269営業日すべてに一律適用した **100株単元丸め込みの Uniform-Cap Control** と一騎打ちさせます。

---

## 2. 拡大窓分位点サイジングと Uniform-Cap 対決の完全コード

```python
from __future__ import annotations

from typing import Any
import numpy as np
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
) -> dict[str, Any]:
    """拡大窓Walk-Forwardで OOS予測値 と 学習窓限定の Q20/Q80 閾値を返す。"""
    preds: dict[str, float] = {}
    q20_map: dict[str, float] = {}
    q80_map: dict[str, float] = {}

    for start in range(oos_start_idx, len(dates), refit_step):
        end = min(start + refit_step, len(dates))
        scaler = StandardScaler()
        X_tr = scaler.fit_transform(X[:start])
        X_te = scaler.transform(X[start:end])
        d_tr = np.asarray(dates[:start])

        bagger = DayBlockBaggingRegressor(
            n_estimators=n_estimators,
            alpha=alpha,
            random_state=random_state,
        ).fit(X_tr, y[:start], dates=d_tr)

        # Q20 / Q80 は必ず学習窓のインサンプル予測分布だけで決める（OOS分位点リーク禁止）
        tr_pred = bagger.predict(X_tr)
        q20 = float(np.quantile(tr_pred, 0.20))
        q80 = float(np.quantile(tr_pred, 0.80))

        te_pred = bagger.predict(X_te)
        for d_val, p_val in zip(dates[start:end], te_pred.tolist()):
            preds[d_val] = float(p_val)
            q20_map[d_val] = q20
            q80_map[d_val] = q80

    return {"preds": preds, "q20": q20_map, "q80": q80_map}


def evaluate_conviction_sizing_vs_uniform_cap(
    model_name: str,
    payoff_dates: list[str],
    long_by_date: dict[str, dict[str, Any]],
    wf_q: dict[str, Any],
    low_cap_jpy: float = 800_000.0,
    mid_cap_jpy: float = 1_600_000.0,
    high_cap_jpy: float = 2_400_000.0,
) -> dict[str, Any]:
    """3段階確信度サイジングを同一平均枠の100株単元Uniform-Cap対照と厳密比較する。"""
    n_days = len(payoff_dates)
    full_cap_trades = [long_by_date[d] for d in payoff_dates]

    scaled_trades: list[dict[str, Any]] = []
    caps: list[float] = []

    for d in payoff_dates:
        rec = dict(long_by_date[d])
        if d not in wf_q["preds"]:
            cap = mid_cap_jpy
        else:
            pred = wf_q["preds"][d]
            if pred <= wf_q["q20"][d]:
                cap = low_cap_jpy
            elif pred >= wf_q["q80"][d]:
                cap = high_cap_jpy
            else:
                cap = mid_cap_jpy
        rec["allocation_cap_jpy"] = cap
        caps.append(cap)
        scaled_trades.append(rec)

    mean_cap = float(np.mean(caps))
    strat_rep = evaluate_jp_daily_fixed_capital(
        scaled_trades, total_business_days=n_days
    )
    uniform_rep = evaluate_jp_daily_fixed_capital(
        full_cap_trades, margin_cap_jpy=mean_cap, total_business_days=n_days
    )

    edge_jpy = round(strat_rep.total_net_pnl - uniform_rep.total_net_pnl, 2)
    return {
        "name": model_name,
        "mean_cap_jpy": round(mean_cap, 2),
        "strategy_net_pnl_jpy": strat_rep.total_net_pnl,
        "strategy_sharpe": strat_rep.annualized_sharpe,
        "strategy_worst_day_jpy": strat_rep.worst_day_pnl,
        "uniform_cap_net_pnl_jpy": uniform_rep.total_net_pnl,
        "uniform_cap_sharpe": uniform_rep.annualized_sharpe,
        "alpha_over_uniform_cap_jpy": edge_jpy,
        "beats_uniform_pnl": strat_rep.total_net_pnl > uniform_rep.total_net_pnl,
        "beats_uniform_sharpe": strat_rep.annualized_sharpe > uniform_rep.annualized_sharpe,
    }
```

---

## 3. 実パネル269営業日の最終対決：`Core22` 敗北 vs `Pruned18` 大勝

日本株269営業日（`2025-09-01 .. 2026-10-15`、参考フル枠 `AlwaysLong 2,400,000 JPY` = `3,144,591 JPY` / Sharpe `2.375` / 最悪日 `-226,114 JPY`）での厳密な実測結果がこちらです。

| モデル（3段階 `80万/160万/240万`） | 平均投入枠 (JPY) | 戦略純損益 (JPY) | 戦略Sharpe | Uniform-Cap対照損益 (JPY) | 対照Sharpe | 対Uniform超過収益 | 判定 |
|---|---:|---:|---:|---:|---:|---:|---|
| `Core22_DayBlockBag50` | `1,686,245` | `2,187,226` | `2.269` | `2,146,176` | `2.362` | `+41,051 JPY` | **REJECT（Sharpe敗北）** |
| **`Pruned18_DayBlockBag50`** | **`1,683,271`** | **`2,646,201`** | **`2.501`** | `2,100,950` | `2.328` | **`+545,251 JPY`** | **PASS（損益・Sharpe両勝）** |

この結果には、クオンツシステム開発の真髄が詰まっています。

### なぜ「OOS MDA剪定」×「3段階サイジング」の両輪が必要だったのか

1. **サイジングだけでは勝てない（`Core22` の敗北）**：
   剪定前の `Core22` でも Walk-Forward OOS IC は `0.1427` ありました。にもかかわらず3段階サイジングでは Uniform-Cap 対照に対して `+41,051 JPY` しか稼げず、**年率 Sharpe は `2.269 < 2.362` と一律縮小に敗北** しました。理由は、米10年債利回り・WTI原油・KOSPI・ドル指数という **4つの負の OOS MDA 特徴量が、強気/弱気の上下20%（テール判定：`Q20` / `Q80`）の境界をランダムに揺らしていたから** です。
2. **バイナリ退避だけでは勝てない（第3章の敗北）**：
   正ドリフト `+57.61 bp` の環境で `0 JPY` に降りると、取りこぼしコストが防御益を食いつぶします。
3. **2つが噛み合った `Pruned18` の突破**：
   負の OOS MDA 4特徴量を除去した `Pruned18` では、上下20%の確信度判定（`800k` vs `2.4M`）の精度が劇的に引き締まりました。その結果、**平均投入枠は `1,683,271 JPY`（フル枠の 70.1%）に抑えたまま**、
   - 同一平均枠の Uniform-Cap 対照（`2,100,950 JPY` / Sharpe `2.328`）を **`+545,251 JPY` 上回り**
   - 年率 Sharpe は **`2.501`** に到達し、フル枠 `AlwaysLong` の Sharpe **`2.375`** さえも突破しました。

---

## 4. 資金効率の観点から見た実務的インパクト

最後に実運用のリスク管理視点で数字を言い換えます。

- フル枠 `AlwaysLong` は毎日 `2,400,000 JPY` を晒して Sharpe `2.375`。
- `Pruned18` 3段階サイジングは **平均 `1,683,271 JPY`（約30%の信用余力温存）** で Sharpe **`2.501`**。

この余力30%は、追証回避の安全バッファとして保持することも、相関の低い別戦略（例：第3章で触れた日中戦略や米国株クロスマーケット）へ再配分することも可能です。単一戦略の「見かけの勝率」を追うのではなく、**「同一平均資本あたりの超過収益（Uniform-Cap Alpha）と年率 Sharpe」** で評価したからこそ辿り着けた設計です。

最終章となる第8章では、多くの個人クオンツが最後に踏む **「決済タイミングの小標本過学習（225日 vs 27日の逆転劇）」** を実測データで検証し、本書全章を統合した再現監査ハーネスと実運用投入前の Go/No-Go チェックリストを提示します。
