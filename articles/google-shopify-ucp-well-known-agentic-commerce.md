---
title: "Google×Shopify UCP実装解説：/.well-known/ucp検証とAIエージェント決済対応"
emoji: "🛒"
type: "tech"
topics: ["shopify", "stripe", "mcp", "ai", "ecommerce"]
published: false
---

Google（Gemini / AI Mode in Search）、Shopify、Stripe らが共同策定を進める **Universal Commerce Protocol（UCP / `ucp.dev`）** により、AIエージェントがECストアの商品検索からカート構築、配送計算、ネイティブ決済までを標準プロトコルで自律実行する **Agentic Commerce（エージェンティック・コマース）** の実装フェーズが本格化しています。

一方で、実際にストア側で公開ディスカバリファイル **`https://<domain>/.well-known/ucp`** や A2A エージェントカード（**`/.well-known/agent.json`**）を構成しようとすると、以下のような「本番事故・仕様違反トラップ」が頻発します。

1. **`ucp.version` の日付形式違反とスキーマ世代ズレ**：`"1.0"` のようなセマンティックバージョンを書いてしまい、AIエージェント側のパーサに弾かれる（UCP仕様は `"2026-01-23"` 形式の `YYYY-MM-DD` リリース日指定が必須）。
2. **`extends` による Capability 継承グラフ（DAG）の循環参照・親欠落**：`dev.ucp.shopping.fulfillment` や `discount` が未宣言の親 Capability を参照したり、拡張同士で循環ループを作って Capability Negotiation が破綻する。
3. **`payment_handlers` への秘密鍵混入や非UUID識別子**：公開ファイルである `/.well-known/ucp` 内に誤って `shpat_...` や `sk_live_...` を書き込む事故、あるいはハンドラ `id` が RFC 4122 UUID 形式を満たさない問題。
4. **`Schema.org Offer` と UCP Checkout の通貨・在庫ドリフト（価格ハルシネーション）**：HTMLのJSON-LDでは `JPY` を返しているのに、UCP Checkout 側が `USD` のみを受け付けるため決済直前でマンデート検証が失敗する。
5. **AP2（Agent Payments Protocol）マンデート・冪等キー（`Idempotency-Key`）欠落による二重課金リスク**：自律エージェントのリトライ時に `complete_checkout` が重複実行される。

本記事では、`ucp.dev`（`2026-01-23` 仕様）および Stateless MCP（`2026-07-28`）に基づき、これら **10個の監査ゲートをPython 3.11 / JavaScriptで決定論的に検査する実装** を解説し、ブラウザ内完結（外部通信ゼロ `connect-src 'none'`）の検証＆1クリック生成ワークベンチ **『UCPGATE』** を公開します。

- **ライブ検証 ＆ 1-Click `/.well-known/ucp` + Cloudflare Worker 生成ツール（登録不要・無料）**：
https://pecopeko.github.io/ucp-agentic-commerce-auditor/
- **GitHub リポジトリ（Python CLI + JS Engine / Apache-2.0）**：
https://github.com/pecopeko/ucp-agentic-commerce-auditor

---

## 1. `/.well-known/ucp`（`2026-01-23`）の4層アーキテクチャ

UCP のディスカバリプロファイルは、単一の巨大なAPI仕様を押し付けるのではなく、**4つの疎結合ブロック**によって「エージェントとマーチャントの共通部分（Intersection）」を動的にネゴシエーションする設計になっています。

