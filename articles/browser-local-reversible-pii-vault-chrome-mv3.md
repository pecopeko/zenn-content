---
title: "ChatGPT/Claude送信前に機密をローカル可逆マスクするChrome MV3拡張と誤検知対策"
emoji: "🔒"
type: "tech"
topics: ["chromeextension", "security", "llm", "javascript", "python"]
published: true
---

## はじめに：業務文書をLLMに貼る前の「伏せ字」はなぜ現場で破綻するのか

ChatGPT・Claude・Gemini に請求書・問い合わせメール・障害ログを貼って要約やチェックを頼みたい場面は日常茶飯事です。しかし、社内の情報セキュリティ規定では「マイナンバー・適格請求書T番号・法人カード番号・顧客メールアドレスをそのまま外部LLMへ送るな」と定められています。

この対策として手元で簡易スクリプトやブラウザ拡張を作ろうとすると、ほぼ全員が次の **3つの壁** にぶつかります。

1. **不可逆な `[REDACTED]` 置換はLLMの推論を壊す**：文書内に2社のT番号が登場するとき、両方を `[REDACTED]` にすると「どちらの会社が請求元か」がLLMに分からなくなる
2. **素朴な正規表現（`\d{12}`, `\d{13}`, `\d{16}`）は誤検知だらけで使い物にならない**：JANコード・ISBN・UNIXミリ秒タイムスタンプ・配送伝票番号・分散トレースIDまで片っ端から黒塗りされ、さらに全角数字やUnicodeハイフン（`−` / `‐`）で肝心の機密を取りこぼす
3. **Chrome Manifest V3 で ChatGPT / Claude / Gemini の入力欄を書き換えても、そのまま送信すると元の未マスク文字列が飛ぶことがある**：React の Controlled Input や ProseMirror（`contenteditable`）の内部状態と DOM がズレる罠

本記事では、**外部API通信ゼロ・100%ブラウザローカル完結** で「可逆Vault（仮名化→AI回答の自動復元）」と「チェックデジット検証による誤検知削減」を実装する設計、実測ベンチマーク（50件日本語業務コーパス＋3,000件ランダム衝突テスト）、そして Chrome MV3 実装のハマりどころと決定論ルールの正直な限界までをコード付きで解説します。

---

## 1. 可逆Vaultパターンと優先度付きスパン解決

不可逆マスクではなく **可逆Vault（Reversible Token Vault）** を使うと、LLM への参照整合性を保ったまま機密をローカルに隔離できます。

```text
[入力]   請求元 T1111222333444（再掲: T1111222333444）、担当 yamada@example.co.jp
   │  redact()  ※ローカルメモリ / chrome.storage.local のみで完結
   ▼
[LLM送信] 請求元 [INVOICE_ID_1]（再掲: [INVOICE_ID_1]）、担当 [EMAIL_1]
   │  LLMが推論・要約（例: 「[INVOICE_ID_1] の件、[EMAIL_1] 宛に返信案を作成しました」）
   ▼
[復元]   restore() → 「T1111222333444 の件、yamada@example.co.jp 宛に返信案を作成しました」
```

実装上の急所は **3点** あります。

1. **同一の生値には同一プレースホルダを割り当てる**：`(kind, raw_value) -> "[INVOICE_ID_1]"` をキャッシュすることで、文書内の複数箇所・複数ターンでもLLMが同一エンティティとして追跡できる
2. **スキャン時は 1-to-1 正規化で文字オフセットを維持する**：`NFKC` を本文全体に掛けると `㈱`（1文字→3文字）等で `(start, end)` がズレるため、全角英数字と各種Unicodeハイフン（`－`, `−`, `‐`, `–`, `—`, `―`）だけを厳密に1対1変換したスキャン用文字列で正規表現を回し、得られたインデックスで元テキストを切り出す
3. **復元（`restore`）時はキー長の降順で正規表現を構築する**：`[EMAIL_1]` と `[EMAIL_10]` が混在するとき、短いキーを先に置換すると `[EMAIL_10]` の先頭部分が誤って書き換わるのを防ぐ

