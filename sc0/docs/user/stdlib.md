# 標準ライブラリ

ソースそのものが `core/modules/*.sc0` にあり、`opaque`（ネイティブ供給）以外は
すべて SC0 で書かれた普通の定義です。ここでは全メンバを列挙します。

## base — 公開プリミティブ

`std` が再公開するため、すべて非修飾で使えます。

### 基底型と別名

`Type`（宇宙・type-in-type）、`B` `N` `Z` `R`。
別名 `U = Type`、`Bool` `Nat` `Int` `Real`。

### 型形成子・列挙

| 名前 | 型 |
|---|---|
| `Vec` | `N -> U -> U` |
| `Buffer` | `(d : N) -> Vec d N -> U -> U` |
| `Interp` | `enum { nearest, linear, cubic }`（sampler の補間） |
| `Sampling` | `enum { center, edge, closed }`（render のセル内サンプル位置） |

### ベクトル / バッファ演算

| 名前 | 型（implicit は省略） |
|---|---|
| `dot` | `Vec n R -> Vec n R -> R` |
| `replicate` | `(n : N) -> T -> Vec n T` |
| `map` | `(A -> B) -> Vec n A -> Vec n B` |
| `zipWith` / `zipWith3` | `(A -> B -> C) -> Vec n A -> Vec n B -> Vec n C`（3 引数版も） |
| `fold` | `B -> (A -> B -> B) -> Vec n A -> B`（左畳み込み・初期値 → 関数の引数順） |
| `scan` | `B -> (A -> B -> B) -> Vec n A -> Vec n B` |
| `iterate` | `(n : N) -> T -> (T -> T) -> T`（有界反復） |
| `for` | `(n : N) -> T -> (N -> T -> T) -> T`（添字つき有界反復） |
| `parallel` | `(size : Vec d N) -> (Vec d N -> T) -> Buffer d size T`（原始の実体化・IR 経路） |
| `parallel_fallback` | 同上（明示 per-sample 版・IR 化しない） |
| `sampler` | `Interp -> Buffer d shape T -> Vec d R -> T` |

### 数学関数（R 上）

`sin` `cos` `tan` `sqrt` `exp` `log` `abs` `floor` `ceil` `round`（`R -> R`）、
`pow` `min` `max` `atan2`（`R -> R -> R`）、`lerp`（`R -> R -> R -> R`）、`pi`（`R`）。

`rand : Vec n N -> R` — seed ハッシュの決定的擬似乱数（[評価モデル](user/evaluation.md#乱数)）。

### 数値の塔をまたぐ変換（損失あり・明示）

| 名前 | 型 |
|---|---|
| `floorN` | `R -> N`（床・負は 0 飽和） |
| `floorInt` | `R -> Z` |
| `toR` | `Z -> R` |

### 命題的等価

`Id : (A : U) -> A -> A -> U`、`refl`、`trustMe`（公理）、`J`（除去子・唯一の原始）。
派生（`transport` など）は `std` 側（[型システム](user/types.md#命題的等価id--と-j)）。

### 有界添字・Maybe

```
Lt : N -> N -> U                              -- 「より小さい」命題
nominal Fin (k : N) = { i : N, lt : Lt i k };
enum Maybe a { some a, none };
fromMaybe : a -> Maybe a -> a                 -- 既定値つき取り出し
```

## prim — 内部の葉演算（Prelude に出ない）

明示 `import prim;` でだけ触れます。オーバーロードされた `+` などの下にある単相演算です。

`addN subN mulN` / `addZ subZ mulZ` / `addR subR mulR divR` /
`ltN ltZ ltR` / `eqN eqZ eqR eqB` / `negZ negR` / `fromNToZ fromNToR fromZToR`

## std — Prelude

`base` を再公開し、オーバーロード層を載せます。常時取り込み。

### クラス（演算子の配線）

`AddOp a b` / `SubOp` / `MulOp` / `DivOp` / `LtOp` / `EqOp`（2 パラメータ・`result : U` 依存）、
`NegOp a`。および変換の `From a b`、小数の中間表現 `Rational = {num : Z, den : N}`。

### インスタンス

- 同種数値演算：`N`/`Z`/`R` の加減乗、`R` の除、`N`/`Z`/`R` の比較、`N`/`Z`/`R`/`B` の等値、
  `Z`/`R` の符号反転
- `From`：恒等（N/Z/R）・塔上げ（N→Z, N→R, Z→R）・`Rational → R`・`Vec k A → Buffer 1 [k] A`

### fixity と演算子

| 優先順位 | 演算子 |
|---|---|
| `infixl 7` | `*` `/` |
| `infixl 6` | `+` `-` |
| `infix 4` | `<` `==` `≡` |

`_+_` `_-_` `_*_` `_/_` `_<_` `_==_`（中置）、`-_` `+_`（前置）、`from`（変換の適用）。

### 論理

`not` `and` `or`（`B` 上。prim ではなく `if` による定義＝短絡）。

### ベクトル連結・配列

| 名前 | 型 |
|---|---|
| `append` | `Vec n T -> Vec m T -> Vec (n + m) T` |
| `concat` | `Vec n (Vec m T) -> Vec (n * m) T` |
| `Array` | `N -> U -> U`（`= λk A. Buffer 1 [k] A`） |
| `at` | `Array k A -> Fin k -> A`（全域・範囲外は実行時エラー） |
| `tabulate` | `(k : N) -> (Fin k -> A) -> Array k A` |

### 実体化

`render`（`Sampling` 指定）/ `rasterize`（セル中心固定）と、その明示 per-sample 版
`render_fallback` / `rasterize_fallback`（[評価モデル](user/evaluation.md#関数とバッファ実体化)）。

### 等価の糖衣と J 派生

`_≡_`（`Id` の中置・`infix 4`）、`transport`、`sym`、`trans`、`ap`（合同律）、
`castVec : (n ≡ m) -> Vec n T -> Vec m T`。

### 型レベル算術の法則（trustMe 製）

`plusComm` `plusAssoc` `plusZeroR` `mulComm` `mulAssoc`。`castVec` と組み合わせて
`Vec (n + m) T` ↔ `Vec (m + n) T` の付け替えなどに使います。

## algebra — 法則つき数学構造

すべて演算を値パラメータに取る nominal レコードです（[クラス](user/classes.md#数学構造algebra--math)）。

`Semigroup m op`（結合律）／ `Monoid m op`（＋単位元）／ `CommMonoid`（＋交換）／
`Group`（＋逆元）／ `CommGroup` ／ `Semiring m add mul`（分配・零）／ `Ring`（＋加法逆元）。

導出関数：`ringAddIsCommGroup`（環の加法群）、
`ringOps`（環 → `{add : AddOp m m, sub, mul, neg}` の演算子配線。`instance open` と併用）。

## math — 数の構造

- `instance NSemiring : Semiring N _+_ _*_`、`instance ZRing : Ring Z _+_ _*_`
  （法則は trustMe）
- `nominal Complex = Vec 2 R` — `cAdd` `cMul` `cNorm`、`instance CRing : Ring Complex cAdd cMul`
- `nominal Quaternion = Vec 4 R` — `qMul` `qInv`、`instance QGroup : Group Quaternion qMul`

**⚙ 実装メモ**：`Complex` の四則を `+` `*` で使うには `CRing` から
`instance open ringOps CRing;` のように配線します（`math` は構造だけを宣言し、
`*Op` への配線は使う側が選ぶ）（要検証：既定でどこまで配線済みか）。

## mesh — トポロジ

[メッシュ](user/mesh.md) 参照。
