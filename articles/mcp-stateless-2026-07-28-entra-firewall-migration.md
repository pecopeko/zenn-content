---
title: "Stateless MCP 2026-07-28移行とEntra MCP Firewall10項目監査"
emoji: "🛡️"
type: "tech"
topics: ["mcp", "security", "azure", "cloudflare", "python"]
published: false
---

Linux Foundation 傘下の **AAIF（Agentic AI Foundation）** による **Stateless MCP `2026-07-28` 仕様** と、2026年11月の **Microsoft Ignite** で発表された **Enterprise MCP Firewall（Entra Agent ID / Azure API Management / Cloudflare Workers / Envoy L7 連携）** により、社内・本番環境での Model Context Protocol（MCP）運用アーキテクチャは大きな転換点を迎えています。

最大の変更点は、初期仕様（`2024-11-05`〜`2025-03-26`）で広く使われてきた **ステートフルな `/sse` トランスポートおよび `Mcp-Session-Id` によるスティッキーセッションが12ヶ月間の非推奨期間（サンセット期限：`2027-07-28`）に入ったこと** です。

本記事では、なぜステートフル `/sse` がエッジファイアウォールと相性が悪かったのかをプロトコル構造から整理した上で、`mcp.json` / `.cursor/mcp.json` / `claude_desktop_config.json` / ゲートウェイ設定を移行・監査するための **決定論的10項目チェックリスト（`MCP-FW-01`〜`MCP-FW-10`）** と、ブラウザ内完結（外部送信ゼロ）のOSS監査・ポリシー生成ツール **MCPFIREWALL** の実装を解説します。

---

## 1. なぜ Stateful `/sse` は非推奨になり、`Mcp-Method` / `Mcp-Name` が導入されたのか

従来の HTTP+SSE 型 MCP サーバーには、エンタープライズ運用において3つの構造的なボトルネックがありました。

1. **L7 ロードバランサでのスティッキーセッション強制とスケール阻害**
   `GET /sse` で張りっぱなしのストリームを維持し、後続の `POST /messages` を同一インスタンスへルーティングするため、`Mcp-Session-Id` によるセッションアフィニティが必須でした。サーバーレス（Cloudflare Workers / AWS Lambda / Cloud Run ゼロスケール）ではコールドスタートやレプリカ切り替えのたびにセッション切断が発生します。
2. **`initialize` ハンドシェイクへの状態ロックイン**
   クライアントとサーバーが接続時に `initialize` → `notifications/initialized` の順序付き状態を共有する前提だったため、リクエスト単位での水平分散やエッジキャッシュが効きませんでした。`2026-07-28` 仕様では冪等かつキャッシュ可能な **`server/discover`** による機能発見（Capability Discovery）へ移行します。
3. **L7 エッジファイアウォールでの JSON-RPC ボディ解析コスト**
   従来の MCP 通信では HTTP パスが単一の `/mcp` や `/messages` に集約されており、「どのツール（`tools/call` の `name`）が呼ばれたか」を判定するには、エッジプロキシ側でリクエストボディ全体をバッファリングして JSON パースする必要がありました。

### `2026-07-28` Stateless HTTP POST コアとエッジルーティングヘッダ

`2026-07-28` 仕様および Ignite 2026 の Enterprise MCP Firewall アーキテクチャでは、リクエストを単発の **Stateless HTTP POST** に統一し、HTTP レイヤに次の2つの明示的ヘッダを要求します。

- `Mcp-Method`: JSON-RPC メソッド名（例: `server/discover`, `tools/list`, `tools/call`, `tasks/get`）
- `Mcp-Name`: 対象ツール・リソース・プロンプト識別子（例: `ledger__list_entries`）

これにより Microsoft Entra MCP Firewall や Cloudflare Workers、Envoy `ext_authz` フィルタは **HTTP ヘッダだけでミリ秒未満の Allowlist 認可・レート制限** を行えます。同時に、ヘッダと JSON-RPC ボディの不一致を突く **Header Smuggling 攻撃**（ヘッダには許可済みの `Mcp-Name: safe_read` を書き、JSON ボディの `params.name` に `drop_table` を入れる手口）を防ぐため、ゲートウェイ側で **`rejectBodyHeaderMismatch: true`** の強制が必須となります。

---

## 2. 本番移行で落とせない10の監査ゲート（`MCP-FW-01`〜`MCP-FW-10`）

開発者の手元の `.cursor/mcp.json` や社内ゲートウェイ定義を本番ファイアウォール配下へ移行する際、トランスポート変更だけでは不十分です。実務で事故が起きやすいポイントを10個の決定論的ゲートとして定義しました。

