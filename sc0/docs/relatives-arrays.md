# 近縁種：Dex・Futhark・Halide——配列とカーネルの言語たち

SC0 に技術的にもっとも近い親戚は、グラフィックツールではなく**配列言語とカーネル DSL**
の世界にいます。この文書は代表的な 3 つ——Dex・Futhark・Halide——を観察し、
SC0 と何を共有し、何が違うかを整理します（2026 年時点の観察）。

## Dex — 配列を「添字からの関数」として型付ける

Dex は Google Research の研究言語（Haskell/ML 系・純粋関数型）で、核となるアイデアは
**型付き添字集合（typed index sets）**です。

- 配列の型は `n => a`——添字集合 `n` から `a` への「表」。**配列とは関数である**という
  型付けを言語の中心に据えています。
- 配列は `for i. e` という**添字内包**で作ります：添字集合の各要素で本体を評価して
  配列を構成する。SC0 の `tabulate`（`(Fin k -> A) -> Array k A`）や原始 `parallel`
  （整数添字格子で評価して Buffer に）と、発想がそのまま重なります。
- 添字が型付けされているので、**範囲外アクセスが型で消えます**——SC0 が
  `Fin k`（有界添字）でやっていることと同じです。

違いは目的と重心です。Dex の主目的は機械学習研究のための**自動微分**
（並列性を保つ autodiff・ICFP 2021）で、`Accum` や `State` といった効果も持ちます。
JAX の系譜にある「NumPy 的な暗黙の形状規約を、型のある言語でやり直す」試みと
言えます。編集環境やグラフ表現は持たず、研究プロジェクトとしての性格が強い
（活発な開発期は過ぎています）。

**SC0 から見ると**：`Fin`・`Array`・`tabulate`・`parallel` という自分の部品構成が、
独立に同じ形へ到達した先行例です。「配列の正しい表現は添字からの関数」という結論が
複数の場所で再発見されている、という事実自体が心強い。

## Futhark — サイズ型と uniqueness、GPU で本気を出す純粋言語

Futhark（コペンハーゲン大学・活発に開発中）は GPU 向けの純粋関数型配列言語です。

- `map` / `reduce` / `scan` / `filter` という二階の配列コンビネータが基本語彙。
  コンパイラが flattening などの本格的な変換で GPU コードを生成します。
- **サイズ型**：配列型が長さ変数を持ちます（`[n]f32`）。関数の型で「同じ長さ」を
  要求できる——SC0 の `Vec n T` と同じ役割です。ただし Futhark のそれは**サイズ専用の
  軽量機構**で、SC0 は一般の依存型の一例としてサイズを扱う、という違いがあります。
- **uniqueness types による in-place 更新**：これが Futhark のいちばん面白い部分です。
  「この配列への参照は呼び出し後にもう使われない」を型（一意性）で保証し、
  純粋な意味論を保ったまま**コピーなしの破壊的更新**を許す。Clean 由来の機構で、
  linear type の親戚です。「純粋性と効率の両立を、型のある所有権で解く」という
  実証例として、SC0 が将来 linear type を考えるときの直接の参考になります。

**SC0 から見ると**：GPU バックエンドを本気でやるとコンパイラに何が要るか
（flattening・メモリ管理・サイズの静的追跡）の最良の教材です。同時に、
uniqueness/linear の方向で「効果らしきもの」が型から出てくる実例でもあります。

## Halide — 「何を計算するか」と「どう実体化するか」の分離

Halide（MIT 発・Adobe / Google で実戦投入）は画像パイプラインの DSL で、
**アルゴリズムとスケジュールの分離**という一点で有名です。

- アルゴリズム＝純粋な `Func`（座標から値への関数）。ここには「何を計算するか」だけを書く。
- スケジュール＝「いつ・どこで・どの順に計算し、どこに格納するか」（tile / fuse /
  vectorize / 計算の共有…）。**意味を一切変えずに**性能だけを変える層。
- Photoshop や Pixel のカメラパイプラインで使われ、「純粋関数＋実体化の指示」という
  切り分けが production で成立することを証明しました。

**SC0 から見ると**：「関数と実体化の分離」の直接の先行者ですが、分離している**層**が
違います。SC0 の実体化仕様（`spec`）は**意味に効く**——結果の `Buffer` の形状を
決める、型に現れる値です。Halide のスケジュールは**意味に効かない**——結果は不変で
性能だけが変わる。つまり Halide のスケジュールに対応するものを SC0 はまだほとんど
持っていません（哲学 §7 の「速さの階層は意味を変えない範囲で透過に」の層）。
将来 GPU 化や実体化の最適化を考えるとき、Halide のスケジュール語彙は
そのまま検討リストになります。

## 並べてみる

| | 添字の型付け | 形状の型 | 実体化 | 環境 | 主目的 |
|---|---|---|---|---|---|
| Dex | 添字集合（型） | 型に出る | `for`（配列構成） | なし | 自動微分・ML 研究 |
| Futhark | 通常の整数 | サイズ型（専用機構） | 言語全体が実体 | なし | GPU 性能 |
| Halide | C++ の変数 | 浅い | スケジュール（意味不変の層） | なし | 画像パイプライン性能 |
| SC0 | `Fin k`（依存型の一例） | `Vec n` / `Buffer d shape`（依存型の一例） | `parallel` / `render`＝**意味に効く値** | テキスト⇄グラフのライブ環境 | 制作 |

## SC0 の位置

3 つの言語はそれぞれ、SC0 が持つ部品のどれかを深く掘っています——
Dex は「配列＝添字からの関数」、Futhark は「サイズの静的追跡と所有権」、
Halide は「純粋関数と実体化の分離」。SC0 に固有なのは、これらを**一般の依存型という
ひとつの土台の上で統一**し、さらに**ライブな制作環境と結合**した点です。
サイズ型も有界添字も実体化仕様も、SC0 では専用機構ではなく「依存型で普通に書けるもの」
として存在しています。

逆に、この 3 つから学べるものもはっきりしています：Futhark の uniqueness
（linear type への足がかり）、Halide のスケジュール語彙（意味不変の性能層）、
Dex の並列性を保つ autodiff（微分可能な制作、という将来の可能性）。

## 参考リンク

- [Dex: array programming with typed indices](https://openreview.net/forum?id=rJxd7vsWPS) /
  [Getting to the Point（ICFP 2021）](https://arxiv.org/pdf/2104.05372) /
  [google-research/dex-lang](https://github.com/google-research/dex-lang) /
  [Sasha Rush による Dex チュートリアル](https://blog.rush-nlp.com/dex-tutorial.html)
- [Futhark（PLDI 2017 論文）](https://futhark-lang.org/publications/pldi17.pdf) /
  [Uniqueness Types and In-Place Updates（公式ブログ）](https://futhark-lang.org/blog/2022-06-13-uniqueness-types.html) /
  [言語ガイド](https://futhark-book.readthedocs.io/en/latest/language.html)
- [Halide](https://halide-lang.org/)
