---
title: "全銀120バイトShift-JIS・T番号(Mod-9)・マイナンバー(Mod-11)実装の全落とし穴"
emoji: "🏦"
type: "tech"
topics: ["javascript", "python", "fintech", "japan", "shiftjis"]
published: true
---

## はじめに：日本の経理・決済フォーマットは「仕様の1バイト差」で本番が止まる

日本のB2Bシステム・経理バッチ・SaaSを開発していると、高確率で次の3つの国内仕様に突き当たります。

1. **全銀協規定形式（総合振込・給与振込の1レコード120バイト固定長ファイル）**
2. **国税庁の法人番号（13桁）および適格請求書T番号（`T` + 13桁、Mod-9）**
3. **総務省のマイナンバー（個人番号12桁、Mod-11）**

どれも一見すると「単純な文字列変換や剰余計算」に見えます。ところが実際に実装して銀行のインターネットバンキングや業務文書パーサへ流し込むと、**UTF-8とShift-JISのバイト数乖離・前株/後株の括弧位置・30バイト目の濁点切断・剰余0の境界バグ・NFKCによるオフセット崩壊** といった地雷を次々に踏むことになります。

本記事では、外部ライブラリに一切依存せず（Zero Dependencies）、JavaScript と Python の両方で安全に実装するための全落とし穴とコピペ可能な検証済みコードを解説します。

---

## 落とし穴 1：`new Blob([str])` は UTF-8 で出力される（半角カナは1バイトではなく3バイトになる）

ブラウザで全銀総合振込ファイルを生成するとき、最も多い本番事故がこれです。

```javascript
// ❌ 銀行ポータルでほぼ確実にエラーになる実装
const blob = new Blob([zenginText], { type: "text/plain" });
```

JavaScript の文字列は内部的に UTF-16 であり、`new Blob([str])` や `new TextEncoder().encode(str)` は文字列を **UTF-8** へエンコードします。

- ASCII 文字（`0-9`, `A-Z`, スペースなど）：UTF-8 でも Shift-JIS（CP932）でも **1バイト**
- 半角カタカナ（`ｱ` = `U+FF71` 〜 `ﾟ` = `U+FF9F`）：
  - **CP932 / Shift-JIS**：`0xA1` 〜 `0xDF` の **1バイト**
  - **UTF-8**：`EF BD B1` のように **3バイト**

たとえば受取人名に `ﾔﾏﾀﾞ ﾀﾛｳ`（スペース除く半角カナ8文字、うち濁点1文字）が含まれていると、JavaScript 上で `line.length === 120` であっても、UTF-8 出力では `120 + (8 × 2) = 136 バイト` になってしまいます。銀行のEB/IBシステムは **1レコード＝Shift-JIS 120バイト（+ CRLFで122バイト）** を厳密に要求するため、文字数検証を通過しているのに銀行側でレコード長エラーとして弾かれます。

### 外部ライブラリゼロの JavaScript CP932（全銀文字セット）エンコーダ

全銀協の許容文字は「ASCII印字文字（`0x20`〜`0x7E`）」「半角カナ（`U+FF61`〜`U+FF9F`）」「改行（`CR/LF`）」に限られるため、重い `encoding-japanese` 等を持ち込まなくても、**20行の `Uint8Array` マッピングだけで厳密な Shift-JIS バイト列** を出力できます。

```javascript
/**
 * 全銀許容文字列（ASCII + 半角カナ + CRLF）を厳密な CP932/Shift-JIS バイト列へ変換する。
 * 半角カナ U+FF61..U+FF9F は JIS X 0201 の 0xA1..0xDF（1バイト）に1対1対応する。
 */
export function encodeZenginShiftJIS(text) {
  const bytes = new Uint8Array(text.length);
  for (let i = 0; i < text.length; i += 1) {
    const code = text.charCodeAt(i);
    if (code === 0x0d || code === 0x0a || (code >= 0x20 && code <= 0x7e)) {
      bytes[i] = code;
    } else if (code >= 0xff61 && code <= 0xff9f) {
      // U+FF61 (｡) -> 0xA1, ..., U+FF71 (ｱ) -> 0xB1, ..., U+FF9E (ﾞ) -> 0xDE, U+FF9F (ﾟ) -> 0xDF
      bytes[i] = code - 0xff61 + 0xa1;
    } else {
      throw new Error(
        `全銀Shift-JIS非対応文字を検出 (index=${i}, char="${text[i]}", U+${code.toString(16).toUpperCase()})`,
      );
    }
  }
  return bytes;
}

// ダウンロード用 Blob は必ず Uint8Array から生成する
export function createZenginBlob(zenginTextWithCrlf) {
  const sjisBytes = encodeZenginShiftJIS(zenginTextWithCrlf);
  return new Blob([sjisBytes], { type: "application/octet-stream" });
}
```

