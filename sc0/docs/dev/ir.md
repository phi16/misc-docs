# eval IR（ir.rs）——runtime の値モデル

対象ソース：`core/src/ir.rs`＋`core/src/ir/tests.rs`（単体・差分テスト）

## 用語（このコードベースには IR が 2 つある）

| | eval IR（ir.rs `Node`） | kernel IR（kernel.rs `Kernel`/`Op`） |
|---|---|---|
| 役割 | **runtime の値・計算そのもの**（member_ir が持つ） | render の内側を座標格子で回す数値核 |
| 扱う値 | 任意データ型（Vec/Record/Tag/Buf/Foreign）＋**高階（Closure）**＋外部入力 | f64 スカラ・Bool・タグ・静的長 f64 ベクトルのみ |
| 関係 | render の座標残差を `kernel::compile` で kernel IR へ **lower** する（高速路） | eval IR の数値フラグメントの lowering ターゲット |

両者は似た構造（hash-cons のアリーナ DAG）ですが**別物**です。eval IR は kernel IR の
一般化ではなく、kernel IR は eval IR の平坦化ターゲット——ノード集合は一致しません。

## なぜ eval IR か（分離の動機）

NbE の `quote`（Value → Term）は**共有 DAG を木に展開**します。`λ(center). mandelbrot …`
のように λ の下に重い計算があると、`z*z` の連鎖が 2ⁿ に爆発して worker が死ぬ（実測）。
そこで **elaboration（NbE：`eval`/`quote`/`Value`）は型検査専用に残し、runtime の値計算は
共有を保つ eval IR に一本化**しました。runtime から quote 経路が消えたので、
この種の爆発は構造的に起きません。

## hash-cons と id 等価

- ノードは thread_local の**単一アリーナに intern** され、同じ構造は同じ `Id`（u32）に
  なります。id は評価履歴に依存しない**正準な同一性**です（哲学 §3）。
- 等価判定は **id 比較 O(1)**。Cook の early-cutoff（変化が下流に伝播するか）が
  「前回の IR id と一致するか」だけで厳密かつ安価に決まります。
  旧実装（正規形の項を quote して比較）より速く、しかも共有を壊しません。
- f64 はビット（u64）で持ち NaN 込みで intern できるように。`Foreign` の
  `Rc<dyn Foreign>` は Hash/Eq を持てないので side-table に逃がし、intern は
  `(head, args)` だけで行います（「同じ構築＝同じ値」の原則そのまま）。

## Node の全体像

- **入力の葉（3 種・名前空間が別）**
  - `Input(i)` — クロージャ code の仮引数・env スロット。`run_code` の入力置換の対象。
  - `Coord(k)` — **カーネルの実行時入力スロット**。render の f を symbolic に適用して
    残差を作るときの葉（座標軸／ループ累算器・添字／捕捉入力）。入力置換では**素通り**し、
    kernel IR では `Var(k)` に落ちる。同じスロットについて symbolic なので、
    **入力値が変わってもカーネル構造は不変**＝hash-cons がヒット＝再コンパイル無し。
  - `Extern(k)` — **外部 runtime 入力**（`extern` 宣言）。ソースに値が無く host が供給する
    大域入力。closure に束縛・捕捉されず自由に残り、render の capture 規則が
    Capture スロットとして拾う。**値は member_ir に入らない**＝値が変わっても
    member_ir 不変＝Cook の再ビルドが起きない。表示時に `apply_externs` で現在値へ置換。
- `Erased` — 型消去された値。型は runtime で分岐しない phantom なので、型引数の位置には
  これを置く（型消去の IR 上の姿）。
- **データ**：`NatLit`/`IntLit`/`RealLit`/`BoolLit`・`Vec`・`Record`・`Tag`（enum variant）・
  `Buf`（render 結果）・`Foreign`（native opaque）。
- **計算（残差）**：`Prim`（飽和/部分適用。具体なら簡約・記号を含めば残る）・`Proj`・
  `Case`（arm はクロージャ）・`If`・`App`。
- **高階**：`Closure(code, env)`——closure conversion 済みの形。`code` は閉じた本体
  （別テーブル `CodeId`）、`env` は捕捉値の Record。**捕捉は使用される自由変数だけ**
  （capture-only-used＝compile してから prune）。
- `KernelRender { kernel, shape, captures }` — コンパイル済み render（後述）。

## 主要な操作

- `compile(env, term) → Id`：Core Term → eval IR。member のビルド時（`member_ir`）に走る。
  `parallel` に出会うと f を `Coord` に symbolic 適用して残差を作り、
  `kernel::compile` で lower を試みる。
- `run_node(id, inputs)`：クロージャ適用などの実行。`Input` を置換しながら DAG を歩く
  （memo つき）。`Coord` は素通り。
- `apply_externs(id, vals)`：`Extern` の葉だけを現在値へ差し替える（表示経路）。
  KernelRender の captures が具体化されればここで格子が回って `Buf` になる。

## KernelRender（層 B・render-under-λ の解決）

`view = λc. render …` のような「入力に依存する render」は、**定義時に 1 回だけ**
カーネルへコンパイルされ、`KernelRender { kernel, shape, captures }` ノードになります。

- `kernel` は kernel.rs 側の side-table ハンドル。compile はこの 1 回きり。
- `captures` は「カーネルが `Var(d+i)` スロットで読む捕捉入力式」（外側 λ の Input や
  Extern を含むスカラ式）。**カーネル本体は captures について symbolic** なので、
  適用や extern 供給で captures の**値**だけが変わってもカーネルは再利用されます——
  slider を動かしても再コンパイルが起きないのはこの構造です。
- captures がすべて具体スカラに揃った時点で格子を回して `Buf` になります（`run_node` /
  `apply_externs`）。

## テストの構え

`ir/tests.rs`（760 行）に単体テストと**差分テスト（IR ≡ NbE）**があります。NbE 側の
`parallel` は常に per-sample の honest-slow な参照実装として残してある（[evaluation](dev/evaluation.md)）
ので、「同じプログラムを両実行系で走らせて一致」が机上でなくテストとして回ります。

## 落とし穴

- `Input` / `Coord` / `Extern` は**別名前空間**です。置換系の操作（run_code の入力置換・
  apply_externs）がどれを触りどれを素通りするかを混ぜると、カーネルの symbolic 性
  （再コンパイル無し）が静かに壊れます。
- `Foreign` の実体は side-table にあるので、アリーナの intern だけ見て「値が全部ある」と
  思わないこと。