```json
{
  "ucp": {
    "version": "2026-01-23",
    "services": {
      "dev.ucp.shopping": [
        {
          "version": "2026-01-23",
          "spec": "https://ucp.dev/specification/overview",
          "transport": "rest",
          "endpoint": "https://nordic-roast.shop/api/ucp/2026-01-23",
          "schema": "https://ucp.dev/2026-01-23/services/shopping/openapi.json"
        },
        {
          "version": "2026-01-23",
          "spec": "https://ucp.dev/specification/mcp",
          "transport": "mcp",
          "endpoint": "https://nordic-roast.shop/api/ucp/mcp",
          "schema": "https://ucp.dev/2026-01-23/services/shopping/openrpc.json",
          "config": {
            "mcp_protocol_version": "2026-07-28",
            "stateless_streamable_http": true,
            "required_meta": ["ucp-agent", "idempotency-key"]
          }
        }
      ]
    },
    "capabilities": {
      "dev.ucp.shopping.checkout": [
        {
          "version": "2026-01-23",
          "spec": "https://ucp.dev/specification/checkout",
          "schema": "https://ucp.dev/2026-01-23/schemas/shopping/checkout.json",
          "config": {
            "supported_currencies": ["USD"],
            "seller_of_record": "nordic-roast.shop",
            "idempotency_key_required": true,
            "ap2_mandate_required": true
          }
        }
      ],
      "dev.ucp.shopping.fulfillment": [
        {
          "version": "2026-01-23",
          "spec": "https://ucp.dev/specification/fulfillment",
          "schema": "https://ucp.dev/2026-01-23/schemas/shopping/fulfillment.json",
          "extends": "dev.ucp.shopping.checkout"
        }
      ],
      "dev.ucp.shopping.ap2_mandate": [
        {
          "version": "2026-01-23",
          "spec": "https://ucp.dev/specification/ap2-mandate",
          "schema": "https://ucp.dev/2026-01-23/schemas/shopping/ap2_mandate.json",
          "extends": "dev.ucp.shopping.checkout",
          "config": {
            "mandate_formats": ["CheckoutMandate", "SD-JWT-VC"],
            "require_user_signature": true
          }
        }
      ]
    },
    "payment_handlers": {
      "com.google.pay": [
        {
          "id": "b1a74957-46c0-5e72-867e-1731b7d4e2a0",
          "version": "2026-01-23",
          "spec": "https://pay.google.com/gp/p/ucp/2026-01-23/",
          "schema": "https://pay.google.com/gp/p/ucp/2026-01-23/schemas/config.json",
          "config_schema": "https://pay.google.com/gp/p/ucp/2026-01-23/schemas/config.json",
          "instrument_schemas": [
            "https://pay.google.com/gp/p/ucp/2026-01-23/schemas/card_payment_instrument.json"
          ]
        }
      ]
    }
  }
}
```

### 逆ドメイン名前空間（Reverse-Domain Grammar）の衝突防止ルール

UCP では拡張機能を中央レジストリの審査なしで安全に追加できるよう、Javaパッケージ名と同様の逆ドメイン構文を採用しています。

- **予約済み名前空間 `dev.ucp.*`**：公式仕様（`dev.ucp.shopping.checkout` / `fulfillment` / `discount` / `order` / `ap2_mandate` / `dev.ucp.identity_linking`）専用です。自社独自のポイント施策や定期購入拡張を `dev.ucp.shopping.custom_vip` のように宣言すると名前空間スクワッティングとなるため、必ず自社ドメインを反転させた `shop.nordicroast.shopping.loyalty` 形式で定義します。
- **HTTPS強制とSSRF遮断**：`services` や `payment_handlers` 内の `endpoint` / `schema` / `spec` はすべて公開TLS（`https://`）が必須です。ステージング設定のまま `http://` や `10.0.x.x` / `127.0.0.1` / `localhost` が残っていると、エージェント側のSSRFフィルタによってストアごとブロックされます。

---

## 2. `extends` による Capability DAG 検証と循環参照の検出

UCP の特徴的な仕組みが `capabilities` 内の `"extends"` フィールドです。`fulfillment`（配送オプション計算）や `discount`（クーポン適用）、`ap2_mandate`（暗号署名付き決済委任）は、単体で存在するのではなく `dev.ucp.shopping.checkout` を拡張（継承）する有向非巡回グラフ（DAG）を構成します。

CI/CD パイプラインで `/.well-known/ucp` を検証する際は、**①未宣言の親への参照（Dangling Parent）がないか**、**②3色DFSによる閉路（Cycle）がないか** をチェックします。

```python
def inspect_capability_dag(capabilities: dict) -> dict:
    nodes = list(capabilities.keys())
    node_set = set(nodes)
    parents_map = {k: [] for k in nodes}
    dangling_parents = []
    cycles = []

    for cap_key, descriptors in capabilities.items():
        for desc in descriptors:
            ext = desc.get("extends")
            if not ext:
                continue
            targets = [ext] if isinstance(ext, str) else list(ext)
            for parent in targets:
                if parent == cap_key:
                    cycles.append([cap_key, cap_key])
                elif parent not in node_set:
                    dangling_parents.append({"capability": cap_key, "missing_parent": parent})
                elif parent not in parents_map[cap_key]:
                    parents_map[cap_key].append(parent)

    # 3-color DFS (0=unvisited, 1=visiting, 2=visited)
    color = {k: 0 for k in nodes}
    stack = []
    topo_order = []

    def dfs(u: str) -> None:
        color[u] = 1
        stack.append(u)
        for p in parents_map[u]:
            if color[p] == 0:
                dfs(p)
            elif color[p] == 1:
                idx = stack.index(p)
                cycles.append(stack[idx:] + [p])
        stack.pop()
        color[u] = 2
        topo_order.append(u)

    for n in nodes:
        if color[n] == 0:
            dfs(n)

    return {
        "is_acyclic": len(cycles) == 0,
        "dangling_parents": dangling_parents,
        "cycles": cycles,
        "topological_order": topo_order if not cycles else [],
    }
```

---