Python で生成・検証する場合も、文字数 `len(line)` ではなく必ず `len(line.encode("cp932")) == 120` をアサートします。

```python
def assert_zengin_record_bytes(line: str) -> bytes:
    """1レコードが厳密に CP932 120バイトであることを検証してバイト列を返す。"""
    raw = line.encode("cp932")
    if len(raw) != 120:
        raise ValueError(f"CP932バイト長エラー: expected=120, actual={len(raw)}")
    return raw
```

### 総合振込（データ区分 1 / 2 / 8 / 9）の120バイト実レイアウト

全銀総合振込の4レコードはすべて **合計120バイト** で設計されています。1桁でもダミー空白の計算を誤ると全行がズレます。

| レコード | 区分(1B) | 主なフィールド構成（バイト数） | 末尾ダミー | 合計 |
|---|---:|---|---:|---:|
| ヘッダ | `1` | 種別`21`(2) + コード区分`0`(1) + 委託者コード(10) + 委託者名(40) + 振込日MMDD(4) + 仕向銀行番号(4) + 銀行名(15) + 支店番号(3) + 支店名(15) + 預金種目(1) + 口座番号(7) | 17B | **120B** |
| データ | `2` | 被仕向銀行番号(4) + 銀行名(15) + 支店番号(3) + 支店名(15) + 手形交換所(4) + 預金種目(1) + 口座番号(7) + **受取人名(30)** + 振込金額(10) + 新規コード(1) + EDI情報(20) + 振込指定区分(1) | 8B | **120B** |
| トレーラ | `8` | 合計件数(6, 右詰0埋) + 合計金額(12, 右詰0埋) | 101B | **120B** |
| エンド | `9` | （フィールドなし） | 119B | **120B** |

特にトレーラ・レコードの件数（6桁）および合計金額（12桁）が明細レコードの集計値と1円でも食い違うと銀行側でバッチ全体がリジェクトされるため、生成時とパース時の双方で必ず検算します。

---

## 落とし穴 2：法人略語の位置ルール（`ｶ)` / `(ｶ` / `(ｶ)`）と「ヲ・長音ー・小文字カナ」禁止仕様

全銀協の受取人名（30バイト）や委託者名（40バイト）には、日本語の一般的なカナ変換だけでは通らない厳格な文字ルールがあります。

### 銀行で弾かれる4つの頻出文字

1. **「ヲ（ｦ）」は使用不可** → 大文字の **`ｵ`** に置換（例：`ヲグリ` → `ｵｸﾞﾘ`）
2. **長音記号「ー（`U+30FC`）」「ｰ（`U+FF70`）」は使用不可** → ASCII ハイフンマイナス **`-`（`0x2D`）** に置換（例：`スーパー` → `ｽ-ﾊﾟ-`）
3. **小文字カナ（`ァィゥェォッャュョ`・半角 `ｧｨｩｪｫｯｬｭｮ`）は使用不可** → 大文字の半角カナ（`ｱｲｳｴｵﾂﾔﾕﾖ`）に昇格（例：`キヤノン` / `キャノン` → `ｷﾔﾉﾝ`、`ニッポン` → `ﾆﾂﾎﾟﾝ`）
4. **英小文字（`a-z`）は使用不可** → 英大文字（`A-Z`）へ統一

### 法人略語の位置依存ルール（前株・後株・中株）

よくある雑な実装が `.replace(/株式会社/g, "(ｶ)")` です。これだと先頭や末尾の法人格でも両側に括弧が付いてしまい、銀行の口座名義照合で不一致（組戻しリスク）になります。全銀協の正式な略語ルールは **位置依存** です。

- **前株（先頭）**：`株式会社〇〇` → **`ｶ)〇〇`**（閉じ括弧のみ）
- **後株（末尾）**：`〇〇株式会社` → **`〇〇(ｶ`**（開き括弧のみ）
- **中株（途中）**：`〇〇株式会社△△` → **`〇〇(ｶ)△△`**（両括弧）

