# 効果と線形型——下調べ

SC0 の効果システム（乱数・シミュレーション・I/O）は未設計です（design.md D11）。
この文書はその下調べで、世の中の効果の扱いを 2 つの系譜に分けて眺めます。
先に立場を書いておくと、SC0 は**線形型（linear types）を根本に置き、効果的なものは
そこから導かれる**という方向を仮説にしています——いわゆる effect system を
そのまま導入する予定はありません。

## 系譜 1：効果を「注釈」で追う

関数の型に「何をやらかしうるか」のラベルを付ける系譜です。

- **モナド（Haskell）**：`IO a`・`State s a`。効果の合成はモナド変換子で——そして
  変換子の積み重ねと持ち上げ（lift）の煩雑さが、この 30 年の不満の中心でした。
- **algebraic effects & handlers**：効果を「操作の集合」として宣言し、ハンドラが
  意味を与える。**Koka**（effect rows：型に効果の行が載る・row 多相）、**Eff**・
  **Frank**（研究言語）、**Unison** の abilities、**OCaml 5**（ランタイムに
  ハンドラが入ったが**型には現れない**、という割り切り）。継続を capture する
  強力さと、その強力さゆえの意味論の重さが両面あります。

この系譜の共通点は、効果が**型の注釈として**存在することです。値の流れとは別の
チャンネル（行・能力集合）が型に増える。

## 系譜 2：効果を「資源」で追う

もう一つの系譜は substructural 型——「値の使用回数」を型が管理する世界です。

- **線形論理**（Girard, 1987）が根。「ちょうど 1 回使う」を型にする。
- **Clean の uniqueness types**：歴史的に重要な実例です。Clean は I/O を
  **一意な `World` 値のスレッディング**でやりました——`World` を受け取り、
  使い、新しい `World` を返す。Haskell が IO モナドを選んだのと同じ問題への、
  「**効果＝線形資源の受け渡し**」という別解。まさに「線形型から効果が出てくる」の
  原型です。
- **Rust**：所有権と借用（affine＝高々 1 回）。ミューテーションという効果を
  「別名参照の不在」という資源規律に還元して、注釈なしで安全にした。
  この 10 年でもっとも成功した substructural 型の実用化。
- **Futhark の uniqueness**：純粋な意味論のまま in-place 更新を許す
  （[配列言語の読み物](relatives-arrays.md)）。
- **Linear Haskell**：矢印に多重度（`a %1 -> b`）。既存言語への後付けの難しさの記録
  としても興味深い。
- **Idris 2 / QTT（Quantitative Type Theory）**：**依存型と線形性の統合**という、
  SC0 にとって本命の理論です。束縛に量 `0 / 1 / ω` が付く：`1`＝ちょうど 1 回、
  `ω`＝自由、そして **`0`＝実行時には存在しない（型の中でだけ使える）**。
  この `0` が「線形な値を型が言及してよいか」という難問への答えで、同時に
  **消去（erasure）の理論**にもなっている——SC0 の「型は実行時に消える」を
  形式化した形とも読めます。
- **Granule**：使用回数を一般の代数（grade）に拡張した研究言語。

## なぜ SC0 は資源側なのか

線形資源として効果を持つと、SC0 の既存の原則とまっすぐ繋がります。

- **効果が値の流れとして見える**。乱数の状態・シミュレーションの状態・出力先は
  「ちょうど 1 回使われる値」として関数を通り抜ける。型に別チャンネルを増やさず、
  データフローそのものが効果の記録になる。
- **グラフに描ける**。ノードグラフの観点（[string diagram の補遺](nodes-landscape.md#補遺string-diagram-という見方)）
  では、環境のような**可換**なものは順序なしに線を引けるのでした。逆に言えば、
  **順序が意味を持つ効果は「1 本の線形な線」としてグラフに現れる**べきで、
  線形資源のスレッディングはまさにその線です。効果の順序が配線として見える——
  ビジュアル環境と線形型は、実は相性が良い。
- **再現性**。資源の生成点（seed の固定・シミュレーションの初期状態）が明示の境界に
  なる（哲学 §7）。ハンドラの継続 capture のような大きな意味論を持ち込まずに済む。
- **カーネルとの整合**。実体化の中で使える効果（乱数・累算）は、線形資源なら
  依存が明示なので並列化の判定が型から読める（Dex が `Accum` 効果で苦労している
  問題への別角度）。

## 開いた問い

正直に難所も並べておきます。

- **書き味**。素朴なスレッディングは全部の関数に資源引数が増えて重い。Rust は借用で、
  Koka は evidence 渡しで、Clean は構文糖で緩和した。SC0 では implicit・グラフ表示との
  組み合わせでどこまで軽くできるか（可換な資源は Reader 的に線を省けるはず、という
  住み分けも含めて）。
- **依存型との統合の細部**。QTT の `0/1/ω` をそのまま採るのか、SC0 の型消去・
  停止性（一般再帰なし）とどう組むか。
- **既存の実体化との関係**。`rand` は今は純粋なハッシュ（効果なし）。効果導入後に
  「効果としての乱数」と「決定的ハッシュ」の両方をどう位置づけるか。

いずれ設計するときは、この文書の系譜 2——特に Clean の World・Futhark の uniqueness・
QTT——が出発点になります。

## 参考リンク

- [Koka — effect rows](https://koka-lang.github.io/) /
  [OCaml — Effect handlers](https://ocaml.org/manual/effects.html)
- [Linear Haskell（POPL 2018）](https://arxiv.org/abs/1710.09756)
- [The Syntax and Semantics of Quantitative Type Theory（Atkey）](https://bentnib.org/quantitative-type-theory.html) /
  [Idris 2: Quantitative Type Theory in Practice（Brady）](https://arxiv.org/abs/2104.00480)
- [Granule](https://granule-project.github.io/)
- [Clean — uniqueness typing](https://clean.cs.ru.nl/)
- [Futhark — Uniqueness Types and In-Place Updates](https://futhark-lang.org/blog/2022-06-13-uniqueness-types.html)
