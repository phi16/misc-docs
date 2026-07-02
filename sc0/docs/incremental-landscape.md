# incremental computation の系譜——Cook はどこに立つか

「編集のたびに全部を計算し直さない」は、独立した研究分野と実装の系譜を持つ主題です。
SC0 の Cook（incremental な型検査・評価エンジン）が使っている言葉——green・
early cutoff——もこの系譜から来ています。地図を描きます。

## ビルドシステムという見方：Build Systems à la Carte

Mokhov・Mitchell・Peyton Jones の「Build Systems à la Carte」（ICFP 2018）は、
Make・Shake・Bazel・Nix、そして **Excel**（スプレッドシートはビルドシステムである！）
を統一の枠で分類した論文です。ビルドシステムは 2 つの直交する部品でできている：

- **scheduler**（どの順に課題を処理するか）：topological（Make）・restarting（Excel・
  Bazel）・suspending（Shake・Nix）
- **rebuilder**（再計算が必要かをどう判定するか）：dirty bit・verifying traces・
  constructive traces …

そして **early cutoff**——再計算した結果が前と同じなら下流を走らせない——が
重要な最適化として位置づけられます。

**Cook をこの語彙で言うと**：宣言の依存 DAG（forest_deps）を上から処理する
topological scheduler ＋「dirty 印と依存の変化で再計算を決め、再計算結果の正規形を
前回と比較して伝播を止める」rebuilder（dirty bit ＋ early cutoff）。ambient
（fixity / nominal / instance / import）の変化だけは精密な依存に乗らないので
粗く全体を無効化する、という安全側の割り切りも入っています。

## クエリシステムという見方：rustc と Salsa

コンパイラの incremental 化で現在の主流は**クエリ（メモ化された純関数）**の形です。

- **rustc の red-green アルゴリズム**：計算をクエリの依存グラフとして記録し、
  入力が変わったとき「再計算して結果が同じなら green（下流はそのまま）、違えば red」
  と印を付けて波及を最小化する。**Cook の「green」という語はここから借りています**。
- **Salsa**（rust-analyzer の基盤）：red-green を Rust のライブラリにしたもの。
  リビジョン番号・durability（変わりにくさの層別）などの実務的な工夫を持つ。

Cook がクエリ粒度でなく**宣言粒度**なのは意図的です。SC0 には「トップレベル宣言は
局所的に型が閉じる」という言語側の保証（§4）があるので、宣言を単位にすれば
依存が名前参照から正確に出て、粒度として十分細かい。言語の性質が incremental の
単位を決めている、という関係です。

## 理論の系譜：self-adjusting computation

より根源的な系譜として、Acar らの **self-adjusting computation**（計算の実行痕跡を
グラフとして持ち、入力変更時に痕跡を辿って差分再実行する）と、その系統の
**Adapton**（demand 駆動の incremental 計算）があります。λ 計算そのものを
incremental にする話で、粒度は式レベルまで細かくなります。

SC0 の将来課題との対応もここにあります：現状の Cook は宣言粒度ですが、
設計メモには **incremental elaborate**（宣言の中の部分再検査・ライブ構造エディタ層 3）
や **render IR キャッシュ**（スライダの値を実行時入力にしてカーネル再コンパイル自体を
省く）が控えています。粒度を細かくするほど、この理論系譜の問題（何を痕跡として
持つか・比較可能性）に近づきます。

## UI の系譜

React の仮想 DOM 差分や、Jane Street の Incremental（DAG ベースの incremental
計算ライブラリ）など、UI 更新も同じ問題の別の顔です。SC0 の Web 層にも同じ構図が
現れています——スライダの surgical 更新（値の span だけ書き換えてエディタの他を
動かさない）や、評価結果の `lastRes` 比較による view への再供給抑制は、
「変わっていないものに触らない」の UI 版です。

## 貫いている原則

系譜全体に共通する成立条件が 2 つあります。

1. **比較できること**。early cutoff は「結果が前と同じ」を判定できて初めて成り立つ。
   Cook が正規形（quote した項）を比較するのはこのためで、NbE が正規形を持つ言語で
   あることが incremental の土台になっています。
2. **正しさの基準は full 再計算との一致**。Cook はこれを differential テスト
   （incremental ≡ full・しかも同じ関数 `build_decls_impl` の 2 モード）で担保して
   います。「full と incr を別実装にしない」は、この分野で壊れやすさの筆頭が
   「両者の乖離」であることへの構造的な答えです。

## 参考リンク

- [Build Systems à la Carte: Theory and Practice（JFP 2020）](https://ndmitchell.com/downloads/paper-build_systems_a_la_carte_theory_and_practice-21_apr_2020.pdf) /
  [ICFP 2018 版（Microsoft Research）](https://www.microsoft.com/en-us/research/wp-content/uploads/2018/03/build-systems.pdf)
- [rustc dev guide — Incremental compilation（red-green）](https://rustc-dev-guide.rust-lang.org/queries/incremental-compilation.html)
- [Salsa](https://github.com/salsa-rs/salsa)
- [Adapton](http://adapton.org/) / Acar, Self-Adjusting Computation
- [Incremental（Jane Street）](https://github.com/janestreet/incremental)
