---
title: "第2章：100株単元丸め・米国1株上限・非複利固定資本会計の厳密実装"
free: true
---

## 1. なぜ「パーセント複利リターン」は個人クオンツを裏切るのか

研究用ノートブックでよく書かれる `equity = (1 + ret_bp / 10000).cumprod()` には、日本株や小口米国株の実運用とかけ離れる **4つの歪み** があります。

1. **100株単元の整数切り捨て（Lot Rounding）の無視**：日本株は原則100株単位です。最大投資枠が `2,400,000 JPY` でも、株価 `18,000円` の銘柄なら `floor(2,400,000 / 1,800,000) * 100 = 100株（180万円）` しか買えず、残り60万円は遊休資金になります。株価 `25,000円` の銘柄は **0株（売買不能）** です。
2. **複利効果による後半期間の支配**：期間前半にたまたま大勝ちして資金が2倍になると、後半の数日の勝敗がトータル損益の大部分を決めてしまい、戦略本来の期待値（エッジ）が見えなくなります。戦略評価はまず **毎日同じ固定資本でリセットする非複利（Non-Compounding）** で行うべきです。
3. **ゼロトレード営業日の分母欠落**：269営業日のうち150日だけトレードしたとき、トレードした150日だけの標準偏差で Sharpe を計算すると、資金を寝かせていた119日の存在が消えて年率 Sharpe が実態より大きく見えてしまいます。**全営業日数 `total_business_days` に対して未トレード日を `0.0 JPY` で埋めた日次系列** で年率 Sharpe を計算しなければなりません。
4. **買い（Long）と売り（Short）の非対称コスト**：日本株の信用取引では、買いのスリッページ＋手数料は `5 bp` 程度で収まっても、空売りには逆日歩・貸株料・一般信用売建プレミアム・寄付きギャップ時の踏み上げコストが乗るため、最低でも **`35 bp`** 程度の厳格なコストを置かないとショート戦略が過剰に良く見えます。

米国株の小口口座（例：現金 `$228.20 USD`、成行注文時の証券会社バッファ `103%` により実質買付上限 `$228.20 / 1.03 = $221.55 USD`、整数1株単位）でもまったく同じ構造です。1位候補の株価が `$350` なら1株も買えないため、**上位10候補（Top-10）から買付可能な最上位銘柄へフォールバックするルール** まで評価コードに忠実に組み込む必要があります。

---

## 2. `FixedCapitalReport` と日次系列集計の完全実装

ここからは、本書全体で使用する Pydantic ベースの固定資本評価モジュールを完全コードで示します。そのまま `fixed_capital.py` として保存し、手元のトレード履歴へ適用できます。

```python
from __future__ import annotations

import math
from typing import Any, Iterable, Sequence

import numpy as np
from pydantic import BaseModel, Field


class FixedCapitalReport(BaseModel):
    """非複利・日次固定資本バックテストの厳密サマリー。"""

    market: str
    currency: str
    base_capital: float
    effective_cap: float
    business_days: int
    trading_days: int
    zero_trade_days: int
    trade_count: int
    win_days: int
    loss_days: int
    flat_days: int
    total_net_pnl: float
    daily_mean_pnl: float
    return_on_base_pct: float
    avg_net_bp: float
    best_day_pnl: float
    worst_day_pnl: float
    annualized_sharpe: float
    total_net_pnl_jpy: float | None = None
    daily_pnl_series: dict[str, float] = Field(default_factory=dict)


def _to_records(trades_df_or_records: Any) -> list[dict[str, Any]]:
    if hasattr(trades_df_or_records, "to_dict"):
        return list(trades_df_or_records.to_dict(orient="records"))
    return [dict(r) for r in trades_df_or_records]


def _summarize_daily_series(
    market: str,
    currency: str,
    base_capital: float,
    effective_cap: float,
    daily_pnl: dict[str, float],
    trade_bps: list[float],
    total_business_days: int | None,
    usd_jpy_rate: float | None = None,
) -> FixedCapitalReport:
    """全営業日ベース（未稼働日は 0.0 補完）で年率Sharpeとリスク指標を集計する。"""
    traded_values = list(daily_pnl.values())
    trading_days = len(daily_pnl)
    b_days = max(trading_days, int(total_business_days or 0))
    zero_days = b_days - trading_days

    # 未稼働日を 0.0 として全営業日系列を構成する（Sharpeの水増しを防ぐ）
    all_day_pnls = traded_values + [0.0] * zero_days
    total_pnl = float(sum(all_day_pnls))
    daily_mean = total_pnl / b_days if b_days > 0 else 0.0

    win_days = sum(1 for v in traded_values if v > 0)
    loss_days = sum(1 for v in traded_values if v < 0)
    flat_days = sum(1 for v in traded_values if v == 0) + zero_days

    best_day = max(all_day_pnls) if all_day_pnls else 0.0
    worst_day = min(all_day_pnls) if all_day_pnls else 0.0

    if b_days >= 2:
        std_daily = float(np.std(np.asarray(all_day_pnls, dtype=float), ddof=1))
        sharpe = (
            (daily_mean / std_daily) * math.sqrt(252.0) if std_daily > 1e-12 else 0.0
        )
    else:
        sharpe = 0.0

    avg_bp = float(np.mean(trade_bps)) if trade_bps else 0.0
    ret_pct = (total_pnl / base_capital) * 100.0 if base_capital > 0 else 0.0
    total_jpy = (
        round(total_pnl * usd_jpy_rate, 2)
        if usd_jpy_rate is not None and currency == "USD"
        else None
    )

    return FixedCapitalReport(
        market=market,
        currency=currency,
        base_capital=round(base_capital, 2),
        effective_cap=round(effective_cap, 2),
        business_days=b_days,
        trading_days=trading_days,
        zero_trade_days=zero_days,
        trade_count=len(trade_bps),
        win_days=win_days,
        loss_days=loss_days,
        flat_days=flat_days,
        total_net_pnl=round(total_pnl, 2),
        daily_mean_pnl=round(daily_mean, 2),
        return_on_base_pct=round(ret_pct, 2),
        avg_net_bp=round(avg_bp, 2),
        best_day_pnl=round(best_day, 2),
        worst_day_pnl=round(worst_day, 2),
        annualized_sharpe=round(sharpe, 3),
        total_net_pnl_jpy=total_jpy,
        daily_pnl_series={k: round(v, 2) for k, v in sorted(daily_pnl.items())},
    )
```

