# クラスとオーバーロード

SC0 に型クラスの専用機構はほとんどありません。原則は：

- **クラス＝ただの nominal レコード型**（辞書の型）
- **インスタンス＝ただの値**（辞書）
- 専用なのは**探索**だけ：implicit 引数の型がクラス（nominal レコード）のとき、
  `instance` 宣言された値から解決される

```
nominal AddOp a b = { result : U, add : a -> b -> result };   -- クラス
instance addRR : AddOp R R = { result = R, add = addR };      -- インスタンス
let _+_ : (?a : U) -> (?b : U) -> (?d : AddOp a b) -> a -> b -> d.result
        = λ?a ?b ?d x y. d.add x y;                           -- 演算子＝辞書を射影するだけの関数
```

`a + b` と書くと `_+_ a b` に脱糖され、implicit `?d : AddOp a b` が探索で埋まります。

## 結果型依存（異種演算）

クラスのレコードが `result : U` フィールドを持ち、演算の結果型が**辞書の値に依存**します
（`a -> b -> d.result`）。これにより異種オペランドの演算も同じ枠で表せます。
比較も同様で、`LtOp N N` の `result` は `B` です。

## 標準のオーバーロード演算子

`std` が `AddOp` / `SubOp` / `MulOp` / `DivOp` / `LtOp` / `EqOp` / `NegOp` と、
`N`/`Z`/`R`（`EqOp` は `B` も）の同種オペランドのインスタンスを定義しています。
`*Op` という接尾辞は「演算子の配線」であることの明示で、素の `Add` や `Lt` は
数学構造・命題のために空けてあります（`base` の `Lt : N -> N -> U` は命題）。

## parametric instance（関数インスタンス）

インスタンスは**関数**でも書けます。前提（premise）を implicit に取り、探索が再帰します。

```
-- 「a 同士が足せるなら Vec n a 同士も足せる」
instance addVec : (?n : N) -> (?a : U) -> (?d : AddOp a a) -> AddOp (Vec n a) (Vec n a)
  = λ?n ?a ?d. { result = Vec n a, add = zipWith d.add };
```

`[1.0, 2.0] + [3.0, 4.0]` は `AddOp (Vec 2 R) (Vec 2 R)` を探索 → `addVec` に単一化 →
前提 `AddOp R R` を再帰探索 → `addRR`、という順で解決されます。

## 無名インスタンスと open

```
instance e;              -- 名前をつけずに e をインスタンス化
instance open e;         -- e（レコード）の全フィールドをそれぞれインスタンス化
```

`instance open` は「導出された辞書の束」を一括で配線するのに使います。たとえば
`algebra` の `ringOps : Ring m add mul -> { add : AddOp m m, sub : …, mul : …, neg : … }`
は環から四則の配線を**普通の関数**として導出するので、

```
instance open ringOps someRing;    -- これで + - * と単項 - が m で使える
```

と書けます（導出に魔法はない）。なお、式のローカルスコープで `open r in body` のように
辞書を開く構文は**ありません**——必要ならフィールドを明示的に分配束縛します。

## From と変換

変換は `From` クラス一本です。型検査器に変換の特殊処理はありません。

```
nominal From a b = { from : a -> b };
let from : (?a : U) -> (?b : U) -> (?d : From a b) -> a -> b = …;
```

標準のインスタンス（**無損失の塔上げのみ**）：

| From | 内容 |
|---|---|
| `From N N` / `From Z Z` / `From R R` | 恒等 |
| `From N Z`, `From N R`, `From Z R` | 塔上げ |
| `From Rational R` | 小数リテラルの解決（`Rational = {num : Z, den : N}`・約分しない中間表現） |
| `From (Vec k A) (Buffer 1 [k] A)` | `from [a, b, c]` で特定データのバッファ直書き |

`R → N` のような損失のある変換は `From` にはなく、明示の関数
（`floorN : R -> N`、`floorInt : R -> Z`）を使います。

## リテラルの解決

`From` には**数値リテラルも乗ります**。`3` は「`N` のリテラルから期待型への `From`」として
解決され、小数は `Rational` を経由します。重要な帰結：

- **文脈から型が決まらないリテラルは曖昧エラー**になります。既定型（Haskell の
  defaulting のようなもの）は**ありません**（No ad-hoc）。
  `some 5` が単体で通らないのは `5` が曖昧だからで、`some (5 as N)` は通ります。
- Vec / レコードのリテラルが nominal 型として生まれる（born-as）のも同じ機構です。
  期待型（未解決のメタ変数を含む）から born が**遅延で**決まるため、
  `iterate 24 [0.0, 0.0] step` の `[0.0, 0.0]` は結果型が `Complex` なら注釈不要です。

**⚙ 実装メモ（限界）**：parametric instance と裸の整数リテラルの組み合わせでは、
結果型の注釈がオペランドまで逆流しないことがあります（`([1, 2] + [3, 4]) as Vec 2 N` は
オペランド側に注釈が要る）。Vec リテラル自体の born は遅延解決で救われますが、
中の数値リテラルの曖昧さは残る場合があります。

## 具体例：数学構造ライブラリ（algebra / math）

ここまでが機構のすべてで、以下は**その上に普通のモジュールとして書かれたライブラリの一例**です
（機構側に algebra / math への特別扱いは何もありません）。

`algebra` は `*Op` の配線階層とは別に、法則つきの数学構造を定義しています
（`Semigroup` / `Monoid` / `CommMonoid` / `Group` / `CommGroup` / `Semiring` / `Ring`）。

- **演算を値パラメータに取ります**：`Monoid m (op : m -> m -> m)`。
  「`N` の加法モノイド」と「`N` の乗法モノイド」は別の型です。
- 法則はフィールドに命題的等価で載ります（`unitL : (x : m) -> op e x ≡ x`）。
  現状のインスタンス（`math` の N / Z / Complex / Quaternion）は法則を `trustMe` で
  埋めています——基底型の性質は postulate 扱いのため。書ける証明は書くのが方針です
  （[型システム](user/types.md#命題的等価id--と-j)）。
- `math` が `N`（半環）・`Z`（環）・`Complex`・`Quaternion` の構造を提供します
  （[標準ライブラリ](user/stdlib.md)）。
