# Surface / Core 分割と表示・グラフ

対象ソース：`core/src/surface.rs`・`core/src/ast.rs`・`core/src/graph.rs`・`core/src/workspace.rs`

## 責務と全体像

項の表現は**2 つの型**に分かれています。

| 型 | 役割 | 持つもの | 持たないもの |
|---|---|---|---|
| `Surface`（surface.rs） | parser の出力。IDE・graph・pretty の対象 | 全ノードに `id: NodeId` と `span`、糖衣・装飾 | 型情報・メタ |
| `Term`（ast.rs） | Core＝正準な意味実体。eval / NbE の対象 | 意味に効く構造だけ | 位置・表示専用 variant |

elaboration（types.rs）が `&Surface → (Term, V)` の唯一の橋です。表示（pretty）と graph は
**Core を経由せず Surface を直接歩きます**——`lower : &Surface → Term` という素朴な脱糖も
ありますが、これは**表示ブリッジではなく**、唯一の用途は prim の型定義文字列
（`"N -> U -> U"` などホール無し・糖衣なしの綺麗な型式）のパースです。ホールに当たると
panic します（メタ無しでは表せないため・意図的）。

この分割が導入される前は、Term に表示専用 variant が同居し、ポインタ照合・脱糖時のクローン・
網羅性の手作業・消費者ごとの食い違いというバグのクラスがありました。型を分けたことで
これらが**構造的に**消えています（span の取りこぼしも「全ノードが id/span を持つ struct」に
したことで構築時に不可能になった）。

## CommonF：共通部を 1 variant に束ねる

surface と core で形が同じ約 20 variant（Var/U/基底型/Pi/Lam/App/Let/If/Prim/Member/
VecLit/RecordTy/Record/Proj/EnumTy/Case と宣言 Nominal/EnumDecl/Instance）は
`CommonF<R>` としてノード型 `R` で抽象されています：

```rust
Surface … SurfaceKind::Common(CommonF<Surface>) ＋ surface 専用 variant
Term    … Term::Common(CommonF<Term>)           ＋ core 専用 variant
```

**実装選択**：Haskell の Trees That Grow のように型引数で variant を増減することは Rust では
できないので、「共通部を `Common` という **1 つの variant** に束ね、phase 固有 variant を
各 enum に足す」形にしています。`Box` は再帰位置（`Pi(Box<R>, Box<R>)`）にだけ付け、
`Vec<R>` は素のまま（Vec 自身が間接化なので二重 Box を避ける）。

phase 固有 variant：

- **Surface 専用**（elaboration が解決し core から消える）：
  `Infix`/`Prefix`（演算子。解決済み `head` を持ち、適用に脱糖される——クローンではなく
  App を組むだけ）、`Ann`（`as` 注釈。core に注釈は残らない）、`Lit`（数値リテラル。
  変換未解決）、`Hole`（`?`/`_`）、`Fold`（`@`）、`Deco`、`Fixity`。
- **Core 専用**（値の読み戻し・内部形）：`Lit(NumBase, Option<conv>)`（elaboration が解決した
  `From` 変換を抱えたリテラル）、`NatLit`/`IntLit`/`RealLit`（quote の出力）、`Meta`
  （elaboration 中のみ・zonk 後の core には残らない）、`BufLit`（バッファ値・構文なし）、
  `Inj`（enum の variant injection 内部形）、`NominalCon`（nominal コンストラクタ参照）。

変数は **de Bruijn 添字**です（NbE と相性が良い）。`shift` の cutoff 規則（Lam/Let/Pi の
body で +1、param 列は順次、case arm は payload 数、enum body は型名＋constructor 数、
instance は名前付きのみ +1）が唯一の束縛規則で、`scope_at`（後述）もこれと一致させています。
**ここがずれると添字がすべて壊れる**ので、束縛を増やす変更ではこの 2 箇所＋ast::shift を
必ず揃えること。

## Deco：意味に効かないが意図に効く層

`Deco` は「気持ちを込めた書き方」を round-trip で保存する透明ラッパです（哲学 §3：
正規化はデフォルト、ただし書き手の意図に効く選択は透明な層として残す）。

| Deco | 保存するもの |
|---|---|
| `Group(N)` | 先頭 N 個の Π 束縛が 1 つの括弧グループ `(a b : T)` で書かれたこと |
| `Semi` | let が `in` でなく `;`（宣言列）で区切られたこと |
| `Block { multiline, bare }` | ブロック `{…}` として書かれたこと・複数行か・ブレース無しのトップレベルか |