### Python リファレンス実装（優先度付き重複解決＋可逆Vault）

```python
from __future__ import annotations

import re

# 1文字も長さを変えない 1-to-1 スキャン用変換テーブル
_SCAN_CHAR_MAP: dict[int, str] = {
    **{ord("０") + i: str(i) for i in range(10)},
    **{ord("Ａ") + i: chr(ord("A") + i) for i in range(26)},
    **{ord("ａ") + i: chr(ord("a") + i) for i in range(26)},
    ord("－"): "-", ord("−"): "-", ord("‐"): "-",
    ord("‒"): "-", ord("–"): "-", ord("—"): "-", ord("―"): "-",
    ord("＠"): "@", ord("．"): ".", ord("＿"): "_",
    ord("＋"): "+", ord("（"): "(", ord("）"): ")", ord("\u3000"): " ",
}

_INVOICE_RE = re.compile(r"(?<![A-Za-z0-9])[Tt]\s*[-]?\s*(\d(?:[\s-]?\d){12})(?![A-Za-z0-9]|[\s-]\d)")
_CORP_RE = re.compile(r"(?<!\d[\s-])(?<![A-Za-z0-9])(\d(?:[\s-]?\d){12})(?![A-Za-z0-9]|[\s-]\d)")
_MYNUMBER_RE = re.compile(r"(?<!\d[\s-])(?<![A-Za-z0-9])(\d{4}[\s-]?\d{4}[\s-]?\d{4})(?![A-Za-z0-9]|[\s-]\d)")
_CARD_RE = re.compile(r"(?<!\d[\s-])(?<![A-Za-z0-9])((?:\d[\s-]?){13,18}\d)(?![A-Za-z0-9]|[\s-]\d)")


def is_valid_corporate_number(digits13: str) -> bool:
    """国税庁 法人番号・T番号 13桁 Mod-9 検査数字検証。"""
    cleaned = re.sub(r"[\s-]", "", digits13).lstrip("Tt")
    if len(cleaned) != 13 or not cleaned.isdigit():
        return False
    cd = int(cleaned[0])
    if cd < 1 or cd > 9:
        return False
    body = cleaned[1:]
    total = sum(int(body[12 - n]) * (1 if n % 2 == 1 else 2) for n in range(1, 13))
    return cd == 9 - (total % 9)


def is_valid_mynumber(digits12: str) -> bool:
    """総務省 マイナンバー 12桁 Mod-11 検査数字検証。"""
    cleaned = re.sub(r"[\s-]", "", digits12)
    if len(cleaned) != 12 or not cleaned.isdigit():
        return False
    cd = int(cleaned[11])
    total = sum(
        int(cleaned[11 - n]) * ((n + 1) if n <= 6 else (n - 5))
        for n in range(1, 12)
    )
    rem = total % 11
    return cd == (0 if rem <= 1 else 11 - rem)


def is_valid_luhn(digits: str) -> bool:
    """クレジットカード番号 Luhn (Mod-10) 検証。"""
    cleaned = re.sub(r"[\s-]", "", digits)
    if not (13 <= len(cleaned) <= 19) or not cleaned.isdigit() or len(set(cleaned)) == 1:
        return False
    total = 0
    for idx, ch in enumerate(cleaned[::-1]):
        val = int(ch)
        if idx % 2 == 1:
            val = val * 2 - 9 if val * 2 > 9 else val * 2
        total += val
    return total % 10 == 0


def redact_reversible(text: str) -> tuple[str, dict[str, str]]:
    """チェックデジット検証＋優先度順スパン解決で可逆マスクを実行する。"""
    scan = text.translate(_SCAN_CHAR_MAP)
    candidates: list[tuple[int, int, int, str]] = []

    for m in _INVOICE_RE.finditer(scan):
        if is_valid_corporate_number(m.group(1)):
            candidates.append((1, m.start(), m.end(), "INVOICE_ID"))
    for m in _CORP_RE.finditer(scan):
        if is_valid_corporate_number(m.group(1)):
            candidates.append((2, m.start(), m.end(), "CORPORATE_ID"))
    for m in _MYNUMBER_RE.finditer(scan):
        if is_valid_mynumber(m.group(1)):
            candidates.append((3, m.start(), m.end(), "MYNUMBER"))
    for m in _CARD_RE.finditer(scan):
        digits = re.sub(r"[\s-]", "", m.group(1))
        if len(digits) >= 14 and is_valid_luhn(digits):
            candidates.append((4, m.start(), m.end(), "CREDIT_CARD"))

    # 優先度昇順 -> スパン長降順 -> 開始位置昇順で非重複採用
    candidates.sort(key=lambda item: (item[0], -(item[2] - item[1]), item[1]))
    accepted: list[tuple[int, int, str]] = []
    for _, s, e, kind in candidates:
        if not any(not (e <= a_s or s >= a_e) for a_s, a_e, _ in accepted):
            accepted.append((s, e, kind))
    accepted.sort(key=lambda item: item[0])

    counters: dict[str, int] = {}
    raw_to_tok: dict[tuple[str, str], str] = {}
    vault: dict[str, str] = {}
    chunks: list[str] = []
    cursor = 0

    for s, e, kind in accepted:
        raw_val = text[s:e]
        key = (kind, raw_val)
        tok = raw_to_tok.get(key)
        if tok is None:
            counters[kind] = counters.get(kind, 0) + 1
            tok = f"[{kind}_{counters[kind]}]"
            raw_to_tok[key] = tok
            vault[tok] = raw_val
        chunks.append(text[cursor:s])
        chunks.append(tok)
        cursor = e
    chunks.append(text[cursor:])
    return "".join(chunks), vault


def restore_from_vault(text: str, vault: dict[str, str]) -> str:
    """長いプレースホルダから優先して元の機密値へ復元する。"""
    if not vault:
        return text
    keys = sorted(vault.keys(), key=len, reverse=True)
    pat = re.compile("|".join(re.escape(k) for k in keys))
    return pat.sub(lambda m: vault[m.group(0)], text)
```

