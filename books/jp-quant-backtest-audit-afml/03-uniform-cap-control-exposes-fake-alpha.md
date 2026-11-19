---
title: "第3章：Uniform-Cap Controlで偽アルファを暴く（CashVetoとメタ閾値の敗北ログ）"
---

## 1. フィルタリング戦略最大の罠：「休むだけで指標が綺麗に見える」現象

バックテスト改善の現場で最も頻発するのが、**「予測スコアが悪い日を見送るゲート（Cash Veto）を入れたら、平均リターン（bp）と最大ドローダウンが改善した！」** という錯覚です。

仮に元の戦略が269営業日すべてに投資しており、新しいゲートが210営業日だけエントリー（稼働率 `210 / 269 = 78.07%`）したとします。このとき、予測力が完全にゼロのコイン投げであっても、

- 市場に晒される日数が22%減るため、累計損益もドローダウンの絶対額も約22%縮む
- たまたま暴落日の1日をランダムに避けるだけで、最大ドローダウンが大きく改善したように見える

この錯覚を構造的に見抜くのが **対一律縮小対照（Uniform-Shrinkage Control）** です。

### 2種類の対照ベンチマーク

ゲート戦略の稼働率を $r = N_{\text{gate}} / N_{\text{base}}$（または可変サイジング時の平均投入枠を $\bar{C}$）としたとき、必ず次の2つの対照群を計算します。

1. **Uniform-Linear Control（線形縮小対照）**
   $$\text{PnL}_{\text{linear}} = \text{PnL}_{\text{base}} \times r$$
   単元丸めが存在しない理想連続空間で、単純に全営業日のポジションサイズを $r$ 倍に一律縮小したときの理論損益です。
2. **Uniform-Cap Control（実単元丸め一律縮小対照）**
   ベースラインの全営業日に対して、投資上限だけを $C_{\text{uniform}} = C_{\text{base}} \times r$（または実平均投入枠 $\bar{C}$）に一律設定し、**100株単元（米国は1株単元）の整数丸めまで再実行した実測損益** です。

**合格の必要条件**：ゲート戦略が「純粋な選別アルファを持つ」と言えるためには、少なくとも **Uniform-Linear と Uniform-Cap の双方の損益を上回り、かつベースラインの年率 Sharpe を上回る** 必要があります。

---

## 2. Uniform-Cap 監査関数の完全実装（日本株・米国株）

第2章の `evaluate_jp_daily_fixed_capital` / `evaluate_us_daily_fixed_capital` をラップし、あらゆるゲート候補を自動的に3基準（ベースライン・Uniform-Linear・Uniform-Cap）で判定する関数を実装します。

```python
from typing import Any


def evaluate_jp_gate_with_uniform_control(
    label: str,
    selected_trades: list[dict[str, Any]],
    all_long_trades: list[dict[str, Any]],
    base_report: FixedCapitalReport,
    total_business_days: int,
    is_cash_veto: bool = True,
    base_margin_cap_jpy: float = 2_400_000.0,
) -> dict[str, Any]:
    """日本株ゲート戦略を Uniform-Linear および 100株丸め Uniform-Cap 対照と比較する。"""
    report = evaluate_jp_daily_fixed_capital(
        selected_trades,
        margin_cap_jpy=base_margin_cap_jpy,
        total_business_days=total_business_days,
    )
    out: dict[str, Any] = {
        "variant": label,
        "trades": report.trade_count,
        "net_pnl_jpy": report.total_net_pnl,
        "avg_net_bp": report.avg_net_bp,
        "worst_day_jpy": report.worst_day_pnl,
        "sharpe": report.annualized_sharpe,
        "beats_always_long_sharpe": report.annualized_sharpe > base_report.annualized_sharpe,
        "beats_always_long_pnl": report.total_net_pnl > base_report.total_net_pnl,
    }
    if is_cash_veto:
        active_rate = report.trade_count / max(base_report.trade_count, 1)
        uniform_linear_pnl = round(base_report.total_net_pnl * active_rate, 2)
        uniform_cap_report = evaluate_jp_daily_fixed_capital(
            all_long_trades,
            margin_cap_jpy=base_margin_cap_jpy * active_rate,
            total_business_days=total_business_days,
        )
        out.update(
            {
                "active_rate": round(active_rate, 4),
                "uniform_linear_pnl_jpy": uniform_linear_pnl,
                "uniform_cap_pnl_jpy": uniform_cap_report.total_net_pnl,
                "uniform_cap_sharpe": uniform_cap_report.annualized_sharpe,
                "beats_uniform_linear": report.total_net_pnl > uniform_linear_pnl,
                "beats_uniform_cap": report.total_net_pnl > uniform_cap_report.total_net_pnl,
            }
        )
    return out


def evaluate_us_gate_with_uniform_control(
    label: str,
    selected_by_day: dict[str, list[dict[str, Any]]],
    all_by_day: dict[str, list[dict[str, Any]]],
    base_report: FixedCapitalReport,
    total_business_days: int,
    webull_cash_usd: float = 228.20,
) -> dict[str, Any]:
    """米国株ゲート戦略を Uniform-Linear / Uniform-Cap 対照と比較する。"""
    report = evaluate_us_daily_fixed_capital(
        selected_by_day,
        webull_cash_usd=webull_cash_usd,
        total_business_days=total_business_days,
    )
    active_rate = report.trade_count / max(base_report.trade_count, 1)
    uniform_linear_usd = round(base_report.total_net_pnl * active_rate, 2)
    uniform_cap_report = evaluate_us_daily_fixed_capital(
        all_by_day,
        webull_cash_usd=webull_cash_usd * active_rate,
        total_business_days=total_business_days,
    )
    return {
        "variant": label,
        "trades": report.trade_count,
        "net_pnl_usd": report.total_net_pnl,
        "uniform_linear_usd": uniform_linear_usd,
        "uniform_cap_usd": uniform_cap_report.total_net_pnl,
        "sharpe": report.annualized_sharpe,
        "beats_uniform_linear": report.total_net_pnl > uniform_linear_usd,
        "beats_uniform_cap": report.total_net_pnl > uniform_cap_report.total_net_pnl,
        "beats_base_sharpe": report.annualized_sharpe > base_report.annualized_sharpe,
    }
```

