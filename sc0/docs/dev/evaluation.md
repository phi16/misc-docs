# NbE（eval.rs）——型検査の評価器

対象ソース：`core/src/eval.rs`＋`core/src/prim.rs`（簡約規則）

## 責務（2026-07 の再編後）

NbE（Normalization by Evaluation）。`eval : &[V] × &Term → V` で値にし、
`quote : level × &Value → Term` で正規形の項へ読み戻します。定義的等価（types.rs）は
「両者を quote して項比較」。

かつては実行時の値計算もこの機構に相乗りしていましたが、**runtime は eval IR
（[dev/ir](dev/ir.md)）に一本化**され、NbE の役割は次の 2 つに絞られました：

1. **型検査**。型の中の計算の正規化・定義的等価・メタ解決。`member_val`（NbE 値）は
   「型の中で使う値」のために残ります。
2. **差分テストの独立参照実装**。NbE の `parallel` は常に per-sample
   （各格子点で f を `apply` する honest-slow な評価）で、kernel IR を一切使いません。
   「IR ≡ NbE」の差分テストが、独立した 2 実装の一致として意味を持つのはこのためです。
   strict / fallback の区別は kernel IR を持つ eval IR 側にだけ存在します。

**quote の共有破壊問題**（λ の下の mandelbrot が 2ⁿ に爆発）はこの分離の動機でした。
型検査で quote する対象は型のサイズなので、実用上問題になりません。

規約：

- `Term` の変数は de Bruijn **添字**、`Neutral` の変数は de Bruijn **レベル**。
  env は `Vec<V>` で `Var(i)` は `env[len-1-i]`。
- 値は **Rc 共有**（`V = Rc<Value>`）。
- readback（quote / quote_sp）は **`Readback` トレイト**で骨格を共有しています
  （かつて 4 本あった重複を 2 本の骨格に）。

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
返す**のが規約です（複合を native が返すと中の値の round-trip が壊れる。合成はライブラリの
let でやる）。eval IR 側の `Node::Foreign` も同じ「(head, args) が同一性」の原則で intern
します。

### Neutral の 2 つの特殊頭

- `NStuck(V)`：簡約しきれない rigid 値（neutral 引数を持つ stuck な VPrim など）を
  neutral の頭として包む。「stuck prim を if/case/proj に晒す」式でも panic せず stuck の
  まま残せる。quote は中身を書き戻す。
- `NMember(module, name)`：簡約規則を持たないモジュールメンバ。**エラー多重化**
  （elaborate に失敗した宣言＝型はあるが値が決まらない stuck メンバ）と、
  **`extern` 宣言**（型検査では opaque な入力）の両方がこれで表されます。
  値を捏造せず、後続宣言の eval を panic させない。

## メタ変数

`METAS`（thread_local の `Vec<MetaEntry>`）。`solve_meta` で解を書き、`force` が
VFlex のスパインを解へ適用して畳む。`zonk` は解を項に書き戻す。
`snapshot_metas` / `restore_metas` は elaboration のトライアルが使う
（**restore は truncate 相当**——trial から返す値は zonk で detach が必要。
[elaboration](dev/elaboration.md) 参照）。

## 評価エラーチャネル

wasm/ライブラリとして panic は許されないので、実行時の失敗は
`set_eval_error` / `take_eval_error`（thread_local・**最初の 1 件が勝つ**）で運びます。
発生源の例：

- 存在しないフィールドの射影・ベクトル添字の範囲外・非関数の適用
- `J` の実行時等式検査の不一致（`trustMe` の嘘。両端が具体値のときだけ検査し、
  neutral の間は stuck に残す——**stuck に残すこと自体が重要**で、さもないと
  transport/sym などの派生定義がビルド時正規化で J を消してしまい検査機会が失われる）
- strict な `parallel` の kernel IR 化失敗（eval IR 側）

## メンバの二重メモ（module.rs）

メンバの実体は `MemberEntry { ty, def, v, ir }`：

- `def` は **メタ無しの Core 項が真実**（`Term`）。opaque / nominal / instance は
  native 値（`Native`）、`extern` は既定値項（`Extern`）。
- そこから **NbE 値 `v`（型検査用）と eval IR `ir`（runtime 用）を対称に遅延導出**して
  それぞれメモします。初回 `member_val` / `member_ir` で埋まる。
- メンバの再定義は entry ごと差し替え＝メモも自然に無効化（side-cache を持たない）。
  incremental の green 再適用は `with_ir` で前回の IR id をそのまま載せて再導出を省く。

旧構造（`MemberVal { Lazy | Forced }` の一本値）はこの二重メモに置き換わっています。
