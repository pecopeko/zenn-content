---
title: "第8章：決済タイミングの過学習検証＆実運用投入前チェックリスト"
---

## 1. 最後の罠：「直近1ヶ月（27日）の幸運」で決済ルールを書き換える悲劇

エントリーモデルとサイジングの監査を通過したあと、多くの開発者が最後にもうひと工夫したくなるのが **「翌朝の決済タイミング最適化（Exit-Wait Optimization）」** です。

- 寄付き（`open_price`）で機械的に成行決済するのではなく、
- 10分待ってから決済（`wait_10_price`）、15分待ってから決済（`wait_15_price`）、
- あるいは「寄付きで含み損のときだけ15分耐える（`conditional_loss_wait_15_price`）」

こうしたルールを直近1ヶ月のトレード履歴で試すと、驚くほど素晴らしい結果が出ることがあります。しかし長期履歴と並べて検証すると、その大半は **小標本レジーム分散（Small-Sample Regime Variance）** に過ぎません。

### 実パネル検証：日本株 `225日` 長期履歴 vs `27日` 直近窓の完全逆転劇

オーバーナイト戦略の翌朝決済履歴（日本株長期 `225営業日`・日本株直近 `27営業日`・米国株長期 `244営業日`）を、第2章の厳密な固定資本会計（日本株 `2.4M JPY` 100株丸め、米国株 `/150.0` で USD 復元後 `$221.55` 1株丸め）で横並び比較した実測結果がこちらです。

#### ① 日本株 長期履歴（`225営業日`）vs 直近窓（`27営業日`）

| 決済ポリシー | 長期225日 純損益 (JPY) | 長期225日 最悪日 (JPY) | 長期225日 Sharpe | 直近27日 純損益 (JPY) | 直近27日 Sharpe |
|---|---:|---:|---:|---:|---:|
| **`open_price`（寄付き即時決済）** | **`+3,882,536`** | **`-158,336`** | **`3.654`** | `+112,442` | `1.203` |
| `immediate_price` | `+3,103,236` | `-166,061` | `2.914` | `+124,142` | `1.267` |
| `wait_10_price`（10分待機） | `+2,859,636` | `-207,676` | `2.373` | `+184,542` | `1.933` |
| `wait_15_price`（15分待機） | `+2,899,586` | **`-307,776`** | `2.296` | **`+282,942`** | `2.885` |
| `conditional_loss_wait_10_price` | `+3,725,036` | `-207,676` | `3.206` | `+178,942` | `1.910` |
| `conditional_loss_wait_15_price` | `+3,869,686` | **`-307,776`** | `3.236` | `+279,042` | **`3.107`** |

#### ② 米国株 長期履歴（`244営業日`、`/150.0` USD復元・`$221.55` 枠・`214/244` 買付可能）

| 決済ポリシー | 純損益 (USD) | 円換算 (156円) | 平均 (bp) | 最悪日 (USD) | 年率 Sharpe |
|---|---:|---:|---:|---:|---:|
| **`open_price`（寄付き即時決済）** | **`$238.98`** | `37,280 JPY` | **`55.78`** | `$-18.36` | **`2.424`** |
| `immediate_price` | `$183.64` | `28,648 JPY` | `42.48` | `$-22.55` | `1.715` |
| `wait_10_price` | `$176.78` | `27,578 JPY` | `39.46` | `$-18.00` | `1.525` |
| `wait_15_price` | `$217.74` | `33,967 JPY` | `49.56` | `$-21.55` | `1.789` |
| `conditional_loss_wait_10_price` | `$216.27` | `33,739 JPY` | `49.08` | `$-18.00` | `2.063` |
| `conditional_loss_wait_15_price` | `$242.51` | `37,831 JPY` | `54.96` | `$-21.55` | `2.289` |

この対比は、全クオンツが壁に貼っておくべき教訓です。

