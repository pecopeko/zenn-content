---
title: "【2026年取適法・300/100人基準・手形禁止対応】ブラウザ完結18連『TORIHIKIGATE』"
emoji: "⚖️"
type: "tech"
topics: ["compliance", "legal", "subcontract", "japan", "javascript"]
published: true
---

2026年施行の **取適法（中小受託取引適正化法 / 改正下請法）** では、従来の資本金基準（増資・減資で対象外となる抜け穴）に加えて **従業員数基準（製造・修理・特定運送委託は300人超→300人以下、役務・情報成果物は100人超→100人以下）** が導入され、**約束手形の交付禁止・割引料/振込手数料の受注者転嫁禁止・価格転嫁協議無視の禁止・特定運送委託の追加** への実務対応が必須となりました。

取引先ごとの資本金・従業員数・支払サイト・手形/決済方法・金型保管条件・単価交渉履歴は機密性が極めて高く、外部クラウドAIへ送信できません。そこで17連ゲートへ第18の門 **[TORIHIKIGATE（取適門）](https://pecopeko.github.io/torihikigate/)** を追加し、**外部通信ゼロ（ブラウザローカル完結）** で次を自動判定できるようにしました。

1. **資本金 ＋ 従業員数300人/100人二重適用判定**: 資本金1,000万円同士で従来の資本金基準に非該当でも、従業員数（420人→85人等）から取適法の対象取引を自動特定します。
2. **手形廃止・60日現金払・価格据置・金型無償保管違反チェック**: 約束手形や割引料/ファクタリング手数料控除・受領後60日超の支払・価格据置・6か月以上の金型無償保管を一発検出します。
3. **監査証明書Markdown生成**: 4条書面交付・特定運送委託・5条書類2年保存を含む **『2026年 取適法遵守監査証明書（Markdown）』** を即時生成します。

## 公開中の全18連ゲート（GitHub Pages・無料・登録不要）

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
18. **[TORIHIKIGATE（取適門）](https://pecopeko.github.io/torihikigate/)** — 2026年取適法（従業員300/100人基準・手形禁止・価格転嫁協議・金型管理）診断

Python製 **18ゲート統合オフライン一括審査CLI (`batch_compliance_suite_cli.py`)** および取引先別CSV一括監査エンジン（`toriteki_shitauke_2026_auditor.py`）も **[BOOTH Proスイート（v1.0.0-pro-kit）](https://github.com/pecopeko/horizon-money-studio/releases/tag/v1.0.0-pro-kit)** に同梱済みです。
