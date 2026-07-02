# 評価（eval.rs）

対象ソース：`core/src/eval.rs`（約 1300 行）＋ `core/src/prim.rs`（簡約規則）

## 責務

NbE（Normalization by Evaluation）。`eval : &[V] × &Term → V` で値にし、
`quote : level × &Value → Term` で正規形の項へ読み戻します。定義的等価（types.rs）は
「両者を quote して項比較」。**型検査の正規化と実行時評価が同じ機構に相乗り**しています
（→ この相乗りを解消するのが進行中の IR runtime 化・末尾参照）。

規約：

- `Term` の変数は de Bruijn **添字**、`Neutral` の変数は de Bruijn **レベル**。
  env は `Vec<V>` で `Var(i)` は `env[len-1-i]`。
- 値は **Rc 共有**（`V = Rc<Value>`）。変数参照は束縛値の Rc を refcount だけ増やして返すので、
  `let x = e in x + x` のような共有は**構造共有された DAG** になり deep clone で 2ⁿ に
  膨らまない。座標を抽象にした部分評価（kernel.rs）はこの共有に依存しています。

## Value の設計

| variant | 要点 |
|---|---|
| `VPi` / `VLam` | cod / body は `Closure`（env＋Term）。`Icit` は VLam では計算に無関係（往復表示用） |
| `VFlex(m, spine)` | 未解決メタ＋消去子スパイン（`Elim` = App/Proj/If/Case）。解けたら `force` が畳む。**elaboration 後には残らない** |
| `VPrim(p, args)` | プリミティブの部分適用。完全適用でも簡約できなければ正準値 / stuck としてこの形のまま |
| `VNominal(name, args)` | **β せず引数を貯める**。頭が nominal のまま残る＝instance 探索が頭を見られる（剛体性と探索の実装が同じ表現で成立） |
| `VRecordTy(RecTel)` | 依存 record 型のテレスコープ。フィールド型は「env＋先行フィールド値」で eval する項 |
| `VEnumTy` / `VTag` | enum 型と variant 値（payload つき） |
| `VForeign { head, args, data }` | native opaque 値（mesh の `Topo` 等）の**汎用 carrier**。専用 variant を作らない（哲学 §5） |
| `VNeutral(n)` | 止まった計算 |

### VForeign の型

`head + args` は「その値を生んだ prim 適用」で、**quote / 型の同一性はこの構築の同一性で
判定**します（`gridTopo 2 3` から生まれた Topo は quote すると `gridTopo 2 3` に戻る＝
正準形がそのまま往復する）。`data` は consumer だけが触る native 実体で、`Foreign` trait の
`type_name` で downcast 前に取り違えを honest に弾きます。**ネイティブ関数は裸の値だけを
返す**のが規約です（`some Topo` のような複合を native が返すと中の Topo の round-trip が
壊れる。合成はライブラリ側の let でやる）。

### Neutral の 2 つの特殊頭

- `NStuck(V)`：簡約しきれない rigid 値（neutral 引数を持つ stuck な VPrim など）を
  neutral の頭として包む。「stuck prim を if/case/proj に晒す」式でも panic せず stuck の
  まま残せる（抽象座標を流す部分評価で必須）。quote は中身を書き戻す。
- `NMember(module, name)`：簡約規則を持たないモジュールメンバ。**エラー多重化**で
  「elaborate に失敗した宣言＝型はあるが値が決まらない stuck メンバ」を表すのに使う。
  値を捏造せず、後続宣言の eval を panic させない。

## メタ変数

`METAS`（thread_local の `Vec<MetaEntry>`）。`solve_meta` で解を書き、`force` が
VFlex のスパインを解へ適用して畳む。`zonk` は解を項に書き戻す。
`snapshot_metas` / `restore_metas` は elaboration のトライアルが使う
（**restore は truncate 相当**——だから trial から返す値は zonk で detach が必要。
[elaboration](dev/elaboration.md) 参照）。

## 評価エラーチャネル

wasm/ライブラリとして panic は許されないので、実行時の失敗は
`set_eval_error` / `take_eval_error`（thread_local・**最初の 1 件が勝つ**）で運びます。
`eval_program` が末尾で拾って `Err` にします。発生源の例：

- 存在しないフィールドの射影・ベクトル添字の範囲外・非関数の適用
- `J` の実行時等式検査の不一致（`trustMe` の嘘。両端が具体値のときだけ検査し、
  neutral の間は stuck に残す——**stuck に残すこと自体が重要**で、さもないと
  transport/sym などの派生定義がビルド時正規化で J を消してしまい検査機会が失われる）
- strict な `parallel` の IR 化失敗

エラー値を返して継続する経路（`Option`/`Result`）とこのチャネルの使い分け：
簡約器の深部（`Value → Value` で失敗を型に載せられない場所）だけがチャネルを使います。

## 遅延メンバ

モジュールメンバの値は `MemberVal { Lazy(Term) | Forced(V) }`（module.rs）で、
**初回参照時に eval してメモ化**します。表（Module）レベルの遅延であって `Value` enum は
無傷です。動機は実測：build_decls の eager eval では render が未解決メタを含む段階で
評価されて IR 落ち → per-sample 空走＋再 eval の二度手間（Mandelbrot 7.3s→69ms の改善）。

## 流動的な部分：IR runtime 化（進行中）

**この層は近く大きく変わる予定です**（`design-workspace-tasks.md` §5・方針決定済み）。

- 現状の問題：NbE の `quote` は**共有を潰す**。λ の下の重い計算（mandelbrot under λ）を
  quote すると DAG が木に展開されて 2ⁿ に爆発し、worker が死ぬ（実測）。また render の
  「IR 化失敗 → fallback」も同根。
- 方針：**elaboration は型検査専用**に絞り、実行は record / enum / closure / nominal / Topo を
  含む**汎用 IR に一本化**。NbE 相乗りの実行時評価を撤去する。
- ドキュメントとしては、本ページの「Value が実行時表現を兼ねる」記述は改修後に
  「Value は型検査の正規形専用」へ書き換えることになる想定です。