---

## 3. 実測ログ①：日本株269営業日での CashVeto・35bpショート反転「全12戦全敗」

日本株オーバーナイト（`2025-09-01 .. 2026-10-15`、269営業日）に対し、Walk-Forward で強い OOS IC（`+0.1427`）を出した `Core22` モデルを使って Cash Veto（弱気日は0株退避）と FlipShort（弱気日は `35 bp` コストでショート反転）を全12パターン検証した結果です。

| 戦略バリアント | 稼働日数 | 純損益 (JPY) | Uniform-Cap (JPY) | Uniform-Linear (JPY) | 平均 (bp) | 最悪日 (JPY) | 年率Sharpe | 判定 |
|---|---:|---:|---:|---:|---:|---:|---:|---|
| `AlwaysLong_Open_5bp` | 269/269 | **3,144,591** | - | - | 57.61 | -226,114 | **2.375** | BASELINE |
| `AlwaysShort_Open_35bp` | 269/269 | -5,434,431 | - | - | -97.29 | -457,979 | -4.108 | BASELINE |
| `T3 Ridge Pred>0 CashVeto` | 169/269 | 1,821,435 | 1,488,921 | **1,975,598** | 51.33 | -226,114 | 1.610 | REJECT |
| `T3 Ridge Pred>0 FlipShort35bp` | 269/269 | -348,474 | - | - | -7.98 | -363,574 | -0.258 | REJECT |
| `T3 DayBlockBag Pred>0 CashVeto` | 179/269 | 2,430,502 | 1,823,275 | 2,092,498 | 64.84 | -226,114 | 2.056 | REJECT |
| `T3 DayBlockBag Pred>0 FlipShort35bp` | 269/269 | 954,330 | - | - | 15.30 | -226,114 | 0.710 | REJECT |
| `T3 DayBlockBag Pred>TrainQ20 CashVeto` | 208/269 | 2,220,085 | **2,526,332** | **2,431,506** | 51.02 | -226,114 | 1.801 | REJECT |
| `T3 DayBlockBag Pred>TrainQ20 FlipShort35bp` | 269/269 | 781,231 | - | - | 12.22 | -226,114 | 0.581 | REJECT |
| `T2 Ridge Pred>0 CashVeto` | 210/269 | 2,431,470 | **2,603,001** | **2,454,885** | 57.32 | -226,114 | 1.980 | REJECT |
| `T2 Ridge Pred>0 FlipShort35bp` | 269/269 | 1,236,747 | - | - | 23.42 | -233,242 | 0.921 | REJECT |
| `T2 DayBlockBag Pred>0 CashVeto` | 215/269 | 2,671,569 | 2,562,627 | 2,513,335 | 61.39 | -226,114 | 2.165 | REJECT |
| `T2 DayBlockBag Pred>0 FlipShort35bp` | 269/269 | 1,757,357 | - | - | 32.81 | -233,242 | 1.313 | REJECT |
| `T2 DayBlockBag Pred>TrainQ20 CashVeto` | 230/269 | 2,963,503 | 2,645,534 | 2,688,684 | 64.01 | -226,114 | 2.336 | REJECT |
| `T2 DayBlockBag Pred>TrainQ20 FlipShort35bp` | 269/269 | 2,449,521 | - | - | 46.04 | -233,242 | 1.838 | REJECT |

