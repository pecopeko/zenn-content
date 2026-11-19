---
title: "【SSBJ開示基準・温対法SHK・グリーンウォッシュ対応】ブラウザ完結15連ゲート『TANSOGATE』実務設計"
emoji: "🌱"
type: "tech"
topics: ["sustainability", "esg", "compliance", "japan", "javascript"]
published: true
---

2025年3月に確定した **SSBJサステナビリティ開示基準（S1 / S2）** や **温対法SHK制度（特定排出者 3,000 t-CO2/年 報告義務）**、さらに景品表示法・環境省「環境表示ガイドライン」に基づく **グリーンウォッシュ（根拠なきカーボンニュートラル訴求）対策** は、上場企業だけでなく中堅メーカー・物流・IT受託企業でも取引先要請として急速に広がっています。

実務上、各拠点の電気検針票・燃料伝票・調達支出額（Scope 3 算定基礎）・非化石証書やJ-クレジットの保有数量は機密性の高い原価情報を含むため、社外SaaSやクラウドAIへそのままアップロードしたくないケースが大半です。そこで既存の14連ゲートに第15の門 **[TANSOGATE（炭素門）](https://pecopeko.github.io/tansogate/)** を追加し、**外部通信ゼロ（ブラウザ内完結）** で次を一括監査できるようにしました。

1. **Scope 1（燃料）＋ Scope 2 デュアル算定（ロケーション基準 / マーケット基準 − 非化石証書）**: 電力系統平均係数と調整後排出係数の並列算定に対応し、非化石証書充当後のマーケット排出量まで即時計算します。
2. **温対法SHK（3,000 t-CO2/年）自動判定**: ロケーション基準の Scope 1 + Scope 2 合計が `3,000 t-CO2` を超える特定排出者ラインを自動検出し、EEGS定期報告準備の有無を点検します。
3. **残余純排出量に基づくグリーンウォッシュ検査**: 総排出量（Scope 1 + Scope 2 Mkt + Scope 3 カテゴリ1）から J-クレジット償却量を差し引いても `Net > 0 t-CO2` のまま「カーボンニュートラル／実質ゼロ」を訴求していないかチェックします。

## 公開中の全15連ゲート（GitHub Pages・無料・登録不要）

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
15. **[TANSOGATE（炭素門）](https://pecopeko.github.io/tansogate/)** — SSBJ基準・GHG Scope 1〜3二重算定・温対法SHK(3,000t)・グリーンウォッシュ診断

Python製 **15ゲート統合オフライン一括審査CLI (`batch_compliance_suite_cli.py`)** および複数拠点・グループ会社GHG台帳CSV一括監査エンジン（`ssbj_ghg_scope123_carbon_auditor.py`）も同梱済みです。
