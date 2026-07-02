# 型クラスの系譜——本当に欲しかったのは辞書と implicit

型クラスを持つ言語の歴史は、こう要約できます：**型クラスとは「演算の入ったレコード
（辞書）」と「それを暗黙に渡す機構（implicit）」の 2 つの部品でできているのに、
最初にそれを 1 つの不透明な機構に融合してしまったため、各言語が部品を少しずつ
掘り出し直してきた**。SC0 は最初から部品のまま持つ、という選択をしています。

## Haskell — 融合した機構と、その代償

Haskell の型クラスは `class` / `instance` という専用宣言で、辞書渡しは**実装の秘密**
です（ユーザーからは見えない）。この融合の代償が、以後 30 年の拡張リストになりました：

- 辞書が値でないので**ローカルなインスタンスが作れない**。`Data.Reflection` のような
  トリックが生まれる。
- 「同じ型に 2 つのインスタンス」が表現できないので、`Sum` / `Product` のような
  **newtype ラッパ**で演算を選ぶ（N のモノイドは加法か乗法か問題）。
- 一貫性（coherence）は「インスタンスは全世界で一意」というグローバルな約束で守る
  ——orphan instance 問題・overlapping instances の泥沼はここから。

## 部品を掘り出した言語たち

- **Scala（implicits → given/using）**：辞書は普通の値（implicit val）、受け取りは
  implicit パラメータ——**辞書＋implicit にほぼ分解した**最初のメジャー言語です。
  ただし implicit conversion との混線や優先順位規則で悪名も背負い、Scala 3 で
  `given` / `using` として整理し直されました。
- **Rust（trait）**：融合型の洗練版。coherence を orphan rule で厳格に守る代わり、
  ローカルインスタンスも複数インスタンスもない。堅牢だが硬い。
- **Agda（instance arguments）**：`{{ _ : Monoid A }}` は**ただの implicit 引数**で、
  解決はスコープ内の値からの探索。辞書は普通のレコード。ローカルインスタンス可。
  SC0 にもっとも近い形です。
- **Idris**：interface は**レコードに脱糖**され、解決は auto implicit。やはり部品構成。
- **Lean**：インスタンスは `@[instance]` の付いたただの定義で、解決は（表を引く）
  証明探索。mathlib の巨大な代数階層がこの上に建っている。

つまり証明支援系の系譜（Agda / Idris / Lean）では「クラス＝レコード・インスタンス＝値・
特別なのは探索だけ」がすでに常識で、SC0 はその側に立っています。

## 依存型だからできること

部品に分解しただけでは終わりません。**依存型があると辞書の型が値を持てる**ので、
Haskell が構造的に表現できなかったものが素直に書けます。

- **演算を型のインデックスにする**：SC0 の `Monoid m (op : m -> m -> m)` は
  「m の・この演算についての・モノイド」という型です。`N` の加法モノイドと
  乗法モノイドは**別の型**であり、newtype ラッパは要りません。
- **結果型を辞書に持たせる**：`AddOp a b = { result : U, add : a -> b -> result }`。
  異種オペランドの演算が、戻り型を辞書の値として運ぶ 1 つの枠に収まります。
- **法則をフィールドに載せる**：`unitL : (x : m) -> op e x ≡ x`。クラスの法則が
  コメントではなく型になる（埋め方は postulate でもよい——[vision](vision.md) 参照）。

## SC0 の探索の特徴

機構が「探索だけ」だからこそ、探索そのものの設計が個性になります。

- **出力方向の解決（result-directed）**：`_+_ 1 2 as N` のように結果型から辞書を絞れる。
  未解決の辞書メタを含む単一化を「その探索の制約」として貯め、候補の仮入れ時に
  まとめて検査する実装です。入力の型だけで解く伝統的な解決より対称的。
- **順序に依存しない遅延解決**：goal にメタが残る間は探索を保留し、他の制約が型を
  確定させてから一意一致で解く。「書いた順に決まらないと失敗」しない。
- **曖昧はエラー**：候補が複数残れば黙って選ばず ambiguous と言う（No ad-hoc）。
  Haskell の overlapping のような優先順位規則を持たない。
- **parametric instance は関数**：`AddOp a a` から `AddOp (Vec n a) (Vec n a)` を
  導出するインスタンスは、premise を implicit に取るただの関数として書く。

## まとめ

「本当に欲しいものは辞書＋implicit だった」という読みに立つと、型クラス史の混乱
（orphan・overlapping・newtype ラッパ・ローカルインスタンス問題）の多くは
**融合の後遺症**として説明できます。SC0 は部品を融合せず、依存型の力
（値インデックス・結果型・法則）をそのまま辞書に流し込む構成です。
`instance` 宣言に残った役割は「この値を探索対象にする」というマーキングだけ——
機構の表面積としてはこれが最小に近いはずです。

## 参考リンク

- [Type Classes — Theorem Proving in Lean 4](https://lean-lang.org/theorem_proving_in_lean4/Type-Classes/)
- [Instance Arguments — Agda docs](https://agda.readthedocs.io/en/latest/language/instance-arguments.html)
- [Scala 3: Given / Using](https://docs.scala-lang.org/scala3/reference/contextual/givens.html)