---

## 2. 実測ベンチマーク：素朴な正規表現 vs チェックデジット検証エンジン

「チェックデジットを入れると誤検知が減る」と口で言うだけでは不十分です。実際に **50件の合成日本語業務コーパス** を構築し、素朴な正規表現ベースライン（`T?\d{13}`, `\d{12}`, `\d{16}`＋ハイフン/空白許容）と、`ComplianceGuardEngine`（1-to-1正規化＋Mod-9/Mod-11/Luhn検証）を同一条件で実測しました。

### テストコーパス設計（全50文書・期待される数値IDエンティティ29件）

意図的な「都合の良い100%」を避けるため、非PIIの業務数値IDトラップ（22件）は **チェックデジット不一致のものだけを選別せず、固定シード（`seed=20261119`）でそのまま生成** しました。

- **正例（25文書・29エンティティ）**：有効な適格請求書T番号（6件）、裸の13桁法人番号（5件）、有効な12桁マイナンバー（6件）、有効な16桁Luhnカード番号（5件）、複合業務文書（3件）。いずれも半角・ハイフン区切り・空白区切り・**全角数字（`Ｔ１２３...`）・Unicodeマイナス（`−` `U+2212`）・Unicodeハイフン（`‐` `U+2010`）** を混在
- **誤検知トラップ（22文書・期待値0件）**：JAN-13・ISBN-13・UNIXミリ秒時刻・受注番号などの13桁（7件）、夜間バッチID・宅配便追跡番号などの12桁（7件）、分散トレースID・会員コード・端末シリアルなどの16桁（5件）、1桁打ち間違えた不正T番号（3件）
- **プロンプトインジェクション攻撃サンプル（3文書）**：ゼロ幅スペース `\u200b` 混入日本語指示上書き、BiDi制御文字 `\u202e` 混入 `### System:` 偽装、`<|im_start|>` ロール区切り偽装

