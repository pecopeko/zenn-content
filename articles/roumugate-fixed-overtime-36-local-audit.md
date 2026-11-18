---
title: "【2024年4月改正 労働条件通知書＆固定残業・36協定80h平均】従業員データ送信ゼロ・ブラウザ完結の労務監査『ROUMUGATE』を作った"
emoji: "🕒"
type: "tech"
topics: ["javascript", "japan", "compliance", "hr", "saas"]
published: false
---

労働条件通知書・賃金台帳・36協定の実残業時間リストには氏名・基本給・労働時間など機微な労務情報が含まれるため、社外SaaSやLLMへ気軽に貼り付けにくい領域です。そこで既存の8連ゲートに **第9の門 [ROUMUGATE（労務門）](https://pecopeko.github.io/roumugate/)** を追加し、**外部通信ゼロ（100%ブラウザ内完結）** で労基署臨検・IPO労務デューデリジェンス対策の事前セルフチェックを自動化しました。

## ROUMUGATE（労務門）でできる4つの決定論的監査

1. **2024年4月改正 労基法施行規則第5条 記載漏れチェック**:
   - 全労働者対象の **「就業場所・業務の変更の範囲」**
   - 有期契約対象の **「更新上限（通算期間・回数）」** と **「無期転換申込機会・転換後条件」**
2. **固定残業代（みなし残業）最高裁3要件の検証**:
   - 基本給との金額明確区分／見込時間数の明記／超過額の別途・追加支給合意を検査
3. **月60時間超 1.50倍割増（中小企業猶予廃止済み）＋最低賃金・未払差額の1円単位算出**:
   - 月60時間迄 `1.25倍`、月60時間超 `1.50倍`、深夜加算 `+0.25倍`、法定休日 `1.35倍` を積み上げ、固定残業代との不足精算額と時間給最低賃金違反を判定
4. **36協定 12か月全移動平均チェック**:
   - 年6回（月45h超）制限・年720h制限・単月100h未満・**連続する2〜6か月すべての区間の移動平均80時間以下** を一括検証

## 全9連ローカルゲート ＆ オフラインCLI

- Web App一覧: [KUROGATE](https://pecopeko.github.io/kurogate/) / [YAKKIGATE](https://pecopeko.github.io/yakkigate/) / [ZENGINGATE](https://pecopeko.github.io/zengingate/) / [KEIYAKUGATE](https://pecopeko.github.io/keiyakugate/) / [TOKUSHOGATE](https://pecopeko.github.io/tokushogate/) / [DENCHOGATE](https://pecopeko.github.io/denchogate/) / [KOJINGATE](https://pecopeko.github.io/kojingate/) / [GENSENGATE](https://pecopeko.github.io/gensengate/) / **[ROUMUGATE](https://pecopeko.github.io/roumugate/)**
- Python一括処理 CLI (`batch_compliance_suite_cli.py`) も全9ゲートに対応し、完全オフライン・SHA-256監査証明書を出力できます。