| Gate ID | 監査ゲート | 主な検知対象・リスク |
| --- | --- | --- |
| `MCP-FW-01` | Legacy Stateful `/sse` 非推奨検査 | `transport: "sse"`, `/sse` エンドポイント, `2024-11-05` 等の旧版指定, `stickySession: true`, 平文 `http://` |
| `MCP-FW-02` | Edge Header Routing 検査 | `Mcp-Method` / `Mcp-Name` ヘッダ強制の欠落、`rejectBodyHeaderMismatch: false` による Smuggling 許容 |
| `MCP-FW-03` | `server/discover` 移行検査 | `Mcp-Session-Id` ヘッダ残存、`handshake: "initialize"` によるステートフル固定 |
| `MCP-FW-04` | 平文認証情報・静的 Bearer 漏洩検査 | `env` / `headers` / CLI 引数 / URL クエリ内の生 API キーや固定 `Authorization: Bearer` トークン |
| `MCP-FW-05` | OAuth 2.1 + PKCE / RFC 8693 / RFC 9449 DPoP | `S256` PKCE 欠落、**RFC 8693 Token Exchange（OBO 権限絞り込み）** 欠落、**RFC 9449 DPoP** 送信者拘束の未設定 |
| `MCP-FW-06` | 未ピン留めコマンド・サプライチェーン検査 | バージョン未固定の `npx -y pkg` / `uvx pkg@latest`、`:latest` コンテナ、`sh -c` や `curl \| sh` パイプ |
| `MCP-FW-07` | MRTR（逆方向リクエスト）情報流出ガード | サーバー発の `sampling/createMessage`・`elicitation/create` に対する `maxTokens` 上限・ホスト許可リスト・HITL 承認の欠落 |
| `MCP-FW-08` | 非同期 `tasks/*` タイムアウト・TTL・中断検査 | ステートレス環境でのゾンビジョブ化を防ぐ `timeoutMs`、`ttlSeconds`、`supportsCancel: true` の欠落 |
| `MCP-FW-09` | Tool Poisoning・ツール名シャドーイング検査 | `description` 内の `<important>` 等の隠し指示・ゼロ幅 Unicode 文字・複数サーバー間の同名ツール衝突（`read_file` vs `read-file`） |
| `MCP-FW-10` | Default-Deny Firewall Policy 出力・検証 | デフォルト拒否（`defaultAction: "deny"`）のサーバー／メソッド／ツール Allowlist 定義の有無と自動生成 |

### 特に見落とされやすい3つの深層リスク

#### ① Confused Deputy（混乱した代理人）と RFC 8693 + RFC 9449 DPoP（Gate 5）
MCP クライアントが取得した広範な社内 SSO トークンをそのまま下流 MCP サーバーへパススルーすると、悪意ある（または侵害された）MCP サーバーがそのトークンを別 API へリプレイできてしまいます。
Entra Agent ID 等の最新ベストプラクティスでは、**RFC 8693 Token Exchange（On-Behalf-Of）** で対象 MCP サーバー専用の `audience`（RFC 8707 `resourceIndicator`）にスコープを絞り込み、さらに **RFC 9449 DPoP（Demonstrating Proof-of-Possession）** で公開鍵バインディングを行うことで、万一トークンが漏洩しても別ホストからリプレイ不能にします。

#### ② MRTR（`sampling/createMessage` & `elicitation/create`）によるコンテキスト抜き取り（Gate 7）
MCP は「クライアント→サーバー」だけでなく、サーバーからクライアントの LLM を呼び出す `sampling/createMessage` や、ユーザー入力を要求する `elicitation/create` を備えています。これを無制限に許可すると、外部 MCP サーバーがホスト側の会話履歴を要約させて外部へ持ち出したり、フィッシング対話を挿入したりできます。原則 `deny`、許可する場合も `maxTokens <= 4096`・許可サーバー限定・Human-in-the-Loop（HITL）承認の3点セットが不可欠です。

#### ③ Cross-Server Tool Shadowing（Gate 9）
社内サーバー `workspace-docs` の `read-file` と、後から追加した外部サーバーの `read_file` のように、正規化名（ハイフン／アンダースコア統一）が衝突すると、LLM のツール選択が乗っ取られます。MCP Firewall レイヤでは `ledger__list_entries` のように名前空間プレフィックスを強制し、汚染・衝突ツールを `quarantinedMcpNames` として自動隔離する設計が安全です。

---

## 3. Before / After：レガシー `/sse` 設定から `2026-07-28` + Firewall Policy への移行例

### 移行前（複数の重大リスクを抱えた従来の `mcp.json`）

```json
{
  "protocolVersion": "2024-11-05",
  "mcpServers": {
    "crm-billing-sse": {
      "transport": "sse",
      "url": "http://mcp-legacy.kestrel-finops.io/sse?api_key=<REDACTED_QUERY_KEY>",
      "handshake": "initialize",
      "stickySession": true,
      "headers": {
        "Authorization": "Bearer <REDACTED_STATIC_BEARER>",
        "Mcp-Session-Id": "sess-sticky-00192"
      },
      "sampling": { "enabled": true },
      "tasks": { "enabled": true, "timeoutMs": 900000 }
    }
  }
}
```

### 移行後（`2026-07-28` Stateless HTTP POST ＋ Default-Deny Firewall 準拠）