`Fold`（`@`）も同類です。**eval・型検査はすべて see-through**（elaboration の check は
Fold/Deco/Fixity の内側へ期待型をそのまま渡す＝ブロックで包んだリテラルにも born-as が届く）。
一方 glyph（`->`/`→`・`λ`/`\`）は per-occurrence 保存**せず**正規化し、正規化先だけを
`PrettyConfig`（thread_local）で選べます。

## pretty：Surface / Core で 1 つの整形ロジック

整形は `PrettyNode` trait（ast.rs）で抽象され、`Surface` と `Term` の両方に実装されています。
共通部の整形（`go_common`）は 1 回だけ書かれ、phase 固有 variant は各実装が処理します。
「Core は正準・Surface は位置つきビュー、だが整形ロジックは一つ」という哲学 §3 の現れです。

- **fixity は暗黙のグローバル状態を持ちません**。parser はインスタンス状態、pretty は引数として
  持ち回り、常に「その項のスコープ（import 集合＋囲みの fixity 宣言）」から決定論的に決まります。
  **pretty と再パースが同じ環境を使う**（`module_fixities`）ことが往復一致の前提です。
- 優先順位 `Prec` は**有理数**です（整数だと既存の 6 と 7 の間に新しい強度を挿せない）。
  未宣言演算子のオペランドには `FORCE_PARENS` で必ず括弧を付けます。
- 小数リテラルは den が 10 の冪なら小数点位置で厳密復元します（`2.5` と書いたものが
  `2.5` のまま戻る。浮動小数点の近似表示にしない）。

## 構造編集：NodeId によるサブツリー差し替え

graph からの編集は「graph 全体から Surface を再構成」**ではなく**、
「Surface 木の局所差し替え」です（真実は Surface・graph はビュー）。

- `find_node` / `replace_at`：NodeId で指したサブツリーを差し替え。残りは構造共有。
- `scope_at`：対象位置の束縛スコープ（外→内の名前列）を返す。新しいテキストは
  このスコープでパースする（named → de Bruijn）ので、**同じスコープ深さでパースされた項は
  再添字なしでそのまま埋められる**——これが編集の中心的な不変条件です。
- `delete_node` / `insert_vec_elem`：リスト要素（VecLit 要素・record フィールド・case arm）の
  挿入・削除。**必須項の「削除」はホール化**（`replace_at` で `?` に）と使い分けます。
  enum variant の payload 型列は「型の削除か variant の削除か」が曖昧なので対象外（意図的）。
- 編集結果は最終的に pretty → 再パースされるので、挿入した部分木の NodeId が既存と
  衝突しても無害です（次のパースで全部振り直される）。

## graph.rs：1 宣言の式グラフ（Surface のビュー）

`to_graph` は Surface を**全展開**したノードグラフにします。モデル：

- `kind = code / lambda / let / var / decl`。包含（`parent`）と結線（`in`/`body`/`value`）は
  **別物**です。lambda / let は「境界」で、連続する λ は 1 ノードに畳まれます（多引数＝binder 配列）。
- `var` は de Bruijn 参照のノード化で、同じ束縛は領域内で共有されます（1 ノードを複数の `in` が指す）。
- body にインラインで残るのは prim（演算子＝操作の正体）だけ。リテラル・型なども独立ノードです。
- 各ノードは `src: NodeId` で**元の Surface サブツリーに対応**します（構造編集の土台）。
  合成ノード（fold の自由変数入力など）は `src: None`。
- `@`（Fold）は「自由変数だけを `in` に出し、本体は閉じたテキスト」という 1 ノードになります。
  nominal / instance / fixity 宣言も同じ仕組みでノード化されます（instance は ty/val を
  子ノードに展開する形態もある）。
- per-decl グラフは `self_module` / `declared` / `fixities` を運びます——body 断片を再パース
  するとき、兄弟宣言への参照を `Member(self, name)` に解決し、ローカル演算子を unbound に
  しないためです。`member_ref`（兄弟参照）・`output_sink`（宣言の出力）・`hole` は
  表示用の読み取り専用フラグで、**往復（from_graph）では無視されます**。

## workspace.rs：宣言の依存 DAG

Workspace は graph とは別のビューです（graph＝1 式の木、Workspace＝モジュールの宣言 DAG）。

- `WorkspaceNode` ＝ トップレベル宣言 1 つ（name / kind / deps / anno / has_hole）。
- 依存辺は本体の `Member` 参照から**導出**します（`modules::forest_deps`）。辺を別データとして
  持ちません（真実は一つ）。
- `#[node]` 注釈は**生テキスト**のまま運びます。core は解釈せず、JS 側が JSON として読みます
  （透明層。core に view の知識を持ち込まない）。

## 落とし穴（実際に踏んだもの）

- **span の取りこぼし**：旧構造では `bx`（span 付与）忘れが頻発した。現構造では
  「span の無いノードが作れない」ので再発しない。合成ノード（graph の `sk`）は id=0・span=(0,0)
  で、これは「source 無し」の意味として扱う。
- **shift が id/span を複製する**：グループ束縛の脱糖で共有ドメインを shift すると id が
  重複するが、解析事実のキーに使う前に再パースされる経路では無害。id をキーにする処理を
  増やすときは複製の可能性を意識すること。
- **graph の読み取り専用フィールド**（type/src/member_ref/output_sink/hole/decl_ty）を
  from_graph が読むように変えないこと。ビューの都合を実体に逆流させない。
