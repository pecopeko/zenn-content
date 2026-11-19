---
title: "【外為法該非判定・みなし輸出特定類型・米国EAR De Minimis対応】完全ブラウザ完結14連ゲート『BOUEKIGATE（貿易門）』の設計とローカル監査CLI"
emoji: "🚢"
type: "tech"
topics: ["compliance", "exportcontrol", "security", "japan", "javascript"]
published: true
---

製造業・ソフトウェア開発・研究機関が海外顧客や国内居住の研究者へ製品・ソースコード・技術資料を提供する際、避けて通れないのが **安全保障貿易管理（外為法＋米国EAR再輸出規制）** の実務チェックです。

1. **外為法・輸出令別表第1（1〜15項リスト規制 / 16項キャッチオール規制）＆外国ユーザーリスト**: パラメータシートに基づく仕様判定に加え、非該当品でも大量破壊兵器（WMD）・通常兵器用途の確認および経産省『外国ユーザーリスト』・OFAC SDNリストとの照合が必須です。
2. **外為法第25条『みなし輸出』特定類型①②③**: 2022年5月施行の規制により、日本国内の役職員・共同研究者への技術提供であっても外国政府等の強い影響下（特定類型①〜③）にある場合は外為法上の許可要否確認・誓約書取得と **7年間保存** が求められます。
3. **米国EAR 15 CFR Part 734『De Minimis 25%/0%ルール』と暗号TSU公開免除**: 米国原産品・ソフトウェアを組み込む場合、一般仕向地では `25%` 超（懸念・禁輸国 E:1/D:5 では `0%` 超＝1円でも組込）で米国EARの域外適用対象となります。ただし、一般公開OSS暗号における **EAR 742.15(b) TSU通知要件** を満たせば例外扱いとなります。

製品の部品原価表や未公開技術仕様書を外部クラウドへ1バイトも送信せずブラウザ完結で点検し、**『安全保障貿易管理 該非判定書（パラメータシート要約）兼 みなし輸出特定類型確認書』** まで生成できる第14のゲート **[BOUEKIGATE（貿易門）](https://pecopeko.github.io/bouekigate/)** を公開しました。

## 公開中の全14連ゲート（GitHub Pages・無料・登録不要）

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
14. **[BOUEKIGATE（貿易門）](https://pecopeko.github.io/bouekigate/)** — 外為法1〜15項/16項キャッチオール・みなし輸出特定類型①②③・米国EAR De Minimis（25%/0%）診断

Python製 **14ゲート統合オフライン一括審査CLI (`batch_compliance_suite_cli.py`)** および製品・技術台帳CSV一括スクリーナー（`export_control_gaihi_screener.py`）も同梱済みです。
