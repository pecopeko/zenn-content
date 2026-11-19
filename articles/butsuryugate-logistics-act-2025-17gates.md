---
title: "【改正物流効率化法・荷待ち2h・標準的運賃対応】ブラウザ完結17連ゲート『BUTSURYUGATE』"
emoji: "🚛"
type: "tech"
topics: ["logistics", "compliance", "supplychain", "japan", "javascript"]
published: false
---

2024年問題（トラック運転者の時間外年960時間・拘束時間上限・継続休息確保）に続き、**改正物資流通効率化法** および **改正トラック運送事業法** により、年間取扱貨物 **9万トン以上の特定荷主における物流統括管理者(CLO)選任・中長期計画・荷待ち＋荷役2時間以内目標・標準的運賃準拠・実運送体制管理簿（2次請け以内抑制）** の実務点検が急務となっています。

運賃表・ルート別物量・荷待ち待機実績・下請階層明細は取引機密を含むため外部クラウドAIへ送信できないケースが大半です。そこで既存の16連ゲートへ第17の門 **[BUTSURYUGATE（物流門）](https://pecopeko.github.io/butsuryugate/)** を追加し、**外部通信ゼロ（ブラウザローカル完結）** で次を自動点検できるようにしました。

1. **特定荷主（9万トン/年）CLO要件 ＆ 標準的運賃90%水準比較**: 年間取扱重量からの特定荷主判定と、国交省「標準的運賃」目安および燃料サーチャージ・附帯作業費の適正収受をローカル検証します。
2. **荷待ち＋荷役2時間ルール ＆ 2024年運転者改善基準告示チェック**: 1運行あたりの荷待ち・荷役合計時間（`2.0h` 超警告）、運転者の月間拘束（`310h` 上限）と1日休息期間（`9h` 下限）を自動判定します。
3. **実運送体制管理簿（2次請け以内抑制）＆ 監査証明書生成**: 書面交付・待機時間料別建て・記録保存を含む **『改正物流効率化法・トラックGメン・改善基準告示2024 監査証明書（Markdown）』** を即時生成します。

## 公開中の全17連ゲート（GitHub Pages・無料・登録不要）

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
17. **[BUTSURYUGATE（物流門）](https://pecopeko.github.io/butsuryugate/)** — 改正物流効率化法（9万t CLO・荷待ち2h）・標準的運賃＆トラック2024年改善基準告示診断

Python製 **17ゲート統合オフライン一括審査CLI (`batch_compliance_suite_cli.py`)** および輸送ルート別CSV一括監査エンジン（`logistics_butsuryu_2025_auditor.py`）も同梱済みです。