有限会社（`ﾕ)` / `(ﾕ` / `(ﾕ)`）、合同会社（`ﾄﾞ)` / `(ﾄﾞ` / `(ﾄﾞ)`）も同様です。正規表現の先頭 `^`・末尾 `$` を明示して置換順序を制御すれば、漏れなく変換できます。

---

## 落とし穴 3：濁点・半濁点の「30バイト目切断バグ（`ｶﾞ` → `ｶ` 破壊）」

全銀フォーマットでは、濁点 `ﾞ`（`0xDE`）と半濁点 `ﾟ`（`0xDF`）はそれぞれ **独立した1バイト** を消費します。つまり「ガ」は `ｶ`（1B）+ `ﾞ`（1B）＝ **2バイト** です。

ここで、受取人名フィールド（最大30バイト）に対して次のように単純スライスすると悲劇が起きます。

```javascript
// ❌ 30バイト目が「ｶ」、31バイト目が「ﾞ」だった場合、「ガ」が「カ」に化ける！
const broken = zenginName.slice(0, 30);
```

たとえば29バイトの社名の末尾に「銀行（`ｷﾞﾝｺｳ`）」や「ガ（`ｶﾞ`）」が来て、インデックス `29`（30バイト目）が `ｶ`、インデックス `30`（31バイト目）が `ﾞ` だった場合：

- `slice(0, 30)` は **`ﾞ` だけを切り落とし、末尾を `ｶ` に変質させてしまう**
- さらに分割フィールドへ後続を流す処理では、先頭バイトが孤立した `ﾞ` になり、銀行側で不正文字エラーになる

### 濁点切断を防ぐ安全な全銀カナ正規化・30バイト切り詰め実装（JavaScript）

