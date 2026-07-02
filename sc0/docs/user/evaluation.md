# 評価モデルと実体化

## 基本

- 評価は**正格（strict）**で、現状は純粋です（効果システムは未実装・設計中）。
- 定義的等価は NbE（Normalization by Evaluation）：型検査中に項が正規化されます。
- 一般再帰はありません。反復は**有界反復**プリミティブで書きます：

```
iterate : (n : N) -> (?T : U) -> T -> (T -> T) -> T        -- x, f x, f (f x), …
for     : (n : N) -> (?T : U) -> T -> (N -> T -> T) -> T   -- 添字つき
```

**⚙ 実装メモ**：`iterate` / `for` は回数 `n` が静的ならカーネル内で unroll されます。
動的な `n` は当面 stuck（簡約されず残る）です。座標から回数を作るには
`floorN : R -> N`（負は 0 に飽和）を使います。

## stuck（止まった項）は正規の状態

引数が具体的でないプリミティブ適用は、エラーにならず**止まった項（stuck / neutral）**として
値になります。部分適用も値です。たとえば λ の下の `render` は、関数引数が具体になるまで
stuck のまま運ばれ、適用されたときに初めて実体化が走ります。

> この節の挙動は実行系の改修（elaboration と実行の分離＝汎用 IR 化・
> `design-workspace-tasks.md` §5）で変わる可能性があります（2026-07-02 時点の記述）。

## トップレベル let は遅延・メモ化

**⚙ 実装メモ**：モジュール（＝プログラム）のトップレベル `let` の**値**は、宣言時ではなく
**初回参照時に評価され、メモ化**されます。型検査は宣言順に即時に走りますが、重い実体化は
実際に使われるまで起きません（使われない重い宣言はコストゼロ）。

## 関数とバッファ：実体化

「関数の世界」と「データの世界」が型で分かれています。

| | 関数 | バッファ |
|---|---|---|
| 型 | `Vec d R -> T` など | `Buffer d shape T`（`shape : Vec d N`） |
| 性質 | 遅延・合成できる・まだ計算していない | 実体化済み・サンプル読みできる |

行き来する原始・関数：

```
parallel  : (size : Vec d N) -> (Vec d N -> T) -> Buffer d size T   -- 原始：整数添字格子で一括評価
render    : Sampling -> (spec : {lo : Vec d R, hi : Vec d R, res : Vec d N}) -> (Vec d R -> T) -> Buffer d spec.res T
rasterize : render Sampling.center                                   -- 画像向け固定（セル中心）
sampler   : Interp -> Buffer d shape T -> Vec d R -> T               -- 読み戻し（領域 [0,1)^d）
```

- `render` / `rasterize` は**原始ではなく** `std` の普通の `let` です。`parallel`（整数添字）の上に
  「添字 → 実座標」のアフィン写像を載せただけで、定義はライブラリとして読めます。
- `Sampling = enum { center, edge, closed }` はセル内サンプル位置。
  `Interp = enum { nearest, linear, cubic }` は補間方式です
  （**⚙ 実装メモ**：`cubic` は当面 `linear` と同じ実装）。
- 「画像」「音」という固定型はありません。`Buffer 2 [w,h] (Vec 3 R)` を画像として PNG に書き出し、
  1 次元バッファを音として扱う、という**ビューの解釈**だけがあります。
- `Array k A = Buffer 1 [k] A`。`at`（`Fin k` 添字の全域読み）と `tabulate`（関数を焼く）、
  `from [a, b, c]`（`Vec` からの直書き）があります。

## カーネル IR コンパイル

**⚙ 実装メモ**：`parallel` の関数はサンプルごとにインタプリタで評価されるのではなく、
**型消去されたカーネル IR にコンパイルされて一括実行**されます。

- コンパイルは部分評価を含みます。サンプル添字に依存しない計算（`render` のアフィン係数など）は
  hoist され、定数分岐（`case` の対象が定数）は畳まれます。
- **IR 化に失敗した場合、黙って遅い経路には落ちず実行時エラーになります**（strict）。
  激重のサイレントフォールバックを防ぐためです。
- 遅くてもよいから per-sample で評価したいときは、明示的に
  `parallel_fallback` / `render_fallback` / `rasterize_fallback` を使います。
- どちらの経路で実行されたか・次元・サンプル数・所要時間は Web の Log パネルに出ます。

## 型消去

型・証明は実行時に消去されます。`Vec n T` と `Vec m T` は同じ表現、`J` / `transport` は
実行時恒等です（[型システム](user/types.md#命題的等価id--と-j) の trustMe 実行時検査も参照）。

## 乱数

```
rand : (?n : N) -> Vec n N -> R    -- seed をハッシュして [0, 1) を返す。純粋・決定的
```

効果としての乱数（`Rand`）は設計中で、現状はこの純粋な擬似乱数だけです。
seed は `Vec n N` なので、成分で次元やチャンネルを分けられます
（`rand [i, 0]` / `rand [i, 1]`）。同じ seed は常に同じ値です。

## 実行時エラー

失敗は honest に報告されます（panic や値の捏造はしない）：

- `at` の範囲外アクセス（`Fin` の信頼供給が破られた場合）
- `J` の実行時等式検査の不一致（`trustMe` の嘘）
- IR 化失敗（strict な `parallel` / `render` / `rasterize`）
- 非関数の適用・存在しないフィールドの射影など（評価エラーとして報告）