- **直近 `27営業日` だけ見ると**：`wait_15_price` が `+282,942 JPY`（Sharpe `2.885`）、`conditional_loss_wait_15` が Sharpe `3.107` で、`open_price`（`+112,442 JPY`, Sharpe `1.203`）を2.5倍以上引き離して見えます。
- **しかし長期 `225営業日` で検証すると**：結論は **180度逆転** します。`open_price`（`+3,882,536 JPY`, Sharpe **`3.654`**, 最悪日 **`-158,336 JPY`**）が全遅延決済を圧倒し、`wait_15` は利益を **約98万円も失ううえ、最悪日損失を `-158,336 -> -307,776 JPY` へと約2倍に悪化** させています。
- **「含み損のときだけ15分待つ」も危険**：一見すると長期でも `+3,869,686 JPY` と肉薄しますが、暴落日の傷口を広げて最悪日が `-307,776 JPY` に倍増し、Sharpe は `3.654 -> 3.236` へ低下します。米国株244日でも同様に、リスク調整後リターン（Sharpe `2.424` vs `2.289`）では機械的な `open_price` が勝利しています。

27営業日程度の短い直近成績に反応して決済ルールを複雑化させるのは、**典型的な小標本過学習＋テールリスクの隠蔽** です。

---

## 2. エンドツーエンド再現監査ハーネス（1コマンド実行テンプレート）

第1章〜第7章で実装した全コンポーネントを1つの監査ドライバに束ね、新しい特徴量やサイジング案を提案するたびに **自動で PASS / REJECT と JSON 証跡を吐き出す再現ハーネス** の骨格を示します。

```python
from __future__ import annotations

import json
from datetime import datetime, timezone
from pathlib import Path
from typing import Any

import numpy as np


def run_full_strategy_audit(
    X_full: np.ndarray,
    y: np.ndarray,
    dates: list[str],
    feature_names: list[str],
    payoff_dates: list[str],
    long_by_date: dict[str, dict[str, Any]],
    oos_start_date: str,
    alpha: float = 50.0,
    refit_step: int = 20,
    n_estimators: int = 50,
    out_json_path: Path | None = None,
) -> dict[str, Any]:
    """1コマンドで全AFML監査（静的PCA/MDA -> 負MDA自動剪定 -> WF -> 3段階サイジング vs Uniform-Cap）を実行する。"""
    split_idx = sum(1 for d in dates if d < oos_start_date)
    if split_idx < 40 or (len(dates) - split_idx) < 40:
        raise ValueError("学習・OOSともに最低40営業日以上のサンプルが必要です。")

    # 1. 静的分割で PCA Kaiser & OOS MDA 監査を実行（リーク防止のため学習窓のみで標準化）
    from sklearn.preprocessing import StandardScaler

    scaler = StandardScaler()
    X_tr_s = scaler.fit_transform(X_full[:split_idx])
    X_te_s = scaler.transform(X_full[split_idx:])
    base_model = DayBlockBaggingRegressor(
        n_estimators=n_estimators, alpha=alpha, random_state=42
    )
    ortho = run_afml_orthogonality_audit(
        model=base_model,
        X_train=X_tr_s,
        y_train=y[:split_idx],
        dates_train=np.asarray(dates[:split_idx]),
        X_oos=X_te_s,
        y_oos=y[split_idx:],
        feature_names=feature_names,
        random_state=42,
        n_repeats=20,
    )

    # 2. 明確な負の OOS MDA 特徴量（下位ドラッグ）を特定して剪定候補を作成
    pruned_idx = [
        i for i, row in enumerate(ortho.feature_audit) if row.oos_mda_ic > -0.01
    ]
    pruned_names = [feature_names[i] for i in pruned_idx]
    X_pruned = X_full[:, pruned_idx]

    # 3. 剪定前 vs 剪定後で拡大窓 Walk-Forward + 3段階確信度サイジングを Uniform-Cap 対照と対決
    wf_before = run_walk_forward_with_quantiles(
        X_full, y, dates, split_idx, alpha=alpha, refit_step=refit_step, n_estimators=n_estimators
    )
    wf_after = run_walk_forward_with_quantiles(
        X_pruned, y, dates, split_idx, alpha=alpha, refit_step=refit_step, n_estimators=n_estimators
    )

    res_before = evaluate_conviction_sizing_vs_uniform_cap(
        "Before_Pruning", payoff_dates, long_by_date, wf_before
    )
    res_after = evaluate_conviction_sizing_vs_uniform_cap(
        "After_MDA_Pruning", payoff_dates, long_by_date, wf_after
    )

    verdict = (
        "GO_CANDIDATE"
        if (
            ortho.edge_on_top_pcs
            and res_after["beats_uniform_pnl"]
            and res_after["beats_uniform_sharpe"]
        )
        else "REJECT_OR_NEEDS_WORK"
    )

    payload: dict[str, Any] = {
        "audited_utc": datetime.now(timezone.utc).isoformat(),
        "split_date": oos_start_date,
        "train_days": split_idx,
        "oos_days": len(dates) - split_idx,
        "orthogonality": {
            "kaiser_effective_dims": ortho.kaiser_effective_dims,
            "total_features": ortho.total_features,
            "pca_top_k_oos_ic": ortho.pca_top_k_oos_ic,
            "pca_bottom_k_oos_ic": ortho.pca_bottom_k_oos_ic,
            "edge_on_top_pcs": ortho.edge_on_top_pcs,
            "flagged_overfit_features": ortho.flagged_overfit_features,
        },
        "pruned_feature_count": len(pruned_names),
        "dropped_features": sorted(set(feature_names) - set(pruned_names)),
        "sizing_before": res_before,
        "sizing_after": res_after,
        "final_verdict": verdict,
    }

    if out_json_path is not None:
        out_json_path.write_text(
            json.dumps(payload, ensure_ascii=False, indent=2), encoding="utf-8"
        )
    return payload
```