この敗北ログから読み取るべき教訓は極めて明快です。

1. **Uniform-Cap なしだと見抜けない偽アルファ**：`T2 Ridge Pred>0 CashVeto` は `+2,431,470 JPY` を稼ぎますが、同稼働率の一律縮小（Uniform-Cap `2,603,001 JPY`）に `17万円` 以上負けています。`T3 DayBlockBag Pred>TrainQ20 CashVeto` も Uniform-Cap（`2,526,332 JPY`）に `30万円` 負けています。
2. **最良の Cash Veto でも「0株退避」の機会損失に勝てない**：`T2 DayBlockBag Pred>TrainQ20 CashVeto` は確かに選別力があり、Uniform-Cap（`2,645,534 JPY`）を `+317,969 JPY` 上回り、平均純利も `64.01 bp` まで上げました。しかし **最悪日（`-226,114 JPY`）は避けられておらず**、さらにベース戦略の `+57.61 bp` の正ドリフトを39営業日ぶん完全に捨てたため、総損益・Sharpe の双方で `AlwaysLong` に届きませんでした。
3. **35bp コストのショート反転は構造的逆風**：`AlwaysShort` の期待値が `-97.29 bp/日` もある市場局面で、`35 bp` の往復ショートコストを払って逆張り反転すると、全バリアントで Sharpe が大きく毀損します。

---

## 4. 実測ログ②：米国株クロスマーケットの非単調メタ閾値崩壊

もう1つ、Uniform-Cap 監査が「危険な閾値チューニング」を食い止めた米国株パネル（`$228.20 / $221.55`、`370〜371日`）の実測例を見てみましょう。

| 米国株バリアント | 稼働日数 | 純損益 (USD) | Uniform-Linear (USD) | Uniform-Cap (USD) | 平均 (bp) | 年率Sharpe | 判定 |
|---|---:|---:|---:|---:|---:|---:|---|
| `Long-Only: AlwaysTrade_5bp` | 341/371 | `$134.21` | - | - | 20.85 | `0.764` | BASELINE |
| `Long-Only: MetaScore>=0.90` | 285/371 | `$92.19` | **`$112.17`** | **`$113.97`** | 17.81 | `0.552` | **REJECT** |
| `Long-Only: MetaScore>=0.95` | 258/371 | `$145.25` | `$101.54` | `$108.54` | 29.86 | `0.891` | 非単調注意 |
| `Long+Short: AlwaysTrade_5bp` | 340/370 | **`$314.38`** | - | - | 44.26 | **`1.502`** | BASELINE |
| `Long+Short: MetaScore>=0.90` | 280/370 | `$269.34` | `$258.90` | `$229.25` | 46.64 | `1.390` | **REJECT** |
| `Long+Short: MetaScore>=0.95` | 243/370 | `$221.50` | **`$224.69`** | `$155.47` | 44.73 | `1.194` | **REJECT** |

注目すべきは `Long-Only` の **`meta_score >= 0.90`（`$92.19`, Sharpe `0.552`）→ `>= 0.95`（`$145.25`, Sharpe `0.891`）という非単調な急変** です。真に単調な確信度スコアであれば、`0.90` でもベースラインを上回り、`0.95` でさらに改善するはずです。`0.90` で大きく崩れて `0.95` だけ飛び出すのは、特定の数日間の大勝・大敗をピンポイントで切り出した **閾値過学習（Threshold Overfitting）** の典型的なサインであり、実運用への採用は却下します。

---

## 5. 対比：「本物の Uniform 超過」はどう見えるのか

では逆に、Uniform 対照を本物として突破した戦略はどのような数字になるのでしょうか。本書で監査した別パネル（日中上昇モデル・416営業日）のアブレーション比較では、以下のような明確な超過が確認されました。

- ベース戦略 `P0`：`340` トレード、`+1,645,429 JPY`、年率 Sharpe `3.94`、最大DD `79,844 JPY`
- **一律拡大対照（Exposure-Scaled Control）**：改善版のトレード数 `375` に合わせてベース戦略を `375 / 340 = 1.1029倍` に一律拡大した損益 = **`+1,814,811 JPY`**
- **改善版（非対称バギング＋午後フィルタ＋市場上限）**：`375` トレード、**`+2,015,883 JPY`**、年率 Sharpe **`4.53`**、最大DD **`70,756 JPY`**

一律拡大対照に対して **`+201,072 JPY` の純超過収益** を出しながら、同時に **Sharpe 上昇（`3.94 -> 4.53`）と最大DD縮小（`79,844 -> 70,756 JPY`）** を達成しています。これが Uniform Control をパスする「本物の構造改善」の姿です。

次章では、こうした本物のエッジを安定抽出するための最初のモデリング武器である **`DayBlockBaggingRegressor`** の理論と完全実装へ進みます。