```javascript
const KANA_MAP = {
  ア: "ｱ", イ: "ｲ", ウ: "ｳ", エ: "ｴ", オ: "ｵ",
  ァ: "ｱ", ィ: "ｲ", ゥ: "ｳ", ェ: "ｴ", ォ: "ｵ",
  カ: "ｶ", キ: "ｷ", ク: "ｸ", ケ: "ｹ", コ: "ｺ",
  サ: "ｻ", シ: "ｼ", ス: "ｽ", セ: "ｾ", ソ: "ｿ",
  タ: "ﾀ", チ: "ﾁ", ツ: "ﾂ", テ: "ﾃ", ト: "ﾄ", ッ: "ﾂ",
  ナ: "ﾅ", ニ: "ﾆ", ヌ: "ﾇ", ネ: "ﾈ", ノ: "ﾉ",
  ハ: "ﾊ", ヒ: "ﾋ", フ: "ﾌ", ヘ: "ﾍ", ホ: "ﾎ",
  マ: "ﾏ", ミ: "ﾐ", ム: "ﾑ", メ: "ﾒ", モ: "ﾓ",
  ヤ: "ﾔ", ユ: "ﾕ", ヨ: "ﾖ", ャ: "ﾔ", ュ: "ﾕ", ョ: "ﾖ",
  ラ: "ﾗ", リ: "ﾘ", ル: "ﾙ", レ: "ﾚ", ロ: "ﾛ",
  ワ: "ﾜ", ヲ: "ｵ", ン: "ﾝ", ヮ: "ﾜ", ヵ: "ｶ", ヶ: "ｹ", ヰ: "ｲ", ヱ: "ｴ",
  ガ: "ｶﾞ", ギ: "ｷﾞ", グ: "ｸﾞ", ゲ: "ｹﾞ", ゴ: "ｺﾞ",
  ザ: "ｻﾞ", ジ: "ｼﾞ", ズ: "ｽﾞ", ゼ: "ｾﾞ", ゾ: "ｿﾞ",
  ダ: "ﾀﾞ", ヂ: "ﾁﾞ", ヅ: "ﾂﾞ", デ: "ﾃﾞ", ド: "ﾄﾞ",
  バ: "ﾊﾞ", ビ: "ﾋﾞ", ブ: "ﾌﾞ", ベ: "ﾍﾞ", ボ: "ﾎﾞ",
  パ: "ﾊﾟ", ピ: "ﾋﾟ", プ: "ﾌﾟ", ペ: "ﾍﾟ", ポ: "ﾎﾟ", ヴ: "ｳﾞ",
  // 半角小文字・半角ヲ・半角長音も規定文字へ補正
  ｧ: "ｱ", ｨ: "ｲ", ｩ: "ｳ", ｪ: "ｴ", ｫ: "ｵ", ｯ: "ﾂ", ｬ: "ﾔ", ｭ: "ﾕ", ｮ: "ﾖ",
  ｦ: "ｵ", ｰ: "-", ー: "-", "－": "-", "―": "-", "−": "-", "‐": "-",
  "　": " ", "．": ".", "（": "(", "）": ")", "／": "/", "，": ",", "、": ",", "・": ".",
};

const CORP_RULES = [
  { re: /^株式会社\s*/, rep: "ｶ)" },
  { re: /\s*株式会社$/, rep: "(ｶ" },
  { re: /株式会社/g, rep: "(ｶ)" },
  { re: /^有限会社\s*/, rep: "ﾕ)" },
  { re: /\s*有限会社$/, rep: "(ﾕ" },
  { re: /有限会社/g, rep: "(ﾕ)" },
  { re: /^合同会社\s*/, rep: "ﾄﾞ)" },
  { re: /\s*合同会社$/, rep: "(ﾄﾞ" },
  { re: /合同会社/g, rep: "(ﾄﾞ)" },
];

// 全銀許容文字（ｦ・ｰ・小文字カナを除外した厳密正規表現）
const STRICT_ZENGIN_RE = /^[A-Z0-9ｱ-ﾝﾞﾟ \-\.\(\)\/,]*$/;

export function toSafeZenginName(raw, maxBytes = 30) {
  let s = String(raw ?? "")
    .trim()
    .replace(/[！-～]/g, (ch) => String.fromCharCode(ch.charCodeAt(0) - 0xfee0))
    .replace(/㈱/g, "株式会社")
    .replace(/㈲/g, "有限会社");

  for (const { re, rep } of CORP_RULES) {
    s = s.replace(re, rep);
  }
  s = s.toUpperCase();

  const out = [];
  const unsupported = [];
  for (const rawCh of s) {
    const cp = rawCh.charCodeAt(0);
    // ひらがな (U+3041..U+3096) -> カタカナ (+0x60)
    const ch = cp >= 0x3041 && cp <= 0x3096 ? String.fromCharCode(cp + 0x60) : rawCh;
    if (Object.prototype.hasOwnProperty.call(KANA_MAP, ch)) {
      out.push(KANA_MAP[ch]);
    } else if (STRICT_ZENGIN_RE.test(ch)) {
      out.push(ch);
    } else {
      unsupported.push(rawCh);
    }
  }

  let normalized = out.join("").replace(/\s{2,}/g, " ").trim();
  let truncated = false;

  if (normalized.length > maxBytes) {
    truncated = true;
    // 境界の直後 (index === maxBytes) が濁点・半濁点なら、直前の基底カナごと1バイト手前で切る！
    let cut = maxBytes;
    if (normalized[cut] === "ﾞ" || normalized[cut] === "ﾟ") {
      cut -= 1;
    }
    normalized = normalized.slice(0, cut).trimEnd();
  }

  return {
    zenginText: normalized,
    byteLength: normalized.length, // 全文字がCP932 1バイト確定のため一致
    truncated,
    unsupportedChars: [...new Set(unsupported)],
    isLegal: STRICT_ZENGIN_RE.test(normalized),
  };
}
```

試しに `"カ".repeat(29) + "ガ"`（31バイト、29文字目の `ｶ` の直後に `ｶﾞ` が来て30〜31バイトをまたぐケース）を渡すと、単純スライスでは `"...ｶｶ"`（`ガ` が `カ` に変質）になるところ、上記実装では `cut = 29` に引き戻し、破壊のない29バイト（＋右空白パディング1バイト）で安全に収まります。

---

## 落とし穴 4：国税庁 法人番号・T番号（Mod-9）の「剰余0」と「左から数える」バグ

インボイス制度の適格請求書発行事業者登録番号（`T` + 13桁の法人番号）は、国税庁のWeb-APIへ問い合わせる前に **ローカルで検査数字（チェックデジット）を検証** すれば、タイプミスの約89%（9分の8）を0ミリ秒・API通信ゼロで即座に弾けます。

### 国税庁 Mod-9 の正式仕様

13桁の法人番号を左から $c, p_{12}, p_{11}, \dots, p_2, p_1$ とします（**最左桁 $c$ が検査数字**、右端から1始まりで $p_1 \dots p_{12}$）。

$$
S = \sum_{n=1}^{12} p_n \cdot q_n, \qquad
q_n = \begin{cases} 1 & (n \text{ が奇数}) \\ 2 & (n \text{ が偶数}) \end{cases}
$$

