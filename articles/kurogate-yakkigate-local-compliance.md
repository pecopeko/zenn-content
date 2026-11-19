---
title: "【外部送信ゼロ】国税庁Mod-9・薬機法・全銀120バイトをブラウザ内だけで検証する無料ツール群を作った"
emoji: "🛡️"
type: "tech"
topics: ["javascript", "compliance", "chromeextension", "japan", "ai"]
published: false
---

業務文書を ChatGPT や Claude に貼る前の PII 伏せ字、LP原稿の薬機法・景表法・ステマ規制チェック、経理の T番号・全銀120バイトデータ検査を **「外部サーバーへ一切データを送らず、ブラウザ内だけで完結させる」** 無料Webツール＋Chrome拡張一式を公開しました。

- **KUROGATE（黒門）**: 国税庁Mod-9 法人/適格請求書T番号・マイナンバーMod-11 検証＋リバーシブル伏せ字＋プロンプトインジェクション検知
  https://pecopeko.github.io/kurogate/
- **YAKKIGATE（薬影）**: 薬機法・景表法（No.1/二重価格）・ステマ規制 事前監査＋Chrome拡張 v1.0.0
  https://pecopeko.github.io/yakkigate/
- **ZENGINGATE（全銀門）**: 適格請求書T番号検証＋全銀半角カナ自動補正＋120バイト総合振込生成＆検算
  https://pecopeko.github.io/zengingate/

技術的な背景（`T+13桁` のMod-9チェックデジットで無関係な受注番号誤爆を防ぐ設計、全銀半角カナの濁点1バイト計算、`DayBlockBagging` やブラウザONNXハイブリッド融合の詳細）は Zenn Book にもまとめています。
