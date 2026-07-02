# Elaboration（types.rs）

対象ソース：`core/src/types.rs`（約 2700 行）

## 責務

`&Surface → (Term, V)`。双方向型検査（`infer` / `check`）で、糖衣（演算子・注釈・リテラル・
ホール・implicit）をすべて解決し、純粋な Core 項と型値を返します。定義的等価は NbE：
「両者を同じ level で quote して項比較」。宇宙は暫定 `U : U`（type-in-type・健全性は後で階層化）。

型は `Value`（eval.rs）として扱います。elaboration は eval と密結合です——型の中の計算は
評価で正規化され、`check` が record リテラルを検査するときはフィールド型のクロージャを
値適用で開きます。

## Ctx と Scope：可変レジストリを持たない

`Ctx` は env（de Bruijn 環境）・level に加えて `Rc<Scope>` を運びます。`Scope` は
「いま見えている nominal 定義と instance の集合」で、**import 集合から純粋に導出**されます
（`scope_of(imports)`）。可変な working レジストリは存在しません（哲学 §4）。

- `resolution_modules`：Prelude＋明示 import＋その `open import`（再公開）先の**推移閉包**。
  同じモジュールを複数経路で見ても 1 回だけ数える＝**instance の二重計上を構造的に防ぐ**。
- 式内の局所宣言（in-body の `nominal` / `instance`）は `Scope` を**拡張した新スコープ**を
  作って内側に渡すだけ。グローバルは書き換えない。
- 遅延した探索（後述）も**宣言地点で捕捉した scope** で解きます。fixpoint は末尾でまとめて
  回るが、解決自体はレキシカルに決定論的。

## メタ変数と単一化

- メタは `fresh_meta()` で立て、`meta_term` が**スコープ内の束縛変数すべてに適用した形**
  `m x0 … xn` で項に埋めます（解が局所束縛に依存できる）。**let 定義はスキップ**する
  （束縛変数だけ）。
- `solve` は **Miller パターン単一化**：メタのスパインが「相異なる束縛変数への適用」のときだけ
  解く。射影が混じる・引数が変数でない場合は非パターン＝推論不能。
- `unify` の構造：基底型・Π（η 込み）・λ・リテラル・enum（タグ＋payload 構造）・
  prim（頭＋引数）・**nominal は名前＋引数**（構造的 record とは一致しない＝剛体性の実装点）・
  依存 record はテレスコープを fresh neutral で開きながら順に。どれにも当たらなければ
  quote して項比較（正規形比較）にフォールバック。
- **探索辞書メタの特別扱い**：未解決の instance 探索の dict メタを含む単一化
  （`d.result = R` など）は、その場で解かず**その探索の制約として積む**
  （`defer_dict_constraint`）。候補を仮入れしたとき制約もすべて検査する＝
  **出力型方向（result-directed）の解決**がこれで効きます。

## implicit の挿入

- `infer_spine`：適用 `head a1 … an` を平坦化して elaborate。各 explicit 引数の前に
  **先頭に並ぶ implicit Pi をメタ / instance 探索で埋め**、引数を check する。
- `insert_implicit`：ドメインが `VNominal`（クラス）なら instance 解決へ。
  **goal が ground なら即解決、メタを含むなら SEARCHES に遅延登録**（順序非依存：
  オペランドが型を確定してから一意一致で解く）。nominal でなければ fresh メタ。
- check 位置で implicit Pi に非 implicit-λ を当てると implicit λ を自動挿入。
- `analyze_implicit_modes`：各 implicit が「どの explicit 引数 / 結果型から決まるか」を
  **型だけから**解析する純粋関数。rigid pattern 位置＝injective な構成子
  （`->`・`Vec`/`Buffer`・nominal 頭）を辿って到達できる位置。非単射な関数適用や
  record 型越しは「決まらない」（保守的・注釈要求に倒す）。determinate 解決
  （探索せず読むだけで埋める）の土台で、現状は index ルックアップ＋trial のハイブリッド。

## postpone：3 つの遅延 problem リストと fixpoint

elaboration 中に「まだ決められない」ものは thread_local の 3 リストに積まれ、
**1 つの fixpoint**（`solve_problems`）で消化されます。

| リスト | 中身 | 確定条件 |
|---|---|---|
| `SEARCHES` | instance 探索（goal・dict メタ・制約・リテラル情報・NodeId） | 候補が**一意**になったら commit。0 件＝no instance、複数＝ambiguous |
| `CHECKS` | 詰まった引数検査（Ctx・surface 項・期待型・プレースホルダメタ・NodeId） | 他の problem が解いたメタで `check` が通ったらプレースホルダと unify |
| `BORNS` | brace リテラル（VecLit/Record）の born 解決（期待メタ・構造型） | 期待メタが pin されたら nominal → unfold と unify（born-as-X）／それ以外 → 構造と unify |

