---
title: "EU AI法50条(12/2期限)C2PAバイナリ解析と日本AI法安全監査"
emoji: "🔍"
type: "tech"
topics: ["ai", "c2pa", "python", "javascript", "security"]
published: false
---

**2026年12月2日** に迫る **EU AI Act（欧州AI規則）第50条（Article 50：透明性義務および機械可読な生成来歴マーキング）** の実務対応期限と、**2026年11月の日本・人工知能関連技術の研究開発及び活用の推進に関する法律（AI推進法）をめぐる安全確保・監査証跡・罰則規定を含む見直し議論** により、生成AIを組み込んだWebサービス・メディア配信パイプライン・自律エージェントの設計要件が急速に具体化しています。

特に開発現場で盲点になりやすいのが、**「生成時に C2PA（Content Credentials）署名を付与したのに、Cloudflare Polish や Next.js（`sharp`）の画像最適化・CDN配信の過程でバイナリチャンク（`caBX` / `APP11`）が剥がれ落ち、本番配信時点で第50条2項違反状態になっている」** というインフラ起因の欠落です。

本記事では、以下の3点をコードとバイナリ構造から体系的に解説します。

1. **PNG / JPEG / WebP / MP4 コンテナ内の ISO/IEC 19566-5（JUMBF）バイナリ構造と IPTC `digitalSourceType` 要件**
2. **外部ライブラリに依存しない Python 3.11 / ブラウザ `Uint8Array` による C2PA バイナリ検査実装**
3. **EU AI Act 第50条（1〜4項）＋ 日本AI推進法（監査ログ・緊急停止 Kill-Switch）を横断する10ゲート監査ツール『C2PAGATE』**

---

## 1. EU AI Act 第50条（`2026-12-02`）と日本AI推進法（2026年11月議論）の実務要点

### EU AI Act 第50条の4つの柱

EU AI Act 第50条は、ハイリスクAIに限らず、一般消費者・公開Web向けの幅広い生成AIシステムおよびデプロイヤー（導入事業者）に適用されます。

| 条項 | 対象システム・出力 | 技術的・運用的要件 |
| --- | --- | --- |
| **第50条1項** | チャットボット等の対話型AI | 最初の対話開始前（または同時）に、自然人へAIと対話している旨を明瞭かつアクセシブル（`aria-label` 等）に通知 |
| **第50条2項** | 合成画像・音声・動画・テキスト生成AI | 出力が人工的に生成・操作されたことを **機械可読（Machine-readable）かつ相互運用可能な形式** でマーキング |
| **第50条3項** | 感情認識・生体カテゴリ分類システム | 対象となる自然人への事前明示通知と GDPR / 個人情報保護法に基づく DPIA（影響評価）。※職場・教育機関での感情推測は第5条1項(f)で原則禁止 |
| **第50条4項** | ディープフェイクおよび公益事項に関するAI生成テキスト | 合成映像・音声の可視開示。公益ニュース等のテキストは可視開示、または **人間による編集審査＋編集責任主体（自然人・法人）の明記** |

第50条2項の「相互運用可能な機械可読マーキング」の中核実装として採用が進んでいるのが、**C2PA（Coalition for Content Provenance and Authenticity）v2.1 Content Credentials** と **IPTC NewsCodes `digitalSourceType` URI** の組み合わせです。

### 日本・AI推進法（2026年11月 安全性・実効性強化の方向性）

一方、日本国内でも2025年成立のAI推進法（第16条の調査・指導・助言権限等）を土台として、2026年秋の内閣府AI戦略会議・関係省庁議論において、重大インシデントや悪質事案に対する実効性確保（罰則・行政処分のあり方を含む法定安全義務の厳格化）が議論されています。エンジニアリング観点では次の2点が必須の防御線となります。

1. **改ざん不能（WORM / ハッシュ連鎖）なプロンプト・生成来歴監査ログ（180日以上保持）と24時間以内のインシデント報告体制（CAIO管掌）**
2. **自律型AIエージェントの異常検知サーキットブレーカーと、60秒以内に外部出力・公開を緊急遮断できる Kill-Switch 機構**

---

## 2. メディアコンテナ別の C2PA JUMBF バイナリ格納位置

C2PA のマニフェストストア（Manifest Store）は、**ISO/IEC 19566-5 JUMBF（JPEG Universal Metadata Box Format）** の `jumb` スーパーボックスとして各メディアファイル内部に格納されます。

| コンテナ形式 | マジックバイト | C2PA 格納チャンク／ボックス | バイナリ構造上の注意点 |
| --- | --- | --- | --- |
| **PNG** | `89 50 4E 47 0D 0A 1A 0A` | **`caBX`** チャンク | ビッグエンディアン長＋4バイト型名＋JUMBF本体＋IEEE 802.3 CRC-32 |
| **JPEG** | `FF D8` (SOI) | **`APP11` (`0xFFEB`)** セグメント | JPEG XT 仕様：`JP` (2B) + `En` インスタンス番号 (2B) + `Z` シーケンス番号 (4B) に続き `jumb`。64KB超では複数 `APP11` セグメントを `Z` 昇順で再結合 |
| **WebP** | `RIFF....WEBP` | **`C2PA`** RIFFチャンク | RIFFチャンク長自体は **リトルエンディアン** だが、内包される JUMBF ボックスは **ビッグエンディアン** |
| **MP4 / ISOBMFF** | `....ftyp` 等 | **`uuid`** ボックス | UUID `d8fec3d6-1b0e-483c-9297-5828877ec481` の内側に JUMBF `jumb` スーパーボックスを格納 |

