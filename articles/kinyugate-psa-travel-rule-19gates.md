---
title: "【資金決済法50%供託・トラベルルール対応】ブラウザ完結19連ゲート『KINYUGATE』"
emoji: "🏦"
type: "tech"
topics: ["fintech", "compliance", "aml", "japan", "javascript"]
published: false
---

SaaS内ポイント・ギフト券・プリペイド残高（**前払式支払手段**）、法人間決済（**資金移動業**）、Web3・暗号資産ウォレットでは、**基準日未使用残高が1,000万円を超えた場合の2分の1（50%）発行保証金供託・信託保全**、**第2種資金移動業の1回100万円上限**、**FATF トラベルルール通知**、**犯収法に基づくeKYC・JAFIC疑わしい取引届出・確認記録7年保存** の継続的な点検が不可欠です。

残高集計値や送金上限設定、AML内部統制方針は金融庁検査・セキュリティ監査に関わる機密情報のため外部LLMへ送信できません。そこで18連ゲートへ第19の門 **[KINYUGATE（金融門）](https://pecopeko.github.io/kinyugate/)** を追加し、**外部通信ゼロ（ブラウザローカル完結）** で次を自動算出・監査できるようにしました。

1. **前払式2分の1（50%）法定供託＆不足額の即時計算**: 基準日未使用残高と現在の供託/信託保全額から、不足額をリアルタイム算出します。
2. **第2種100万円上限・トラベルルール・STR届出遅延・7年保存チェック**: 1回100万円超過、VASP間通知欠落、JAFIC届出10日超遅延、保存年数7年未満を一発検知します。
3. **自己点検証明書Markdown生成**: eKYC・OFAC/財務省制裁・95%コールドウォレット保全まで網羅した **『資金決済法・犯収法AML・トラベルルール 自己点検証明書』** を生成します。

## 公開中の全19連ゲート（GitHub Pages・無料・登録不要）

1. [KUROGATE（黒門）](https://pecopeko.github.io/kurogate/) — Mod-9/11＋可逆PIIマスキング＋Prompt Injection遮断
2. [YAKKIGATE（薬影）](https://pecopeko.github.io/yakkigate/) — 薬機法・景表法（No.1/二重価格）・ステマ規制監査
3. [ZENGINGATE（全銀門）](https://pecopeko.github.io/zengingate/) — 適格請求書T番号＋全銀半角カナ補正＋120バイト総合振込生成
4. [KEIYAKUGATE（契約門）](https://pecopeko.github.io/keiyakugate/) — 取適法60日サイト＋フリーランス新法＋印紙税節税試算
5. [TOKUSHOGATE（特商門）](https://pecopeko.github.io/tokushogate/) — 特商法第11条・第12条の6確認画面・悪質解約NG検査
6. [DENCHOGATE（電帳門）](https://pecopeko.github.io/denchogate/) — 電帳法第7条ファイル名正規化＋索引簿CSV＋事務処理規程
7. [KOJINGATE（個情門）](https://pecopeko.github.io/kojingate/) — 改正APPI第17〜39条＋外部送信規律第27条の12公表テーブル生成
8. [GENSENGATE（源泉門）](https://pecopeko.github.io/gensengate/) — 源泉徴収10.21%/20.42%・グロスアップ・2026年10月インボイス50%＆支払調書
9. [ROUMUGATE（労務門）](https://pecopeko.github.io/roumugate/) — 2024年4月労働条件通知書・固定残業3要件・月60h超1.5倍＆36協定
10. [JISGATE（基準門）](https://pecopeko.github.io/jisgate/) — 改正障害者差別解消法・JIS X 8341-3:2016 AA監査＆アクセシビリティ方針生成
11. [BOUHEKIGATE（防壁門）](https://pecopeko.github.io/bouhekigate/) — 経産省サプライチェーン評価★3/★4・PPC漏えいSLA・SPF/DMARC＆取引先回答書生成
12. [AIGOVGATE（統治門）](https://pecopeko.github.io/aigovgate/) — AI事業者ガイドライン第1.1版・著作権法30条の4・シャドーAI＆社内ガイドライン生成
13. [HANSHAGATE（清廉門）](https://pecopeko.github.io/hanshagate/) — 反社排除（無催告解除/免責）・犯収法25%超UBO計算・改正公益通報者保護法チェック
14. [BOUEKIGATE（貿易門）](https://pecopeko.github.io/bouekigate/) — 外為法1〜15項/16項キャッチオール・みなし輸出特定類型①②③・米国EAR De Minimis（25%/0%）診断
15. [TANSOGATE（炭素門）](https://pecopeko.github.io/tansogate/) — SSBJ基準・GHG Scope 1〜3二重算定・温対法SHK(3,000t)・グリーンウォッシュ診断
16. [KENSETSUGATE（建設門）](https://pecopeko.github.io/kensetsugate/) — 改正建設業法（労務費勧告・価格協議・特定許可5,000万/8,000万）・2024年上限規制診断
17. [BUTSURYUGATE（物流門）](https://pecopeko.github.io/butsuryugate/) — 改正物流効率化法（9万t CLO・荷待ち2h）・標準的運賃＆トラック2024年改善基準告示診断
18. [TORIHIKIGATE（取適門）](https://pecopeko.github.io/torihikigate/) — 2026年取適法（従業員300/100人基準・手形禁止・価格転嫁協議・金型管理）診断
19. **[KINYUGATE（金融門）](https://pecopeko.github.io/kinyugate/)** — 資金決済法（50%供託・第2種100万円上限）・犯収法7年KYC・トラベルルール診断

Python製 **19ゲート統合オフライン一括審査CLI (`batch_compliance_suite_cli.py`)** および決済・AML CSV一括監査エンジン（`fintech_kinyu_aml_2026_auditor.py`）も **[BOOTH Proスイート（v1.0.0-pro-kit）](https://github.com/pecopeko/horizon-money-studio/releases/tag/v1.0.0-pro-kit)** に同梱済みです。