### 50件日本語業務コーパス 実測結果

| 評価指標 | 素朴な正規表現 (`Naive Regex`) | チェックデジット＋1対1正規化 (`ComplianceGuardEngine`) | 差分 |
|---|---:|---:|---:|
| True Positive (TP) | `23` | **`29`** | `+6` |
| False Positive (FP) | `24` | **`2`** | **`-22`（`-91.67%` 削減）** |
| False Negative (FN) | `6` | **`0`** | `-6`（取りこぼしゼロ） |
| **Precision（適合率）** | `48.94%` (`0.4894`) | **`93.55%`** (`0.9355`) | **`+44.61 pt`** |
| **Recall（再現率）** | `79.31%` (`0.7931`) | **`100.00%`** (`1.0000`) | **`+20.69 pt`** |
| **F1 Score** | `0.6053` | **`0.9667`** | **`+0.3614`** |
| 50文書合計 処理時間 | `0.60 ms` | **`3.03 ms`**（約 `0.06 ms/文書`） | 実時間影響ゼロ |
| 可逆Vault往復テスト（漏洩0＋完全一致復元） | - | **`25 / 25`（`100%`）** | - |
| プロンプトインジェクション検知 | - | **`TP 3 / FP 0 / FN 0`** | - |

内訳を見ると、なぜ素朴な正規表現が現場で破綻するかが一目瞭然です。

- **素朴な正規表現の `FN = 6`**：全角数字（`Ｔ９...`）や Unicode マイナス `−`（`U+2212`）・`‐`（`U+2010`）を含む正例6件を丸ごと取りこぼし、**生の機密番号がそのままLLMへ漏洩**
- **素朴な正規表現の `FP = 24`**：JANコード・UNIX時刻・宅配追跡番号・タイプミスT番号など、22件の非PIIトラップを全件誤爆（＋複合文書の境界誤爆2件）し、**Precision は `48.94%`（マスク箇所の半数以上が誤爆）** へ低下
- **`ComplianceGuardEngine` の残存 `FP = 2`**：ここが重要な発見です。残った2件の誤検知は、16桁の分散トレースID（`H01: 1100-4796-4699-8834`）と端末シリアル番号（`H04: 9841446566926100`）が **偶然 Luhn (Mod-10) チェックサムを満たしてしまった** ケースでした。

### ランダム3,000件衝突監査（理論値 vs 実測値）

チェックデジットの効能を誇張せず数学的な上限を押さえるため、無作為な疑似業務ID（各1,000件、`seed=42`）に対する偶発衝突率（誤検知の下限）も実測しました。

| 検証アルゴリズム | テスト件数 | 理論上の偶発通過率 | 実測偶発通過率 (`seed=42`) | 意味するところ |
|---|---:|---:|---:|---|
| 国税庁 Mod-9（先頭 `1–9`・13桁） | 1,000 | `11.11%` (`1/9`) | **`12.3%`** | JAN/ISBN/UNIX時刻の約88〜89%を弾く |
| 国税庁 Mod-9（先頭 `0–9`・13桁） | 1,000 | `10.00%` (`9/10 × 1/9`) | **`8.9%`** | 先頭`0`を全件棄却するためさらに低い |
| 総務省 Mod-11（12桁） | 1,000 | `10.00%` (`1/10`) | **`8.8%`** | 11桁固定に対し有効な12桁目は常に1通り |
| クレカ Luhn Mod-10（16桁） | 1,000 | `10.00%` (`1/10`) | **`8.5%`** | 残る ~10% を削るには IIN/BIN プレフィックス制約が効く |