さらに JUMBF `jumb` スーパーボックスの先頭には必ず `jumd`（Description Box）が置かれ、16バイトの Content-Type UUID と `toggles` ビットフラグに続いて UTF-8 ヌル終端文字列のラベルが格納されます。

- `c2pa`（ルート Manifest Store）
- `c2pa.assertions` → `c2pa.actions` / `c2pa.actions.v2`
- `c2pa.claim` / `c2pa.claim.v2`
- `c2pa.signature`（COSE_Sign1 暗号署名ペイロード）

### EU AI Act 第50条2項で必須となる IPTC `digitalSourceType` URI

単に C2PA 署名が付いているだけでは「カメラ撮影」なのか「AI生成」なのかを機械判別できません。`c2pa.actions.v2` アサーション内に、以下の正規 IPTC URI が記録されている必要があります。

- 純粋な学習済み生成AI出力：
  `http://cv.iptc.org/newscodes/digitalsourcetype/trainedAlgorithmicMedia`
- 実写・人間編集と生成AIの合成（AIローカライズ動画・インペイント等）：
  `http://cv.iptc.org/newscodes/digitalsourcetype/compositeWithTrainedAlgorithmicMedia`
- 非ニューラル・手続き的アルゴリズム生成：
  `http://cv.iptc.org/newscodes/digitalsourcetype/algorithmicMedia`

逆に、生成AI画像に対して実写カメラ用の `digitalCapture` を付与する設定ミスや偽装は、第50条2項における重大な不適合となります。

---

## 3. ゼロ依存 Python 3.11 による JUMBF ボックス走査の最小実装

CI/CD や軽量サーバーレス関数（AWS Lambda / Cloud Run / GitHub Actions）では、ネイティブ拡張ビルドを伴う重い画像ライブラリを入れずに「配信直前のバイナリに `caBX` / `APP11` / `C2PA` / `uuid` と IPTC AI URI が残っているか」をミリ秒で検査したい場面が多々あります。

標準ライブラリの `struct` と `zlib` だけで、PNG の `caBX` CRC検証および JUMBF ラベル抽出を行うコアロジックは次のようになります。

```python
import struct
import zlib

PNG_SIG = b"\x89PNG\r\n\x1a\n"

def extract_jumbf_labels(buf: bytes) -> list[str]:
    labels: list[str] = []
    pos = 0
    while pos + 8 <= len(buf):
        lbox = struct.unpack_from(">I", buf, pos)[0]
        tbox = buf[pos + 4 : pos + 8]
        box_size = (len(buf) - pos) if lbox == 0 else lbox
        if box_size < 8 or pos + box_size > len(buf):
            break
        payload = buf[pos + 8 : pos + box_size]
        if tbox == b"jumb" and len(payload) >= 25 and payload[4:8] == b"jumd":
            jumd_len = struct.unpack_from(">I", payload, 0)[0]
            toggles = payload[24]
            if toggles & 0x02:
                nul = payload.find(b"\x00", 25, jumd_len)
                raw = payload[25:nul] if nul != -1 else payload[25:jumd_len]
                labels.append(raw.decode("utf-8", errors="replace"))
            labels.extend(extract_jumbf_labels(payload))
        pos += box_size
    return labels

def inspect_png_cabx(png_bytes: bytes) -> dict:
    assert png_bytes.startswith(PNG_SIG), "Not a PNG stream"
    pos = 8
    while pos + 12 <= len(png_bytes):
        length = struct.unpack_from(">I", png_bytes, pos)[0]
        ctype = png_bytes[pos + 4 : pos + 8].decode("ascii", errors="replace")
        cdata = png_bytes[pos + 8 : pos + 8 + length]
        expected_crc = struct.unpack_from(">I", png_bytes, pos + 8 + length)[0]
        actual_crc = zlib.crc32(png_bytes[pos + 4 : pos + 8 + length]) & 0xFFFFFFFF
        if ctype == "caBX":
            return {
                "crc_valid": expected_crc == actual_crc,
                "labels": extract_jumbf_labels(cdata),
            }
        pos += 12 + length
    return {"crc_valid": None, "labels": []}
```

---

## 4. CDN メタデータ剥落への多層防御（HTTP `Link` ヘッダ ＋ Schema.org JSON-LD）

実運用で最も多い事故は、オリジンストレージ（S3 / R2）上の画像には `caBX` や `APP11` が入っているのに、エッジ画像変換プロキシや SNS クローラ経由でメタデータが削ぎ落とされるケースです。C2PA v2.1 および EU AI Act 第50条2項の相互運用設計では、**3層の冗長バインディング** が推奨されます。

