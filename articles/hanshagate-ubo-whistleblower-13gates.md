---
title: "【反社排除条項・犯収法25%超UBO・公益通報者保護法対応】ブラウザ完結13連ゲート『HANSHAGATE』実務設計"
emoji: "🏛️"
type: "tech"
topics: ["compliance", "kyc", "whistleblower", "japan", "javascript"]
published: false
---

上場準備（IPO法務デューデリジェンス）や新規取引先オンボーディングにおいて、法務・総務部門が毎回確認を迫られる重要コンプライアンス要件が3つあります。

1. **反社会的勢力排除条項（東京都暴排条例・東証審査）**: 現在および過去5年・将来の非該当表明だけでなく、共生者（密接交際・資金提供）排除、**無催告即時解除**、および解除側への**損害賠償免責**まで条文に含まれているか。
2. **犯収法第4条『実質的支配者（UBO 25%超）』と確認記録7年保存**: 個人株主の直接保有だけでなく、50%超議決権を持つ法人株主経由の間接議決権合算（例：`直接15%＋支配法人経由22%＝37%`）や、該当なし時の代表者みなし判定・外国PEPs確認が漏れていないか。
3. **改正公益通報者保護法**: 公益通報対応業務従事者の書面指定（刑事罰対象守秘義務）、**退職後1年以内およびフリーランス・業務委託の保護対象化**、そして「犯人探し（特定・探索行為）」の明示的禁止。

これら機密株主構成・取引先契約条文を社外クラウドへ一切送信せずブラウザ上で完結診断し、**『反社会的勢力排除に関する表明・確約書 兼 UBO・内部通報体制確認書』** まで即時生成する第13のゲート **[HANSHAGATE（清廉門）](https://pecopeko.github.io/hanshagate/)** を公開しました。

## 公開中の全13連ゲート（GitHub Pages・無料・登録不要）

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
13. **[HANSHAGATE（清廉門）](https://pecopeko.github.io/hanshagate/)** — 反社排除（無催告解除/免責）・犯収法25%超UBO計算・改正公益通報者保護法チェック

法人オンボーディングCSVを一括審査する Python 製 **13ゲート統合オフラインCLI (`batch_compliance_suite_cli.py`)** および取引先KYC一括スクリーニングツール（`vendor_kyc_ubo_hansha_screener.py`）も同梱済みです。
