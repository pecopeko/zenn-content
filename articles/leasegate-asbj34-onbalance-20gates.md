---
title: "【新リース会計基準(第34号) PV試算・組込リース対応】ブラウザ完結20連『LEASEGATE』"
emoji: "🏢"
type: "tech"
topics: ["accounting", "compliance", "lease", "japan", "javascript"]
published: true
---

2027年4月以降開始事業年度から強制適用（2025年4月以降早期適用）される **新リース会計基準（企業会計基準第34号）** により、従来は賃貸借処理（オフバランス）されていたオフィス賃貸・倉庫・社用車・複合機・専有ITサーバーや委託契約内の **組込リース** も、原則すべて **使用権資産 ＆ リース負債（割引現在価値 PV）** としてB/Sオンバランス計上が必要になりました。

オフィス賃料一覧や契約期間、追加借入利子率（IBR）、業務委託条件は経営機密のため外部クラウドAIへ送信できません。そこで節目の第20の門として **[LEASEGATE（リース門）](https://pecopeko.github.io/leasegate/)** を追加公開し、**外部通信ゼロ（ブラウザローカル完結）** で次を自動試算・監査できるようにしました。

1. **割引現在価値（PV）＆ 初年度P/L影響（償却＋利息）の即時計算**: 月額賃料・期間・IBRからリース負債初期計上額と1年目の減価償却費・支払利息をブラウザ内で算定します。
2. **短期リース(12か月以内)・少額リース(新品300万円以下)免除チェック**: 実務上の便法に該当するか自動判定し、計上漏れ（オフバランスリスク）を警告します。
3. **組込リース識別・監査書Markdown生成**: 非リース構成部分（保守料・共益費）配分や注記開示要件を含めた **『新リース会計基準 オンバランス影響試算・監査書』** を即時出力します。

## 公開中の全20連ゲート（GitHub Pages・無料・登録不要）

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
19. [KINYUGATE（金融門）](https://pecopeko.github.io/kinyugate/) — 資金決済法（50%供託・第2種100万円上限）・犯収法7年KYC・トラベルルール診断
20. **[LEASEGATE（リース門）](https://pecopeko.github.io/leasegate/)** — 新リース会計基準（企業会計基準第34号）PV試算・短期/少額300万免除・組込リース診断

Python製 **20ゲート統合オフライン一括審査CLI (`batch_compliance_suite_cli.py`)** および不動産・リース台帳CSV一括監査エンジン（`asbj34_new_lease_auditor.py`）も **[BOOTH Proスイート（v1.0.0-pro-kit）](https://github.com/pecopeko/horizon-money-studio/releases/tag/v1.0.0-pro-kit)** に同梱済みです。