## 3. AP2（Agent Payments Protocol）マンデートと MCP `idempotency-key`

AIエージェントが人間の代わりに決済を実行する際、最大のセキュリティ課題は **「ユーザーの意図しない金額・店舗での無断決済」** と **「ネットワークタイムアウト時の二重決済」** です。

UCP + AP2 スタックでは、以下の3層でこれを防ぎます。

1. **Merchant of Record（Seller of Record）の維持**：AIプラットフォーム側が勝手に再販業者になるのではなく、マーチャント自身が販売主体として残り、`com.google.pay`・`com.stripe.pay`・`shop.pay` などのトークン化ハンドラ経由で決済証憑のみを受け取ります。
2. **AP2 暗号マンデート（`dev.ucp.shopping.ap2_mandate`）**：チェックアウト確定時に、カート内容のハッシュを含む `CheckoutMandate` とユーザー同意を示す `SD-JWT-VC` 形式の `PaymentMandate` を検証します。
3. **REST / Stateless MCP（`2026-07-28`）双方での冪等性強制**：
   - REST トランスポートでは HTTP ヘッダ `Idempotency-Key` を要求。
   - MCP トランスポートでは旧来のステートフル `/sse` を廃止して Streamable HTTP（`/mcp`）へ移行した上で、`complete_checkout` / `cancel_checkout` 呼び出しの JSON-RPC パラメータ `meta["idempotency-key"]` を必須化します。

また、`payment_handlers` の `id` フィールドには、環境ごとにランダム生成してドリフトさせるのではなく、**RFC 4122 UUIDv5（`NAMESPACE_URL` + ドメイン + ハンドラ名）** を使うことで、Python バックエンドと Cloudflare Worker 間で完全に同一の決定論的 UUID を維持できます。

```python
import uuid

def deterministic_handler_uuid(domain: str, handler_key: str, version: str = "2026-01-23") -> str:
    seed_uri = f"https://{domain.lower()}/.well-known/ucp?version={version}#{handler_key.lower()}"
    return str(uuid.uuid5(uuid.NAMESPACE_URL, seed_uri))
```

---

## 4. 公開時の落とし穴：SPAの `text/html` ソフト404と CORS ヘッダ

Next.js や Shopify Hydrogen、SPA ルーティング配下で `/.well-known/ucp` を配信する際、最も多い運用ミスが **「HTTP 200 だが `Content-Type: text/html`（フォールバックHTML）が返っている」** ケースと、ブラウザ型AIエージェントからのプリフライトを弾いてしまう **CORS ヘッダ欠落** です。

Cloudflare Worker でエッジ配信する場合は、以下のように `application/json; charset=utf-8`、`Access-Control-Allow-Origin: *`、`Cache-Control`、`X-Content-Type-Options: nosniff` を明示します（**[UCPGATE の Builder タブ](https://pecopeko.github.io/ucp-agentic-commerce-auditor/)** から店舗ドメインを入力すると、そのままデプロイ可能な Worker コードと Nginx 設定を1クリック出力できます）。

---

## 5. 複数ストア・大規模ECカタログの一括監査と連携API

1店舗ずつの `/.well-known/ucp` 検証・生成はブラウザローカル版の **[UCPGATE](https://pecopeko.github.io/ucp-agentic-commerce-auditor/)** で完結しますが、EC支援会社・SEO/GEOコンサルティング・AIショッピング開発チームが **数十〜数万店舗の商品フィード整合性や技術スタックを一括抽出・監査** したい場合は、公開中の **Apify Store Pay-Per-Event（PPE）API** をそのままパイプラインに組み込めます。

- **Shopify 公開カタログ・バリアント価格・在庫フィード抽出 API（`$1 / 1,000 products`）**：
https://apify.com/pecopeko/shopify-public-catalog-feed-extractor
- **Webサイト技術スタック・サイトマップ・AI対応構造一括抽出 API（`$2 / 1,000 sites`）**：
https://apify.com/pecopeko/website-tech-stack-sitemap-extractor

また、自律型AIエージェントのローカル防御設計・検証パイプラインや、バックオフィス実務のゼロ依存オフライン自動化ツール群については、以下の実践ガイド・ツールキットでも詳しく解説・配布しています。

- **Zenn 実践書『AFML定量検証とローカルAIコンプライアンス防衛設計』（¥1,200）**：
https://zenn.dev/pecopeko/books/afml-quant-and-local-ai-guard
- **BOOTH『経理・バックオフィス自動化オフライン5点セット』（¥1,980）**：
https://pecopeko1011.booth.pm/items/9179705

AI検索・AIショッピングエージェント経由の流入と決済コンバージョンを取りこぼさないための技術チェックリストとして、ぜひ自社の `/.well-known/ucp` 設計に活用してみてください。