> **実務上の厳格な注意（データスヌーピング防止）**：
> 上記テンプレートのように「同じ OOS 期間で OOS MDA 剪定を行い、同じ OOS 期間でサイジングを評価する」ことは、**診断と仮説生成（探索フェーズ）** としては極めて強力ですが、厳密には1段ぶんの選択バイアスを含みます。最終的な本番投入前には、剪定後の特徴量セット（`Pruned18` 等）を固定したまま、**まだ一度も触っていない未来のフォワード期間（Paper Trading / Live Shadow 運用）** で最低数週間〜数ヶ月の再現性を確認する運用ルールを強く推奨します。

---

## 3. 実運用投入前 Go / No-Go 最終チェックリスト（10項目）

新しい戦略やモデル改修を本番環境へマージする前に、次の10項目をすべて機械的にチェックし、1つでも欠ければ迷わず棄却（REJECT）する運用を徹底してください。

1. **欠損処理**：ターゲットおよび特徴量に `.fillna(0)` が一切入っておらず、除外行数・除外列数がログに記録されているか
2. **因果タイムスタンプ**：全特徴量の確定時刻（`audit_*_available_at`）が発注締切（例：`15:25 JST`）以前であることを自動アサートしたか（違反 `0` 件）
3. **前処理リーク遮断**：`StandardScaler`・PCA・分位点閾値（`Q20` / `Q80`）の `fit` がすべて Walk-Forward の学習窓 `[:start]` 内に閉じているか
4. **日次ブロック相関統制**：多銘柄・日次パネルに対して行ブートストラップではなく `DayBlockBaggingRegressor`（またはパージ付き時系列CV）を用いているか
5. **非複利・実単元丸め**：日本株100株単元・米国株1株上限（成行バッファ込）・非対称往復コスト（Long `5 bp` / Short `35 bp`）で損益を再計算したか
6. **ゼロトレード営業日の計上**：未稼働日を `0.0` として全営業日数で年率 Sharpe（`ddof=1`）を計算したか
7. **Uniform-Cap Control 突破**：フィルタ・サイジング案が、同一平均資本の Uniform-Cap 対照を **純損益・年率 Sharpe の双方で上回っているか**
8. **閾値の単調性**：`meta_score >= 0.90` と `>= 0.95` のように、隣接閾値で結果が逆転する非単調スパイクではないか
9. **PCA ＆ OOS MDA 整合性**：`edge_on_top_pcs == True` であり、インサンプル重要度が高く OOS MDA が負のドラッグ特徴量を放置していないか
10. **小標本バイアスの排除**：20〜30営業日程度の直近成績だけで決済待機ルールやパラメータを書き換えず、200営業日以上の長期履歴と最悪日テール損失（Worst Day PnL）を確認したか

---

## おわりに

機械学習クオンツの世界で長く生き残る秘訣は、「派手な最新モデルで Train IC `0.80` を出すこと」ではありません。**「9割の偽陽性を冷徹に棄却できる監査ハーネスを先に持ち、すべての関所を生き残った1割の地味だが堅牢なエッジだけに実弾を載せること」** です。

本書で実装したコード群が、読者の皆さまのバックテストから偽の聖杯を一掃し、再現可能な実運用エッジを掘り当てるための強固な土台となれば幸いです。
