---
title: "【改正建設業法・労務費勧告・2024年上限規制対応】ブラウザ完結16連ゲート『KENSETSUGATE』設計"
emoji: "🏗️"
type: "tech"
topics: ["compliance", "construction", "legaltech", "japan", "javascript"]
published: false
---

2024年4月からの **建設業時間外労働上限規制（単月100時間未満・2〜6ヶ月平均80時間以内）** の全面適用に続き、段階施行が進む **改正建設業法**（標準労務費を大きく下回る契約の禁止・資材高騰『おそれ情報』通知時の変更協議義務・著しく短い工期の禁止・特定建設業許可基準額の5,000万円／建築一式8,000万円への政令引上げ）により、元請・専門工事業の契約と工程チェックの負担が急増しています。

しかし工事別見積原価・法定福利費内訳・下請発注総額・作業員の月間残業時間は極めて機密性が高く、社外クラウドへそのまま送信できない情報ばかりです。そこで既存の15連ゲートへ第16の門 **[KENSETSUGATE（建設門）](https://pecopeko.github.io/kensetsugate/)** を追加し、**外部通信ゼロ（ブラウザ内完結）** で次を自動点検できるようにしました。

1. **2025年政令対応『特定建設業許可』閾値判定**: 一般工事業種 `5,000万円` ／ 建築一式工事 `8,000万円` ラインを下請発注総額から自動判定し、特定許可の要否を即時に可視化します。
2. **標準労務費・法定福利費ダンピング ＆ 価格転嫁拒否特約チェック**: 請負金額が標準労務費＋法定福利費合計を下回っていないか、また「物価変動による増額請求は認めない」といった NG 特約が含まれていないかをローカル検証します。
3. **2024年建設業残業上限 ＆ 法第19条/22条/24条/40条の3 監査証明書生成**: 単月・移動平均の残業時間判定に加え、丸投げ禁止・50日払・帳簿5年/10年保存を含む **『改正建設業法兼2024年工期残業上限 監査証明書（Markdown）』** を即生成します。

## 公開中の全16連ゲート（GitHub Pages・無料・登録不要）

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
16. **[KENSETSUGATE（建設門）](https://pecopeko.github.io/kensetsugate/)** — 改正建設業法（労務費勧告・価格協議・特定許可5,000万/8,000万）・2024年上限規制診断

Python製 **16ゲート統合オフライン一括審査CLI (`batch_compliance_suite_cli.py`)** および下請案件台帳CSV一括監査エンジン（`construction_kensetsugyo_2024_auditor.py`）も同梱済みです。
