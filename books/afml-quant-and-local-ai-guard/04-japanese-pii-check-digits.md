---
title: "第4章: 正規表現の誤検知を潰す国税庁Mod-9・マイナンバーMod-11検査アルゴリズム"
---

社内の請求書・顧客CSVを生成AIへ渡す前のマスク処理で、単純な `\\d{13}` や `\\d{12}` の正規表現だけを使うと、注文番号・SKU・タイムスタンプまで片っ端から `[CORP_ID]` 等に潰れて業務が止まります。

これを防ぐのが **公的チェックデジット（検査数字）の厳密検証** です。

## 適格請求書T番号（T+13桁）・法人番号の Modulo-9 実装

国税庁仕様では最左桁が検査数字であり、右端を第1桁として奇数桁×1・偶数桁×2 の総和から `9 - (sum % 9)` で検査数字を求めます。さらに実務では全角文字（`Ｔ７００００...`）やハイフン区切りを事前正規化します。

```js
export function verifyCorporateNumberMod9(raw) {
  const half = String(raw ?? "").replace(/[！-～]/g, (c) =>
    String.fromCharCode(c.charCodeAt(0) - 0xfee0),
  );
  const digits = half.replace(/[-\s]/g, "");
  if (!/^\d{13}$/.test(digits)) return false;
  const check = Number(digits[0]);
  let sum = 0;
  for (let i = 1; i <= 12; i += 1) {
    sum += Number(digits[13 - i]) * (i % 2 === 1 ? 1 : 2);
  }
  return check === 9 - (sum % 9);
}
```

## リバーシブル・ボルト（Vault）パターン

1文書（またはバッチCSVの全行）内で同一エンティティには同じプレースホルダー（`[INVOICE_ID_1]` 等）を割り当てて記憶し、AIの回答取得後に手元のブラウザメモリ内で元文字列へ復元します。外部APIへ原本データを一切送信しないゼロトラスト設計が成り立ちます。
