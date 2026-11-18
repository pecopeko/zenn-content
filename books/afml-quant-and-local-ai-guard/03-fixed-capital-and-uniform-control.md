---
title: "第3章: 固定資本・100株単元丸めと対一律縮小対照（Uniform Control）の罠"
---

パーセント複利リターンや株数端数許可のバックテストは、日本株 100 株単元・米国株 1 株整数丸め・信用枠固定上限のある小〜中規模口座の実運用と大きく乖離します。

## 実運用準拠の固定資本シミュレーション原則

1. **欠損値を 0 で埋めない（No `.fillna(0)`）**: 欠損日や取引不可銘柄を無言で `0 bp` にせず明示除外する。
2. **100株単元（日本株）／1株整数（米国株）の切り捨て**:

```python
import math

def compute_jp_lot_shares(cap_jpy: float, entry_price: float, lot_size: int = 100) -> int:
    if entry_price <= 0:
        return 0
    return math.floor(cap_jpy / (entry_price * lot_size)) * lot_size
```

3. **対一律縮小対照（Uniform-Cap Control）を必ず並べる**:
   フィルターで売買回数を 85% に減らしたり平均投下資金を減らしただけで Sharpe が上がったと錯覚するのを防ぐため、「全営業日に同じ平均資金（例: `1,683,271 JPY`）で均等投入したベンチマーク」に勝っているか検証します。

実検証（269営業日・2,400,000 JPY上限）では、2値の現金退避（Cash Veto）だと AlwaysLong のプラスドリフト（`+57.61 bp`）を捨ててしまい劣後する一方、`Pruned18 + DayBlockBagging(alpha=50)` による **3段階ポジションサイズ調整（80万 / 160万 / 240万円）** では均等配分対照より **`+545,251 JPY`・Sharpe `2.501`（対照 `2.328`）** を記録しました。