```json
{
  "protocolVersion": "2026-07-28",
  "gateway": {
    "protocolVersion": "2026-07-28",
    "discovery": "server/discover",
    "enforceHeaders": ["Mcp-Method", "Mcp-Name"],
    "rejectBodyHeaderMismatch": true
  },
  "mrtr": {
    "enabled": false,
    "samplingCreateMessage": "deny",
    "elicitationCreate": "require_hitl_approval",
    "maxTokens": 1024,
    "allowedServers": ["ledger-stateless"],
    "requireApproval": true
  },
  "tasks": {
    "timeoutMs": 30000,
    "ttlSeconds": 300,
    "maxConcurrent": 5,
    "supportsCancel": true
  },
  "firewall": {
    "defaultAction": "deny",
    "serverAllowlist": [
      {
        "serverName": "ledger-stateless",
        "upstreamEndpoint": "https://mcp.northstar-ledger.io/ledger/mcp",
        "allowedMcpMethods": ["server/discover", "tools/list", "tools/call"],
        "allowedMcpNames": ["ledger__list_entries", "ledger__get_voucher"]
      }
    ]
  },
  "mcpServers": {
    "ledger-stateless": {
      "protocolVersion": "2026-07-28",
      "transport": "stateless-http",
      "url": "https://mcp.northstar-ledger.io/ledger/mcp",
      "discovery": "server/discover",
      "edgeRouting": {
        "requireMcpHeaders": true,
        "requiredHeaders": ["Mcp-Method", "Mcp-Name"],
        "rejectBodyHeaderMismatch": true
      },
      "auth": {
        "type": "oauth2.1",
        "codeChallengeMethod": "S256",
        "tokenExchange": true,
        "dpop": true,
        "resourceIndicator": "https://mcp.northstar-ledger.io/ledger/mcp"
      },
      "env": {
        "LEDGER_CLIENT_ID": "${LEDGER_OAUTH_CLIENT_ID}"
      }
    }
  }
}
```

---

## 4. ブラウザ完結（外部通信ゼロ）の10ゲート監査＆ポリシー出力ツール『MCPFIREWALL』

社内の `mcp.json` には内部エンドポイント URL や環境変数名が含まれるため、外部の Web 診断サービスへそのまま貼り付けるのはセキュリティ上望ましくありません。

そこで、**100% ブラウザローカルの JavaScript エンジンのみで動作（外部通信ゼロ・テレメトリゼロ）** し、上記10ゲートの採点と同時に **Enterprise MCP Firewall JSON ポリシー** および **`2026-07-28` マイグレーションパッチ** をワンクリック出力できる OSS ツール **MCPFIREWALL (`mcp-firewall-v2-auditor`)** を公開しました。

- **ライブ監査 Web アプリ（登録不要・ブラウザローカル完結）**：
https://pecopeko.github.io/mcp-firewall-v2-auditor/

- **GitHub リポジトリ（Python 3.11 ゼロ依存 CLI 同梱・Apache-2.0）**：
https://github.com/pecopeko/mcp-firewall-v2-auditor

CI/CD パイプライン（GitHub Actions 等）に組み込む場合は、同梱の Python 3.11 標準ライブラリのみの CLI で次のようにゲート検査とポリシー生成を自動化できます。

```bash
python3 -m mcp_firewall_v2.cli .cursor/mcp.json \
  --export-policy ./mcp-enterprise-firewall-policy.json \
  --export-migrated ./mcp.stateless-v2.json \
  --fail-on fail
```

### 静的マニフェスト監査の限界と実運用での組み合わせ

本ツールのような静的構成監査は「設定ファイル段階での `/sse` 残存・平文キー混入・未ピン留め `npx -y`・既知プロンプトインジェクション文字列」をミリ秒で阻止できる一方、次の点はランタイム側の責務となります。

1. **DPoP 署名（`DPoP` HTTP ヘッダの JWK 検証）や RFC 8693 トークン交換の実装正当性** は、実際の認可サーバー・API ゲートウェイ側での統合テストが必要です。
2. **動的に書き換わるリモート MCP サーバーの `tools/list` 応答**（ラグプル型の後から混入する Tool Poisoning）を防ぐには、出力された `mcp-enterprise-firewall-policy.json` の `allowedMcpNames` ハッシュ／許可リストをエッジゲートウェイ側で強制してください。

`2027-07-28` の `/sse` サンセット直前に慌てないためにも、まずは手元の `mcp.json` を棚卸しし、Stateless HTTP POST ＋ Default-Deny ファイアウォール構成への段階的移行を進めてみてください。

---

### 📘 さらに深く実装したい方へ（体系解説書 ＆ オフライン実務キット）

ブラウザローカル完結の AI セキュリティガード（可逆 PII Vault・チェックデジット検証・Prompt Injection 遮断）から、金融時系列機械学習（AFML）の検証設計までを体系化した実践書と、社内環境でそのまま使える完全オフライン実務キットも公開しています：

https://zenn.dev/pecopeko/books/afml-quant-and-local-ai-guard

- **オフライン実務コンプライアンス＆AI社外秘ガード Pro キット（BOOTH）**：
https://pecopeko1011.booth.pm/items/9179705
