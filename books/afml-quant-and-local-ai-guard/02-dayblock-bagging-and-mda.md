---
title: "第2章: Day-Block Bagging と OOS MDA によるノイズ特徴量の外科的剪定"
---

金融時系列では各日の複数銘柄リターンが同一マクロショックを共有するため、通常の行単位ブートストラップ（Random Bootstrap）は **OOB（Out-of-Bag）サンプルへ同日の未来・同時刻情報を漏洩** させます。

## Day-Block Bagging 実装パターン

```python
import numpy as np
from sklearn.base import BaseEstimator, RegressorMixin
from sklearn.linear_model import Ridge

class DayBlockBaggingRegressor(BaseEstimator, RegressorMixin):
    def __init__(self, n_estimators: int = 50, alpha: float = 50.0, random_state: int = 42):
        self.n_estimators = n_estimators
        self.alpha = alpha
        self.random_state = random_state

    def fit(self, X: np.ndarray, y: np.ndarray, dates: np.ndarray):
        rng = np.random.default_rng(self.random_state)
        unique_days, inverse = np.unique(dates, return_inverse=True)
        day_indices = [np.where(inverse == i)[0] for i in range(len(unique_days))]
        self.models_ = []
        for _ in range(self.n_estimators):
            sampled_days = rng.integers(0, len(day_indices), size=len(day_indices))
            idx = np.concatenate([day_indices[d] for d in sampled_days])
            self.models_.append(Ridge(alpha=self.alpha).fit(X[idx], y[idx]))
        return self

    def predict(self, X: np.ndarray) -> np.ndarray:
        return np.mean([m.predict(X) for m in self.models_], axis=0)
```

## OOS MDA（Out-of-Sample 平均精度減少）による特徴量削減

実パネルデータ（`T2_us_macro` 予測・OOS 262日）に対する 20日ローリング再学習実験では、共線92特徴量から `Core22` へ削減した段階で Pearson IC が `0.0791 -> 0.1427` へ向上し、さらに OOS MDA がマイナスだった4特徴量（米10年債利回り、WTI原油、KOSPI、ドルインデックス1日変化）を外した **`Pruned18`** で `0.1436` まで引き締まりました。MDI（訓練内重要度）ではなく必ず **OOS側のMDA** で切るのが鍵です。