- `infer_spine` 内では引数が詰まったら**プレースホルダメタ**を置いて型を先に進め、
  スパイン内多パス → 進展が止まったらグローバル CHECKS へ移す。「左から右で決まらないと
  失敗」しないのはこの構造のおかげ。
- fixpoint は「進展が無くなるまで」回る。strict（プログラム末尾の `solve_problems`）では
  残りを確定エラーにするが、その前に **born のフォールバック**を一度だけ試す：
  未 pin の brace リテラルを構造 Vec/Record を既定として確定して再挑戦
  （**数値リテラルは no-default、brace リテラルは構造既定を持つ**という非対称は仕様）。
- `progress_problems`（非厳格）は型注釈の境界などで呼ばれ、解けるものだけ進める
  （型レベルで静的に必要な Vec 長などを早めに確定させる）。

### スナップショットの不変条件（重要）

トライアル（instance 候補の仮入れ・失敗した引数の再検査）は `snapshot_problems` で
**「メタ表＋3 リスト」の全状態を丸ごと撮り、失敗時に丸ごと戻します**。
「どのリストを戻すか」を手で列挙しない構造にしたのは、過去に CHECKS の戻し漏れで
「失敗中に push された problem が truncate 済みメタを抱え、後の `solve_meta` が添字外 panic」
というバグを踏んだためです。問題リストを増やすときは、必ず `ProblemState` にも足すこと。
また、失敗した再検査の副作用を巻き戻さないと「失敗のたびに探索が leak → 次ラウンドで
偽の進展 → fixpoint が止まらない」という形でも壊れます。

## instance 探索の実装

`instance_candidates`：まず **determinate な index ルックアップ**（閉じた ground キー）を試し、
引けなければ **trial**（全候補を仮入れして巻き戻す）へフォールバック。

- 単相候補：goal と unify → 制約（result-directed）を検査 → 通れば候補。
- **parametric instance**（型が Π）：`try_parametric_instance` が
  Π テレスコープを剥がし（ドメインが nominal の束縛＝premise、それ以外＝型/値パラメータの
  メタ）、**終端を goal と unify してから** premise を再帰解決します。
  - 停止性ガード：再帰深さ 32（`Add a a -> Add a a` のような頭が減らない悪性 instance の暴走防止。
    通常は構造減少で自然に止まる）。
  - **返す候補は zonk で detach 済みにする**こと。呼び出し側の trial ループが
    `restore_metas` でメタ表を切り詰めるので、メタ参照が残った値は後で壊れます。
- リテラル（`resolve_lit_conv`）：goal は `From base target`（整数は `N` 基点、小数は
  `Rational = {num, den}` 基点）。候補が一意なら**即 commit**（target が未確定でも出力型に
  pin する——遅延すると文脈メタが escape する事例があった）。多義なら `lit` 情報つきで
  SEARCHES へ遅延し、最後まで残れば「リテラルの型を注釈して」というユーザー向けエラーに
  変換されます（内部表現 `From N ?m` を見せない）。

## nominal の扱い

`NominalDef { param_types, rhs, env }`。単一化は名前＋引数（injective）。
`check` が RHS を unfold するのは **brace リテラルを nominal 期待型に検査するときだけ**
（born-as-X）。RHS が record / Vec に限られる検査は宣言時。`as` は注釈であって coercion では
ないので、既存の値を nominal に「変換」する経路は存在しません。

## 診断・可視化のための記録（すべて opt-in の thread_local）

| 記録 | 内容 | 消費者 |
|---|---|---|
| `TYPE_REC` | 表層 NodeId → (level, 型値)。infer/check が残す | hover・graph の型表示（solve 後に quote） |
| `BINDER_TYPES` | 束縛の絶対 level → 型値。**graph 専用**（fold 入力など NodeId を持たない合成 var 用） | graph |
| `ERR_CHAIN` | 失敗した surface NodeId を**内側→外側**順に積む | analysis が NodeId→span と join し「最も内側の解決できる span」に診断を置く |
| `QUERY_HOLES` | `?` の (level, 期待型, NodeId)。エラーではなく報告 | 診断（severity 低）・Web のホール表示 |

既定はすべて None（無効）＝通常の elaboration は記録コストを払いません。
fixpoint 中はラッパがノイズを積むので、探索失敗の位置は最後に `clear_err_chain` してから
確定的にセットします。エラー多重化（宣言ごと）は modules 側（build_decls の collect モード）
の責務です（[modules-cook](dev/modules-cook.md)）。

## 落とし穴まとめ

- **problem リストの戻し漏れ**→ `ProblemState` 丸ごとスナップショットで構造的に防止（前述）。
- **parametric 候補の detach 忘れ**→ zonk してから返す（前述）。
- **リテラル探索の leak**：失敗した再検査を巻き戻さないと fixpoint が発散する（前述）。
- 型検査そのものに時間を食う場合、Nat リテラル born の From trial（N→N/Z/R 候補）が
  主因になった実測がある（cube で ~45ms）。determinate 化（結果型 index 直引き）が
  将来の整理投資として記録されている。
