---
title: "第6章: 教師LLM知識蒸留（Qwen → DeBERTa INT8 ONNX）とサーバー費ゼロ公開構成"
---

厳密なチェックデジット・正規表現ルールだけでは、「間接プロンプトインジェクション」や「遠回しな薬機法表現・機密文脈（非公開TOB価格など）」を取りこぼします。一方で SaaS 側に推論GPUサーバーを常駐させると固定費が跳ね上がります。

## 解決策: Colab GPU で蒸留し、顧客のブラウザ上で INT8 推論する

1. **Teacher 学習データ生成（API課金 $0）**: Colab L4/A100 上に量子化オープン重みLLM（`Qwen2.5-14B-Instruct` 4-bit NF4）を載せ、ドメイン合成データ＋Teacher Soft Probabilities（先頭デジット確率分布）を生成。
2. **KL情報量＋Cross-Entropy 蒸留**: `ku-nlp/deberta-v3-base-japanese` へ知識蒸留し、`optimum[onnxruntime]` で約 45〜60 MB の `onnx/model_quantized.onnx` へ量子化。
3. **ブラウザ内ハイブリッド融合**: 確定ルールスコア $r \in [0, 1]$ とニューラル出力確率 $p_{\text{risk}}$ を次のように安全側（`max` 保証付き）で統合します。

```js
export function fuseRuleAndNeuralScore(ruleRisk, neuralProbs, safeIdx = 0, weight = 0.65) {
  const neuralRisk = 1 - (neuralProbs[safeIdx] ?? 0);
  return Math.min(1, Math.max(ruleRisk, (1 - weight) * ruleRisk + weight * neuralRisk));
}
```

この構成なら、運用ホスト費用 **\$0（GitHub Pages + Release ZIP）** のまま個人情報保護と高精度文脈判定を両立できます。
