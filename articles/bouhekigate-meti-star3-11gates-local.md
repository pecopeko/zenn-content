---
title: "【経産省サプライチェーン評価★3・★4 / PPC漏えい報告 / DMARC対応】11連ゲート『BOUHEKIGATE』設計"
emoji: "🛡️"
type: "tech"
topics: ["security", "dmarc", "japan", "compliance", "javascript"]
published: false
---

中堅・中小企業やSaaSベンダーにとって、発注元企業から毎年のように届く「情報セキュリティ対策チェックシート」への回答と、経済産業省の **サプライチェーン強化に向けたセキュリティ対策評価制度（★3・★4）**、**個人情報保護委員会（PPC）への漏えい速報（概ね3〜5日）/ 確報（30日・60日）SLA**、および **BEC対策としての SPF / DMARC 強制（`p=reject`・SPF 10ルックアップ制限）** の自己点検は大きな負担になっています。

そこで **第11のゲート [BOUHEKIGATE（防壁門）](https://pecopeko.github.io/bouhekigate/)** を追加し、社内規程やチェックシート記述を外部サーバーへ1バイトも送らずにブラウザ上で完結診断できる **11連日本B2Bコンプライアンスゲート** を公開しました。

## BOUHEKIGATE（防壁門）の診断エンジン

1. **経産省（METI）★3 / ★4 想定8領域スコア＆反証パターン検出**: MFA・EDR/CVEパッチSLA・イミュータブル3-2-1復旧訓練・PPC速報/確報SLA・不正競争防止法 営業秘密3要件・再委託統制・TLS1.3/AES-256暗号化ログを自動採点。「MFA未導入」「USB制限なし」「報告期限なし」等の否定的表現も明示的に減点します。
2. **RFC 7208 SPF ルックアップ上限（10回）＆ DMARC 構文検証**: `+all` の致命的不備、11回超の `include:` / `mx` による PermError、`p=none` 監視止まりをローカルで即座に検算します。
3. **『取引先提出用 情報セキュリティ対策状況回答書』Markdown生成**: 診断結果・CSIRT窓口・適合/是正計画表・改正APPI第26条SLA声明をワンクリックで取引先提出用ドキュメント化します。

## 公開中の全11連ゲート（GitHub Pages・無料・登録不要）

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
11. **[BOUHEKIGATE（防壁門）](https://pecopeko.github.io/bouhekigate/)** — 経産省サプライチェーン評価★3/★4・PPC漏えいSLA・SPF/DMARC＆取引先回答書生成

企業内CSVを一括処理する Python 製 **11ゲート統合オフラインCLI (`batch_compliance_suite_cli.py`)** およびチェックシートCSV回答ドラフトエンジン（`security_checksheet_csv_responder.py`）も同梱しています。