$$
\text{expected} = 9 - (S \bmod 9)
$$

ここで実務コードに紛れ込みやすい **3つのバグ** があります。

1. **`(9 - S) % 9` と書いてしまうバグ**：`S % 9 == 0` のとき、正しくは `9 - 0 = 9` なのに `(9 - 0) % 9 = 0` を返してしまう。国税庁法人番号の検査数字は **`1`〜`9` のみで、`0` は仕様上存在しません**。
2. **左から重み `1, 2, 1, 2...` を掛けてしまうバグ**：12桁なので右から奇数番目（最右端 $p_1$）の重みが `1`、左端 $p_{12}$ の重みが `2` です。逆向きにすると奇偶がすべて反転します。
3. **個人事業主のT番号の扱い**：個人事業主には法人番号がありませんが、適格請求書登録番号（`T` + 13桁）の末尾12桁＋先頭1桁には **法人番号とまったく同じ Mod-9 検査数字ルール** が適用されているため、法人・個人問わず同一アルゴリズムで検証できます。

### Python & JavaScript 検証済み実装（Mod-9）

```python
import re

def is_valid_corporate_or_t_number(raw: str) -> bool:
    """13桁の法人番号 または T+13桁の適格請求書登録番号を国税庁Mod-9で検証する。"""
    cleaned = re.sub(r"[\s-]", "", raw).upper()
    if cleaned.startswith("T"):
        cleaned = cleaned[1:]
    if len(cleaned) != 13 or not cleaned.isdigit():
        return False

    check_digit = int(cleaned[0])
    if check_digit < 1 or check_digit > 9:
        return False  # 検査数字 0 は仕様上ありえない

    body = cleaned[1:]  # 12桁: body[11] が p_1（右端）
    weighted_sum = 0
    for n in range(1, 13):
        p_n = int(body[12 - n])
        q_n = 1 if (n % 2 == 1) else 2
        weighted_sum += p_n * q_n

    expected = 9 - (weighted_sum % 9)  # 値域は 1..9
    return check_digit == expected

# 国税庁自身の法人番号（7000012050002）等でのスモークテスト
assert is_valid_corporate_or_t_number("7000012050002") is True
assert is_valid_corporate_or_t_number("T7000012050002") is True
assert is_valid_corporate_or_t_number("T8000012050002") is False  # 検査数字1桁違い
assert is_valid_corporate_or_t_number("0000012050002") is False   # 先頭0は常に不正
```

```javascript
export function verifyNtaMod9(raw) {
  const cleaned = String(raw ?? "")
    .replace(/[！-～]/g, (ch) => String.fromCharCode(ch.charCodeAt(0) - 0xfee0))
    .replace(/[-\s]/g, "")
    .toUpperCase();
  const isInvoice = cleaned.startsWith("T");
  const digits = isInvoice ? cleaned.slice(1) : cleaned;
  if (!/^\d{13}$/.test(digits)) return { valid: false, reason: "FORMAT_ERROR" };

  const actual = Number(digits[0]);
  if (actual < 1 || actual > 9) return { valid: false, reason: "ZERO_CHECK_DIGIT" };

  let sum = 0;
  for (let n = 1; n <= 12; n += 1) {
    const p = Number(digits[13 - n]);
    sum += p * (n % 2 === 1 ? 1 : 2);
  }
  const expected = 9 - (sum % 9);
  return { valid: actual === expected, expected, actual, normalized: isInvoice ? `T${digits}` : digits };
}
```

---

## 落とし穴 5：マイナンバー（Mod-11）の重み循環と余り `0 / 1` の境界罠

マイナンバー（個人番号12桁）は法人番号と逆で、**最右端（12桁目）が検査数字** です。左11桁を右端から $p_1, p_2, \dots, p_{11}$ とし、重み $q_n$ を次のように定義します（総務省令第85号）。

$$
q_n = \begin{cases} n + 1 & (1 \le n \le 6) \quad \Rightarrow 2, 3, 4, 5, 6, 7 \\ n - 5 & (7 \le n \le 11) \quad \Rightarrow 2, 3, 4, 5, 6 \end{cases}
$$

$$
r = \left(\sum_{n=1}^{11} p_n \cdot q_n\right) \bmod 11, \qquad
\text{expected} = \begin{cases} 0 & (r \le 1) \\ 11 - r & (r \ge 2) \end{cases}
$$