つまり、**チェックデジット検証だけでランダム数値の約9割の誤検知は確実に消せるが、残る約10%の偶発衝突は数学的にゼロにはならない** のが本質です。特にクレジットカード（上記の `1100-...` や `9841...`）は、Luhn に加えて **決済ブランドの IIN/BIN 帯域（Visa `4`, Mastercard `51–55` / `2221–2720`, JCB `3528–3589`, Amex `34`/`37`）** で絞るだけで、無関係な `1` 始まり・`9` 始まりのトレースID誤検知を劇的に落とせます。

---

## 3. Chrome Manifest V3 拡張での実装と DOM 操作の全落とし穴

エンジンができたら、次は ChatGPT（`chatgpt.com`）・Claude（`claude.ai`）・Gemini（`gemini.google.com`）上でワンクリック黒塗り／復元を行う Chrome MV3 拡張に載せます。ここで多くの開発者がハマる実務ポイントを3つ整理します。

### ハマりどころ 1：React Controlled Input と ProseMirror（`contenteditable`）の内部状態ズレ

ChatGPT・Claude・Gemini の入力欄は、サイトやバージョンによって `<textarea>`（React管理）だったり `div.ProseMirror[contenteditable="true"]` だったり `<rich-textarea>` だったりします。

- React が管理する `<textarea>` に対して `el.value = nextText` だけ代入しても、React の仮想DOMトラッカーが値変化を検知せず、送信ボタンを押すと **マスク前の文字列がそのまま送信される** ことがある
- 同様に `contenteditable` の ProseMirror へ `el.innerText = nextText` を入れただけでは、直後に `input` イベントを発火しないとエディタ側の状態が同期されない

最小権限の `manifest.json`（`storage` と `activeTab` のみ、`<all_urls>` 不使用）と、ESMエンジン読み込み＋安全な DOM 読み書き・復元コードは次の通りです。

```json
{
  "manifest_version": 3,
  "name": "Local Japanese LLM Privacy Gateway",
  "version": "1.0.0",
  "permissions": ["storage", "activeTab"],
  "content_scripts": [
    {
      "matches": [
        "https://chatgpt.com/*",
        "https://claude.ai/*",
        "https://gemini.google.com/*"
      ],
      "js": ["content.js"],
      "run_at": "document_idle"
    }
  ],
  "web_accessible_resources": [
    {
      "resources": ["core/guard-engine.mjs"],
      "matches": [
        "https://chatgpt.com/*",
        "https://claude.ai/*",
        "https://gemini.google.com/*"
      ]
    }
  ]
}
```

MV3 の Content Script は通常のスクリプトとして実行されるため、共通の ESM エンジン（`core/guard-engine.mjs`）を Web アプリと共有する場合は `web_accessible_resources` に登録したうえで動的 `import(chrome.runtime.getURL("core/guard-engine.mjs"))` を使います。

```javascript
// content.js の核心部：入力欄探索・React/ProseMirror同期・TreeWalkerによる安全復元
const INPUT_SELECTORS = [
  "#prompt-textarea",
  "textarea[data-id]",
  'div.ProseMirror[contenteditable="true"]',
  'rich-textarea div[contenteditable="true"]',
  'div[contenteditable="true"][role="textbox"]',
  "textarea",
];

function findPromptTarget() {
  const active = document.activeElement;
  if (active && (active.tagName === "TEXTAREA" || active.isContentEditable)) {
    return active;
  }
  for (const sel of INPUT_SELECTORS) {
    const el = document.querySelector(sel);
    if (el) return el;
  }
  return null;
}

function writeTargetText(el, nextText) {
  if (!el) return;
  if (el.tagName === "TEXTAREA" || el.tagName === "INPUT") {
    // ReactのControlled Inputトラッカーをバイパスして確実に変更通知を飛ばす
    const proto = el.tagName === "TEXTAREA"
      ? window.HTMLTextAreaElement.prototype
      : window.HTMLInputElement.prototype;
    const nativeSetter = Object.getOwnPropertyDescriptor(proto, "value")?.set;
    if (nativeSetter) {
      nativeSetter.call(el, nextText);
    } else {
      el.value = nextText;
    }
    el.dispatchEvent(new Event("input", { bubbles: true }));
    return;
  }

  // ProseMirror / contenteditable 向け：段落テキスト置換＋InputEvent発火
  el.focus();
  el.innerText = nextText;
  el.dispatchEvent(new InputEvent("input", { bubbles: true, inputType: "insertText", data: nextText }));
}
```

