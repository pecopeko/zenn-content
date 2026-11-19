---
title: "【AI事業者ガイドライン・著作権法30条の4・シャドーAI統制】ブラウザ完結12連ゲート『AIGOVGATE』実務設計"
emoji: "⚖️"
type: "tech"
topics: ["ai", "governance", "copyright", "japan", "compliance"]
published: false
---

企業内で生成AIやコーディングエージェントの業務導入が進む一方、情シス・法務部門では次のような現場リスクが急増しています。

1. **シャドーAI・二次学習リスク**: 無認可の無料個人アカウントへ顧客データや契約書を入力し、モデル学習に利用される問題。
2. **著作権法第30条の4但書リスク**: 特定作家・既存キャラクターの「作風」を狙い撃ちしたプロンプト（享受目的併存）による侵害トラブル。
3. **OSSモデルライセンス誤用**: `CC-BY-NC` や `Research Only` 限定モデルの商用サービス組み込み、および採用・与信など高リスクAIにおける人間確認（HITL）欠如。

これらを社内未公開規程のまま外部SaaSへ送らずブラウザ完結・通信ゼロで診断し、**『社内生成AI利活用ガイドライン（Markdown）』** まで自動生成する第12のゲート **[AIGOVGATE（統治門）](https://pecopeko.github.io/aigovgate/)** を公開しました。

## AIGOVGATE（統治門）の主な検査軸

- **EU AI Act リスク階層 × モデルライセンス自動判定**: 「職場感情監視・社会スコア（禁止）」「採用/与信/医療トリアージ（高リスク）」「顧客チャットボット/合成メディア（透明性表示）」を判別し、非商用ライセンスの本番投入も警告します。
- **第1.1版 8統制領域チェック＋反証検知**: 学習オプトアウト（ZDR）・著作権法第30条の4・個人情報/営業秘密事前マスキング・HITL人間確認・プロンプトインジェクション防御・AI生成明記・監査ログ・CAIO窓口を自動スコア化し、「オプトアウト不要」「人間確認不要」などのNG記述も明示検出します。

## 公開中の全12連ゲート（GitHub Pages・無料・登録不要）

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
12. **[AIGOVGATE（統治門）](https://pecopeko.github.io/aigovgate/)** — AI事業者ガイドライン第1.1版・著作権法30条の4・シャドーAI＆社内ガイドライン生成

一括CSV処理向けの Python 製 **12ゲート統合オフラインCLI (`batch_compliance_suite_cli.py`)** およびシャドーAI台帳監査ツール（`ai_tool_inventory_risk_register.py`）と連携してそのまま社内監査へ組み込めます。