---

## 3. 日本株 100株単元評価と米国株 `$221.55` 1株単元評価

次に、日本株と米国株それぞれの単元丸め・コスト控除・可変ロット上限（第7章の確信度サイジングでも再利用する `allocation_cap_jpy`）に対応した評価関数を実装します。

```python
def evaluate_jp_daily_fixed_capital(
    trades_df_or_records: Any,
    base_capital_jpy: float = 800_000.0,
    margin_cap_jpy: float = 2_400_000.0,
    lot_size: int = 100,
    long_cost_bp: float = 5.0,
    short_cost_bp: float = 35.0,
    total_business_days: int | None = None,
) -> FixedCapitalReport:
    """日本株：基礎資金80万円 / 最大信用枠240万円 / 100株単元切り捨ての固定資本評価。"""
    records = _to_records(trades_df_or_records)
    daily_pnl: dict[str, float] = {}
    trade_bps: list[float] = []

    for rec in records:
        trade_date = str(rec.get("date") or rec.get("signal_date") or "").strip()
        if not trade_date:
            raise ValueError("各レコードに 'date' または 'signal_date' が必要です。")
        price = float(rec.get("entry_price") or rec.get("price") or 0.0)
        if price <= 0.0:
            continue

        # レコード個別の allocation_cap_jpy があれば優先（確信度サイジング用）
        cap = float(rec.get("allocation_cap_jpy") or margin_cap_jpy)
        shares = math.floor(cap / (price * lot_size)) * lot_size
        if shares < lot_size:
            continue  # 1単元すら買えない高株価銘柄はエントリー不可

        side = str(rec.get("side") or "LONG").upper()
        if "gross_return_bp" in rec and rec["gross_return_bp"] is not None:
            gross_ret = float(rec["gross_return_bp"]) / 10_000.0
        elif "exit_price" in rec and rec["exit_price"] is not None:
            exit_px = float(rec["exit_price"])
            gross_ret = (
                (price - exit_px) / price
                if side == "SHORT"
                else (exit_px - price) / price
            )
        else:
            raise ValueError("'exit_price' または 'gross_return_bp' が必要です。")

        cost_bp = short_cost_bp if side == "SHORT" else long_cost_bp
        net_ret = gross_ret - (cost_bp / 10_000.0)
        notional = shares * price
        pnl = notional * net_ret

        daily_pnl[trade_date] = daily_pnl.get(trade_date, 0.0) + pnl
        trade_bps.append(net_ret * 10_000.0)

    return _summarize_daily_series(
        market="JP",
        currency="JPY",
        base_capital=base_capital_jpy,
        effective_cap=margin_cap_jpy,
        daily_pnl=daily_pnl,
        trade_bps=trade_bps,
        total_business_days=total_business_days,
    )


def evaluate_us_daily_fixed_capital(
    candidates_by_day: dict[str, list[dict[str, Any]]] | Sequence[tuple[str, list[dict[str, Any]]]],
    webull_cash_usd: float = 228.20,
    market_order_buffer: float = 1.03,
    lot_size: int = 1,
    roundtrip_cost_bp: float = 5.0,
    usd_jpy_rate: float = 156.0,
    total_business_days: int | None = None,
) -> FixedCapitalReport:
    """米国株：現金$228.20 / 成行103%バッファ上限$221.55 / Top-10買付可能選択の固定資本評価。"""
    buying_power = webull_cash_usd / market_order_buffer
    items: Iterable[tuple[str, list[dict[str, Any]]]] = (
        candidates_by_day.items() if isinstance(candidates_by_day, dict) else candidates_by_day
    )

    daily_pnl: dict[str, float] = {}
    trade_bps: list[float] = []
    observed_days = 0

    for trade_date, ranked_candidates in items:
        observed_days += 1
        chosen: dict[str, Any] | None = None
        chosen_shares = 0
        chosen_price = 0.0

        # 予測スコア上位10銘柄から、1株以上買える最上位銘柄を採用
        for cand in ranked_candidates[:10]:
            px = float(cand.get("entry_price") or cand.get("price") or 0.0)
            if px <= 0.0:
                continue
            shares = math.floor(buying_power / (px * lot_size)) * lot_size
            if shares >= lot_size:
                chosen = cand
                chosen_shares = shares
                chosen_price = px
                break

        if chosen is None:
            continue

        side = str(chosen.get("side") or "LONG").upper()
        if "gross_return_bp" in chosen and chosen["gross_return_bp"] is not None:
            gross_ret = float(chosen["gross_return_bp"]) / 10_000.0
        elif "exit_price" in chosen and chosen["exit_price"] is not None:
            exit_px = float(chosen["exit_price"])
            gross_ret = (
                (chosen_price - exit_px) / chosen_price
                if side == "SHORT"
                else (exit_px - chosen_price) / chosen_price
            )
        else:
            raise ValueError("'exit_price' または 'gross_return_bp' が必要です。")

        net_ret = gross_ret - (roundtrip_cost_bp / 10_000.0)
        pnl = (chosen_shares * chosen_price) * net_ret
        daily_pnl[str(trade_date)] = daily_pnl.get(str(trade_date), 0.0) + pnl
        trade_bps.append(net_ret * 10_000.0)

    effective_total_days = max(observed_days, int(total_business_days or 0))
    return _summarize_daily_series(
        market="US",
        currency="USD",
        base_capital=webull_cash_usd,
        effective_cap=buying_power,
        daily_pnl=daily_pnl,
        trade_bps=trade_bps,
        total_business_days=effective_total_days,
        usd_jpy_rate=usd_jpy_rate,
    )
```