### ハマりどころ 2：AI 回答の復元で `innerHTML` を触ると XSS・レイアウト崩壊が起きる

LLM が返してきた回答領域（`article`, `.markdown`, `[data-message-author-role='assistant']` 等）にある `[INVOICE_ID_1]` を元の文字列へ戻すとき、`node.innerHTML = node.innerHTML.replace(...)` は **絶対に使ってはいけません**。React の DOM 参照が壊れるうえ、元の機密値に `<` や `&` が含まれていた場合に HTML インジェクションの温床になります。

代わりに `document.createTreeWalker(node, NodeFilter.SHOW_TEXT)` を使い、**テキストノードの `nodeValue` だけを置換** します。

```javascript
function restoreAssistantResponses(engine, vaultMap) {
  const keys = Object.keys(vaultMap);
  if (keys.length === 0) return 0;

  let restoredNodes = 0;
  const containers = document.querySelectorAll(
    "article, .markdown, message-content, [data-message-author-role='assistant']",
  );
  for (const container of containers) {
    if (!container.innerText || !keys.some((k) => container.innerText.includes(k))) {
      continue;
    }
    const walker = document.createTreeWalker(container, NodeFilter.SHOW_TEXT);
    for (let textNode = walker.nextNode(); textNode; textNode = walker.nextNode()) {
      const next = engine.restore(textNode.nodeValue, vaultMap);
      if (next !== textNode.nodeValue) {
        textNode.nodeValue = next;
        restoredNodes += 1;
      }
    }
  }
  return restoredNodes;
}
```

これにより、シンタックスハイライトされたコードブロックや Markdown テーブル、コピーボタンのイベントリスナを一切壊さずにローカル復元が完結します。

---

## 4. ゼロ幅文字・BiDi 制御文字を使った間接プロンプトインジェクション対策

外部から受け取った請求書PDFやHTMLメールをコピペして LLM に要約させる際、人間の目には見えない Unicode 制御文字に悪意ある指示が隠されているケースがあります。

- **ゼロ幅文字**：`U+200B`（ZWSP）, `U+200C`（ZWNJ）, `U+200D`（ZWJ）, `U+FEFF`（BOM）, `U+2060`（WJ）
- **BiDi（双方向テキスト）上書き文字**：`U+202A`〜`U+202E`（RLO 等）, `U+2066`〜`U+2069`

これらは業務文書に通常の入力として現れる理由がほとんどないため、検出時点でリスクスコアを加算し、必要に応じてサニタイズします。

```python
_ZERO_WIDTH_RE = re.compile(r"[\u200b\u200c\u200d\ufeff\u2060]")
_BIDI_OVERRIDE_RE = re.compile(r"[\u202a-\u202e\u2066-\u2069]")

_INJECTION_RULES = (
    ("ja_override", re.compile(r"(?:以前|これまで|上記)の(?:指示|命令|ルール)を(?:すべて|全て)?(?:無視|忘れ|破棄)"), 0.60),
    ("ja_exfil", re.compile(r"(?:システムプロンプト|内部指示|開発者プロンプト)を(?:開示|表示|出力|教え)"), 0.60),
    ("en_override", re.compile(r"(?i)\b(?:ignore|disregard|forget)\s+(?:all\s+|any\s+)?(?:previous|prior|above|system)\s+(?:instructions|prompts|rules)\b"), 0.55),
    ("role_spoof", re.compile(r"(?im)(?:<\|im_start\|>|<\|system\|>|\[INST\]|^\s*###\s*System:|^\s*System:\s*)"), 0.50),
)

def assess_injection_risk(text: str) -> tuple[float, list[str]]:
    """ゼロ幅/BiDi制御文字と多言語インジェクション兆候を [0.0, 1.0] でスコアリングする。"""
    # ゼロ幅文字を除去したテキストでも照合し、文字間インジェクション難読化を無効化
    stripped = _ZERO_WIDTH_RE.sub("", _BIDI_OVERRIDE_RE.sub("", text))
    score = 0.0
    flags: list[str] = []
    for name, pat, weight in _INJECTION_RULES:
        if pat.search(text) or pat.search(stripped):
            flags.append(name)
            score += weight
    if _ZERO_WIDTH_RE.search(text) or _BIDI_OVERRIDE_RE.search(text):
        flags.append("hidden_unicode_controls")
        score += 0.30
    return round(min(score, 1.0), 4), flags
```