1. **Layer 1（バイナリ埋め込み）**：画像・動画最適化パイプラインでメタデータを保持（`Cache-Control: no-transform`、またはリサイズ後に `c2patool --parent` で派生アセットとして再署名）。
2. **Layer 2（HTTP `Link` サイドカーヘッダ＋RFC 9530 `Content-Digest`）**：
   ```http
   Link: <https://cdn.example.eu/media/hero.png.c2pa>; rel="c2pa-manifest"; type="application/x-c2pa-manifest-store"
   Content-Digest: sha-256=:47DEQpj8HBSa+/TImW+5JCeuQeRkm5NMpJWZG3hSuFU=:
   Access-Control-Expose-Headers: Link, Content-Digest, X-AI-Provenance
   ```
3. **Layer 3（HTML / Schema.org JSON-LD 構造化データ）**：
   ```html
   <script type="application/ld+json">
   {
     "@context": "https://schema.org",
     "@type": "ImageObject",
     "contentUrl": "https://cdn.example.eu/media/hero.png",
     "digitalSourceType": "http://cv.iptc.org/newscodes/digitalsourcetype/trainedAlgorithmicMedia",
     "sdPublisher": { "@type": "Organization", "name": "Example Studio" }
   }
   </script>
   ```

---

## 5. ブラウザ内完結（アップロードゼロ）のバイナリ検査＆10ゲート監査ツール『C2PAGATE』

社内公開前のキービジュアル画像や動画、ガバナンス設定ファイルを外部サーバーへ送信せずにその場で検証できるよう、**100% ブラウザローカルの `Uint8Array` / `DataView` のみで動作する OSS 監査ワークベンチ『C2PAGATE (`c2pa-ai-act50-auditor`)』** を公開しました。

- **ライブ Web インスペクタ（登録不要・外部送信ゼロ・1クリック合成バイナリデモ搭載）**：
https://pecopeko.github.io/c2pa-ai-act50-auditor/

- **GitHub リポジトリ（Python 3.11 ゼロ依存 CLI & Node.js エンジン・Apache-2.0）**：
https://github.com/pecopeko/c2pa-ai-act50-auditor

### C2PAGATE が検査する10の決定論的ゲート（`C2PA-50-01`〜`C2PA-50-10`）

1. **`C2PA-50-01`**：PNG `caBX` / JPEG `APP11` / WebP `C2PA` / MP4 `uuid` の JUMBF ボックス＆COSE署名構造・CRC整合性検査
2. **`C2PA-50-02`**：IPTC `digitalSourceType` 正規URI検証と `digitalCapture` 偽装検知
3. **`C2PA-50-03`**：HTTP `Link: rel="c2pa-manifest"`・TLS強制・RFC 9530 `Content-Digest`・CORS公開ヘッダ検査
4. **`C2PA-50-04`**：HTML / Schema.org JSON-LD の機械可読 AI 来歴マークアップ検査
5. **`C2PA-50-05`**：EU AI Act 第50条1項 対話型AIの事前通知・ARIAアクセシビリティ・人間なりすまし防止検査
6. **`C2PA-50-06`**：第50条3項 感情認識・生体分類の事前通知＆DPIA検査（＋第5条1項(f) 職場・教育機関での感情監視ブロック）
7. **`C2PA-50-07`**：第50条4項 ディープフェイク開示＆公益テキストの人間編集審査・編集責任主体（`editorial_responsibility_holder`）検査
8. **`C2PA-50-08`**：Cloudflare Polish / `sharp` 等の CDN メタデータ剥落耐性ポリシー検査
9. **`C2PA-50-09`**：日本AI推進法（2026年11月議論対応）WORM監査ログ（180日以上）・24時間以内インシデント報告SLA・CAIO責任体制検査
10. **`C2PA-50-10`**：自律AIエージェントの緊急停止 Kill-Switch・自動サーキットブレーカー（60秒以内停止SLA）検証＆4種コンプライアンスバンドル出力

手元の PNG / JPEG / WebP / MP4 ファイルをドラッグ＆ドロップするだけでバイトオフセットマップと JUMBF ツリーが可視化され、そのまま本番用の **HTTP ヘッダ・JSON-LD・`c2patool` v2.1 マニフェストテンプレート・Cloudflare/Nginx 設定スニペット** をワンクリック出力できます。`2026-12-02` の EU AI Act 第50条適用開始前の最終チェックにぜひ活用してください。

---

### 📘 さらに深く実装したい方へ（体系解説書 ＆ オフライン実務キット）

ブラウザローカル完結の AI セキュリティガード（可逆 PII Vault・チェックデジット検証・Prompt Injection 遮断）から、金融時系列機械学習（AFML）の検証設計までを体系化した実践書と、社内環境でそのまま使える完全オフライン実務キットも公開しています：

https://zenn.dev/pecopeko/books/afml-quant-and-local-ai-guard

- **オフライン実務コンプライアンス＆AI社外秘ガード Pro キット（BOOTH）**：
https://pecopeko1011.booth.pm/items/9179705
