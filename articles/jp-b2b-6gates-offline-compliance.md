---
title: "【完全ブラウザ完結6連ゲート】契約書(取適法)・特商法・電帳法・全銀・薬機法・PII遮断をサーバー送信ゼロで作った"
emoji: "⚖️"
type: "tech"
topics: ["javascript", "japan", "compliance", "saas", "privacy"]
published: false
---

日本のバックオフィス・法務・EC運営では「秘密保持・個人情報・未公開契約書を海外LLMや外部SaaSへアップロードしたくない」という現場制約が強く働きます。そこで **外部通信ゼロ（100%ブラウザローカルES Modules）** で即座に動く6つの静的コンプライアンスゲートを公開・拡張しました。

## 公開中の6ゲート（すべて無料・登録不要・GitHub Pages稼働）

1. **[KUROGATE（黒門）](https://pecopeko.github.io/kurogate/)**: 国税庁Mod-9 法人/適格請求書T番号・マイナンバーMod-11検証＋リバーシブルPIIマスク＋プロンプトインジェクション遮断（Chrome拡張同梱）
2. **[YAKKIGATE（薬影）](https://pecopeko.github.io/yakkigate/)**: 薬機法・景品表示法（No.1表示・二重価格）・ステマ規制スパン検査＋適法リライト案生成（Chrome拡張同梱）
3. **[ZENGINGATE（全銀門）](https://pecopeko.github.io/zengingate/)**: 適格請求書T番号Mod-9検算＋小書きカナ・濁点1バイト分解対応の全銀半角カナ補正＋120バイト総合振込データ生成・検算
4. **[KEIYAKUGATE（契約門）](https://pecopeko.github.io/keiyakugate/)**: 取適法（60日支払サイト・協議なき価格据置・買いたたき）＋フリーランス新法3要素＋著作権27/28条・無制限賠償チェック＋収入印紙節税計算（電子契約で最大60万円節税試算）
5. **[TOKUSHOGATE（特商門）](https://pecopeko.github.io/tokushogate/)**: 特定商取引法第11条必須開示項目・第12条の6最終確認画面（初回格安＋2回目以降料金/総額明示）・電話限定解約NG検査＋審査通過向けMarkdownひな形生成
6. **[DENCHOGATE（電帳門）](https://pecopeko.github.io/denchogate/)**: 改正電子帳簿保存法第7条電子取引の3要件（和暦`令和`/`R`・全角数字対応）一括ファイル名正規化 `YYYYMMDD_取引先名_金額.pdf`＋国税庁モデル準拠 検索用索引簿CSV（UTF-8 BOM）＋訂正削除防止 事務処理規程ジェネレータ

## 技術的なこだわり（なぜ決定論的ルール＋ローカル検算が効くのか）

- **T番号・和暦の誤爆ゼロ**: 正規表現だけの判定と異なり、国税庁法人番号・適格請求書番号のウェイト `[1, 2]` Mod-9チェックデジットおよび閏年・元号境界（`令和元年`→`2019`）を厳密に検算します。
- **一括CSV監査 CLI & オフライン配布キット**: 全CSV監査・SHA-256改ざん検知ハッシュ付き監査報告書を出せる Python CLI も GitHub Release（[Japan B2B Compliance Pro Suite v1.0.0](https://github.com/pecopeko/horizon-money-studio/releases/tag/v1.0.0-pro-kit)）に同梱しています。
- 関連設計・AFML日本株検証手法は有料Zenn Book（全6章・¥2,400）でも詳しく解説予定です。

---

### 📦 実務ですぐ使える完全オフライン同梱キット ＆ 体系解説書

本記事の「Shift-JIS 厳密120バイト全銀データ出力」「電帳法タイムスタンプ名PDF一括リネームZIP」「支払調書・源泉試算」「ローカルPII可逆ガード」の5ツールを、社内ネットワークから外部通信ゼロでそのまま使えるスタンドアロン同梱版としてまとめています：

- **【完全オフライン・Shift-JIS実出力】経理・バックオフィス自動化5点セット v2.0（BOOTH）**：
https://pecopeko1011.booth.pm/items/9179705

また、1-to-1正規化・チェックデジット検証からローカルLLMセキュリティ・クオンツ検証基盤までを全8章で詳解したZennブックはこちらです：

https://zenn.dev/pecopeko/books/afml-quant-and-local-ai-guard