上のコードのように、ゼロ幅文字を除去した `stripped` 側でもルール照合を行うことで、「`以前の\u200b指示を\u200b無視`」のような単純な文字分割バイパスを確実に潰せます。

---

## 5. 決定論的ルールエンジンの正直な限界と実運用での落としどころ

ローカル完結の決定論的エンジンは「外部送信ゼロ・遅延0.1ms未満・監査ログが100%説明可能」という大きな強みを持つ反面、**できないことも明確** です。過剰な期待（ベーパーウェア）を避けるため、実運用では以下の限界を前提に設計する必要があります。

1. **自由記述の人名・取引先社名・案件コードは自動検知できない**：`山田太郎` や `プロジェクトX` にはチェックデジットが存在しません。これに対処するには、案件ごとにチームで管理する **カスタム固有名詞辞書（Custom Dictionary）** をローカル登録し、同じ可逆Vaultへ合流させるのが最も現実的です（軽量NERモデルをブラウザ内WASM/ONNXで動かす手もありますが、日本語固有名詞の誤検知・メモリ負荷のトレードオフが生じます）。
2. **チェックデジットの偶発衝突（約10%）はゼロにならない**：実測で示した通り、ランダムな16桁のトレースIDや13桁のJANコードでも約1割は偶然チェックサムを満たします。クレカは **IIN/BINプレフィックス制約**、裸の13桁法人番号は **周辺文脈キーワード（「法人番号」「取引先」等）の有無で警告レベルを分ける** 設計が有効です。
3. **意味的な間接プロンプトインジェクションは正規表現だけでは防ぎ切れない**：トリガー語を含まない巧妙なロールプレイ誘導や多段指示は、静的なパターンマッチをすり抜けます。あくまで「明らかな難読化・制御文字・定型攻撃の第一防壁（Pre-filter）」と位置づけるべきです。

それでも、「素朴な正規表現でPrecision `48.9%`・全角取りこぼし連発」という状態から、チェックデジット検証＋1-to-1正規化＋カスタム辞書＋可逆Vaultへ移行するだけで、現場のストレスと漏洩事故の大部分をゼロコスト・外部通信ゼロで潰すことができます。

今回解説したチェックデジット検証・可逆Vault・インジェクション検知を、インストール不要かつブラウザ内完結（外部通信ゼロ）で試せるデモ兼リファレンス実装を公開しています。

https://pecopeko.github.io/kurogate/

*(2026-11-21 検証環境: Python 3.11 / Node.js v22)*
---

### 📘 さらに深く実装したい方へ（体系解説書 ＆ オフライン実務キット）

本記事の可逆 Vault・チェックデジット検証・金融バックテスト偽陽性対策までを1冊（全8章・テストコード完備）に体系化した実践書と、社内PCに置いてそのまま使える完全オフライン実務ツールキットも公開しています：

https://zenn.dev/pecopeko/books/afml-quant-and-local-ai-guard

- **経理・バックオフィス自動化5点セット Pro キット（Shift-JIS 全銀120B・電帳法PDF一括リネームZIP・支払調書・AI社外秘ガード収録）**：
https://pecopeko1011.booth.pm/items/9179705
