---
title: "第1章: バックテストはなぜ本番で壊れるか — 静的ホールドアウトと時系列リークの罠"
---

日本株および日米クロスマーケットの短期クオンツモデルで最も起きやすい失敗は、**訓練期間の高い情報係数（IC）がそのまま本番へ引き継がれない現象**です。

たとえば日次リターン予測で `Full 92特徴量` の線形モデルを静的ホールドアウトで組むと、訓練期間 `IC = 0.7792` を記録してもアウトオブサンプル（OOS）で `IC = -0.0617` へ反転するケースが実測されます。主因は次の3点です。

1. **同日観測値の重複・高共線性（Collinear Lags）**: 相関の高い海外指数・為替・ボラティリティ指標を無選別に投入すると、自由度だけが増え回帰係数が不安定化します。
2. **タイムスタンプ・リーク（15:25 JST 境界違反）**: 東証の引け間際〜引け後に観測可能なデータ（米国プレマーケットや遅延更新指標）が監査カラムなしで混入する問題。
3. **非定常レジーム変化**: 相場ボラティリティが反転した際、20日程度のローリング再学習（Expanding Walk-Forward Refit）を行わないと古い係数が逆効果になります。

## 必須のタイムスタンプ監査コード

```python
from datetime import datetime

def count_post_cutoff_leaks(dates: list[datetime], available_at: list[datetime | None]) -> int:
    violations = 0
    for d_raw, av in zip(dates, available_at):
        if av is None:
            continue
        cutoff = d_raw.replace(hour=6, minute=25, tzinfo=av.tzinfo)  # 15:25 JST = 06:25 UTC
        if av > cutoff:
            violations += 1
    return violations
```

特徴量を多くする前に、`audit_*_available_at` 列を強制チェックして **リーク 0 件** を最初に保証することが出発点になります。
