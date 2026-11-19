---
title: "【改正障害者差別解消法・JIS X 8341-3 AA対応】全10連・完全ブラウザ完結日本B2Bコンプライアンスゲート群『JISGATE』まで揃えた設計まとめ"
emoji: "♿"
type: "tech"
topics: ["accessibility", "wcag", "javascript", "japan", "compliance"]
published: true
---

改正障害者差別解消法により民間事業者にも「合理的配慮」が法的義務化され、あわせて自治体・EC・SaaS企業での **JIS X 8341-3:2016（WCAG 2.1 Level AA）** 準拠確認と『ウェブアクセシビリティ方針』の公表ニーズが急増しています。  
しかし、公開前の未発売LP・会員画面HTML・社内システムのソースコードを外部SaaSクローラへ送信することは情報管理上避けたいケースが多くあります。

そこで **第10のゲート [JISGATE（基準門）](https://pecopeko.github.io/jisgate/)** を追加し、**完全クライアントサイド実行・通信ゼロの日本B2Bコンプライアンス10連ゲート** を完成させました。

## JISGATE（基準門）の検査エンジン

1. **JIS X 8341-3:2016 / WCAG 2.1 必須構文チェック**: `<html lang>`・`<title>`・`viewport`拡大制限禁止（`user-scalable=no` 等）・`<img> alt`・空`<button>`/`<a>`・曖昧リンク（「こちら」等）・ラベル未結合`<input>`・正の`tabindex`・見出し階層飛び級を一括検出。
2. **W3C sRGB 相対輝度・コントラスト比検証**: テキスト色と背景色 HEX から `4.5:1`（AA通常文字）／`7.0:1`（AAA）をブラウザ内で即時検算。
3. **『ウェブアクセシビリティ方針』自動Markdown生成**: 検出結果をもとに WAIC 指針に沿った公表方針文と相談窓口ドラフトをワンクリック生成。

## 公開中の全10連ゲート（GitHub Pages・無料・登録不要）

1. [KUROGATE（黒門）](https://pecopeko.github.io/kurogate/) — Mod-9/11＋可逆PIIマスキング＋Prompt Injection遮断
2. [YAKKIGATE（薬影）](https://pecopeko.github.io/yakkigate/) — 薬機法・景表法（No.1/二重価格）・ステマ規制監査
3. [ZENGINGATE（全銀門）](https://pecopeko.github.io/zengingate/) — 適格請求書T番号＋全銀半角カナ補正＋120バイト総合振込生成
4. [KEIYAKUGATE（契約門）](https://pecopeko.github.io/keiyakugate/) — 取適法60日サイト＋フリーランス新法＋印紙税節税試算
5. [TOKUSHOGATE（特商門）](https://pecopeko.github.io/tokushogate/) — 特商法第11条・第12条の6確認画面・悪質解約NG検査
6. [DENCHOGATE（電帳門）](https://pecopeko.github.io/denchogate/) — 電帳法第7条ファイル名正規化＋索引簿CSV＋事務処理規程
7. [KOJINGATE（個情門）](https://pecopeko.github.io/kojingate/) — 改正APPI第17〜39条＋外部送信規律第27条の12公表テーブル生成
8. [GENSENGATE（源泉門）](https://pecopeko.github.io/gensengate/) — 源泉徴収10.21%/20.42%・グロスアップ・2026年10月インボイス50%＆支払調書
9. [ROUMUGATE（労務門）](https://pecopeko.github.io/roumugate/) — 2024年4月労働条件通知書・固定残業3要件・月60h超1.5倍＆36協定
10. **[JISGATE（基準門）](https://pecopeko.github.io/jisgate/)** — 改正障害者差別解消法・JIS X 8341-3:2016 AA監査＆アクセシビリティ方針生成

社内のCSV／HTMLを一括監査したいチーム向けには Python 製 **10ゲート統合CLI (`batch_compliance_suite_cli.py`)** とフォルダ一括ツール一式も公開しています。