ここでよくある実装バグは **`rem == 1` のケースを落として `11 - 1 = 10` を出してしまうこと** です。`rem == 0` だけでなく **`rem <= 1` の両方を `0` に丸める** 必要があります（このため12桁目の検査数字 `0` は他の数字の2倍の頻度＝`2/11` で出現します）。

```python
def is_valid_mynumber_mod11(raw: str) -> bool:
    """12桁の個人番号（マイナンバー）を総務省令Mod-11で検証する。"""
    cleaned = re.sub(r"[\s-]", "", raw)
    if len(cleaned) != 12 or not cleaned.isdigit():
        return False

    check_digit = int(cleaned[11])
    weighted_sum = 0
    for n in range(1, 12):
        p_n = int(cleaned[11 - n])
        q_n = (n + 1) if n <= 6 else (n - 5)
        weighted_sum += p_n * q_n

    rem = weighted_sum % 11
    expected = 0 if rem <= 1 else (11 - rem)
    return check_digit == expected
```

---

## 落とし穴 6：全角・Unicodeハイフン正規化で `NFKC` を使うと文字オフセットが壊れる

日本の実務テキスト（請求書メール、Excelコピペ、PDF抽出テキスト）には、半角ハイフン `-`（`U+002D`）の代わりに以下の Unicode ダッシュ類が混在します。

- `－`（`U+FF0D` 全角ハイフンマイナス）
- `−`（`U+2212` 数学マイナス記号）
- `‐`（`U+2010` Unicode ハイフン）
- `‒` / `–` / `—` / `―`（`U+2012`〜`U+2015` 各種ダッシュ）

さらに `Ｔ１２３４` のような全角英数字も日常茶飯事です。これを手っ取り早く吸収しようとして **テキスト全体に `unicodedata.normalize("NFKC", text)` を掛けてから正規表現でスキャンすると、ハイライトや伏せ字置換の `(start, end)` オフセットが壊れます**。

理由は明快で、NFKC は文字数を変える変換を多数含むためです。

- `㈱`（1文字） → `(株)`（3文字、+2ズレる）
- `ｶﾞ`（2文字） → `ガ`（1文字、-1ズレる）
- `㍿`（1文字） → `株式会社`（4文字、+3ズレる）

### 解決策：スキャン専用の「厳密1対1文字変換テーブル」

元テキストの文字長を **1文字も変えない 1-to-1 マップ** でスキャン専用文字列を作り、検出した `(start, end)` をそのまま元テキストへスライス適用すれば、オフセットのズレは構造的にゼロになります。

```python
# 元文字列と1文字も長さが変わらない 1-to-1 スキャン用変換テーブル
_SCAN_CHAR_MAP: dict[int, str] = {
    **{ord("０") + i: str(i) for i in range(10)},
    **{ord("Ａ") + i: chr(ord("A") + i) for i in range(26)},
    **{ord("ａ") + i: chr(ord("a") + i) for i in range(26)},
    ord("－"): "-",  # U+FF0D
    ord("−"): "-",  # U+2212
    ord("‐"): "-",  # U+2010
    ord("‒"): "-",  # U+2012
    ord("–"): "-",  # U+2013
    ord("—"): "-",  # U+2014
    ord("―"): "-",  # U+2015
    ord("\u3000"): " ",  # 全角スペース -> 半角スペース
}

def to_scan_text_preserve_offsets(original: str) -> str:
    """文字数とインデックスを完全保存したまま英数字・ハイフン・空白だけ正規化する。"""
    scanned = original.translate(_SCAN_CHAR_MAP)
    assert len(scanned) == len(original)
    return scanned

# 例: "㈱サンプル 登録番号：Ｔ７０００−０１２０−５０００−２" でも
# "㈱" が1文字のまま残るため、scanned で見つけた (start, end) が original に厳密一致する！
```

この「1-to-1スキャン用文字列 → 検出オフセットを元テキストに適用」という分離設計は、請求書番号チェッカーだけでなく、次記事で扱うローカル PII マスキングやコードエディタ上の Linter 実装でも極めて効きます。

---

## おわりに：ブラウザ完結で手元検証したい場合

今回解説した「国税庁 Mod-9 適格請求書T番号の一括検証」「全銀半角カナ正規化」「120バイト総合振込データの生成・トレーラ検算」を、外部サーバーへのデータ送信ゼロ（ブラウザ内JavaScript完結）ですぐ試せる検証ツールを公開しています。テストデータの挙動確認や経理バッチの突き合わせにご活用ください。

https://pecopeko.github.io/zengingate/