---

## 4. 実パネルでのベースライン実測と「買えない日」の可視化

上記モジュールを実データに適用するだけで、パーセント計算では見えない重要な事実が即座に浮かび上がります。

| 市場・ベースライン | 買付可能 / 全営業日 | 純損益 | 平均純利 (bp) | 最悪日損益 | 年率 Sharpe |
|---|---:|---:|---:|---:|---:|
| 日本株 `AlwaysLong_Open_5bp` (`2.4M JPY` 枠) | `269 / 269` | **`+3,144,591 JPY`** | `+57.61` | `-226,114 JPY` | **`2.375`** |
| 日本株 `AlwaysShort_Open_35bp` (`2.4M JPY` 枠) | `269 / 269` | `-5,434,431 JPY` | `-97.29` | `-457,979 JPY` | `-4.108` |
| 米国株 `Long-Only AlwaysTrade_5bp` (`$221.55` 枠) | **`341 / 371`** | `+$134.21` (`+20,937 JPY`) | `+20.85` | `$-46.30` | `0.764` |
| 米国株 `Long+Short AlwaysTrade_5bp` (`$221.55` 枠) | **`340 / 370`** | **`+$314.38`** (`+49,043 JPY`) | `+44.26` | `$-46.30` | **`1.502`** |
| 米国株 `Long+Short Short35bp感度` (`$221.55` 枠) | `340 / 370` | `+$277.12` (`+43,230 JPY`) | `+38.97` | `$-46.30` | `1.325` |

米国株では、`370〜371営業日` のうち **約30営業日（約8%）は1位シグナルの株価が `$221.55` を超えており1株も買えない** ことが数字として露出しています。また、日本株では `AlwaysLong` が日次 `+57.61 bp` の強い正ドリフトを持つ一方、`AlwaysShort`（35bpコスト）は `-5,434,431 JPY`（Sharpe `-4.108`）という強烈な逆風下にあります。

さらに為替換算にも落とし穴があります。データファイルによっては米国株価格が「当時の固定レート（例：`150.0 JPY/USD`）で円建て保存」されていることがあります。そのまま `156.0` を掛けて評価すると二重換算になるため、**必ず元のネイティブ通貨（`/ 150.0` で USD 復元）へ戻してから `$221.55` の1株単元丸めを適用する** のが厳格な会計手順です。

---

## 次章から有料パート：偽エッジを暴き、本物の `+545,251円` エッジを掘り出す

ここまで無料公開した第1〜2章で、「リークゼロ・欠損捏造ゼロ・単元丸め厳密会計」という土台が完成しました。

第3章以降の有料パートでは、この土台の上で **一見すると成功したように見えるフィルタリング戦略がなぜ Uniform-Cap Control で次々と棄却されるのか**、そして **Day-Block Bagging × PCA直交性 × OOS MDA特徴量剪定 × 3段階確信度サイジング** によっていかにして Uniform-Cap 対照を `+545,251 JPY`（Sharpe `2.501` vs `2.328`）上回る本物のエッジへ到達したのかを、実測ログと完全コードで余すところなく解説します。
