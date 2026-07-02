# カーネル IR（kernel.rs）

対象ソース：`core/src/kernel.rs`（約 1050 行）＋ `prim.rs` の `red_parallel` 経路

## 責務と全体の流れ

`parallel size f` の `f : Vec d N → T` を、サンプルごとにインタプリタで評価する代わりに
**型なしの数値 IR** にコンパイルして N 回回します（design.md §4）。流れ：

1. **部分評価は NbE がやる**：`f` を**抽象座標**（成分ごとの neutral 変数）に apply して
   quote すると、座標に依存しない部分（型・辞書射影・定数・静的 `map`/`iterate` の unroll）は
   評価で畳まれ、**座標に関する純粋な残差項**だけが残る。specialization のための専用機構は
   なく、「inline の結果として創発」する。
2. その残差を `Lower` が IR へ落とす。**落とせない演算に出会ったら全体が失敗**します
   （`None` を返す・部分成功はない）。
3. strict な `parallel` は None を**実行時エラー**にする。`parallel_fallback` だけが
   per-sample の `apply` 評価に落ちる（黙る silent fallback は撤去済み）。

## IR の設計（実装選択）

- **ハッシュコンスした単一アリーナの DAG**（`Vec<Node>`）。quote が共通部分式を複製しても
  intern で畳まれ、各ノードは 1 サンプルにつき一度だけ評価される（CSE）。
- **子は必ず自分より小さい index** を指す（intern 順）ので、実行は index 昇順の
  1 パス線形。再帰なし。
- **分岐は eager**（Select/Case は両枝とも計算）。木の遅延性は失うが CSE で重複が消え、
  GPU の predication モデルに近い形を先取りしている。
- 値ドメインは **f64 スカラ / Bool / タグ（小整数 id）/ 静的長ベクトル**。
  f64 定数はビット（u64）で持つ（hash-cons のため。NaN の同一視も bit 比較）。
- `Un`/`Bin`/`Tern` は演算を**関数ポインタ**で持つ。dedup は 1 回のカーネルビルド内の
  比較だけなので関数アドレス比較で正しい（lint は抑制済み・理由コメントあり）。
- `sampler` は**バッファを定数テクスチャとしてカーネルに閉じ込め**（`BufConst`）、
  座標依存の添字で gather する（`Sample`＝GPU の texture fetch 対応・nearest/linear）。
  `at`/mesh の接続子は整数添字の `Gather`（範囲外はクランプ）。
- 動的回数の `iterate`/`for` は `Loop` ノード＋別アリーナの `LoopDef`（サブカーネル）。
  累算器は T のスカラ成分列。静的回数は NbE 段階で unroll されるので IR に Loop は出ない。
- `Runner` はレジスタ列（スカラ/Bool/タグ/ベクトル別）を**カーネルごとに 1 回確保して
  全サンプルで使い回す**。ループのスクラッチもネスト込みで事前確保。
  per-sample allocation を出さないことが速度の前提。

## 落とし穴（実際に踏んだもの）

**Rc アドレス再利用による誤 CSE（テストの稀フレークの真因）**：
lowering のメモは `Rc::as_ptr(v)` をキーにした「値の同一性による CSE」だが、lowering 中に
作る一時値（case の arm 評価など）が drop されると、**次の一時値が同じアドレスに再割り当て
されて別の値が同じキーを引く**ことがある。修正は `Lower.keep` にキーの Rc を握って
**コンパイル中はすべて生かす**こと（アドレス再利用が起きない）。メモをポインタキーにする
最適化を書くときは常にこの罠を思い出すこと。

**λ 境界**：`parallel` の f が非具体（λ の下・未適用）の間、`red_parallel` は stuck に残る。
「λ の下の render」が動くのはこの stuck 化のおかげで、IR コンパイルは適用時に初めて走る。

## 計測

実行経路（IR / per-sample fallback）・次元・サンプル数・所要時間は `log.rs` 経由で
Web の Log パネルに出ます。IR 化の成否を疑うときはまずここを見る。

## 流動的な部分

- GPU / Worklet バックエンドは設計のみ（design.md §4.5）。現 IR は predication・texture 対応
  など GPU を意識した形になっている。
- **汎用 IR runtime 化**（`design-workspace-tasks.md` §5）が進むと、この「render 専用の
  数値 IR」は任意データ型（record/enum/closure/nominal/Topo）を含む実行 IR に拡張ないし
  置き換えられる見込み。「IR 化失敗」という失敗モード自体を無くすのが狙い。
- render の **IR キャッシュ**（scalar を実行時入力にして、slider 操作で再コンパイルすら
  しない最適化＝Cook B3）が次の投資として記録されている。
