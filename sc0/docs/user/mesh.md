# メッシュ

SC0 に特権的な「Mesh 型」はありません。設計は次の 3 点に分解されます。

1. **トポロジは値** `j : Topo`（内部はハーフエッジ構造・opaque）
2. 要素は `j` で**インデックスされた型**：`Point j` / `Vert j` / `Hedge j` / `Prim j`
3. **属性はただの関数**：位置は `Point j -> Vec 3 R`、UV は `Vert j -> Vec 2 R`、
   材質は `Prim j -> N` など

「メッシュ」はこれらを束ねた**普通のレコード**です：

```
{ j : Topo, pos : Point j -> Vec 3 R, color : Point j -> Vec 3 R }
```

共有された `j` が属性間の整合（同じトポロジの上に載っていること）を型で保証します。
トポロジを変えない演算は `j` が不変であることが型に出ます。トポロジが変わる演算は
新しい `j1` を依存レコードで存在パックして返します。

## 要素の 4 クラス

| 型 | 意味 |
|---|---|
| `Point j` | 共有点（position が載る単位） |
| `Vert j` | 角（面ごとの頂点。UV など面で切れる属性の単位） |
| `Hedge j` | ハーフエッジ（有向辺） |
| `Prim j` | 面 |

- 各要素は `Fin`（有界添字）を包んだ nominal です：`nominal Point (j : Topo) = { index : Fin (points j) }`。
  `points j` / `verts j` / `hedges j` / `prims j` が要素数を返します。
- `j` について injective なので、`Point j` の値から `?j` が推論されます
  （メッシュ関数の implicit `?j : Topo` はほぼ書かずに済む）。

## ナビゲーション

| 名前 | 型（`?j` 省略） | 意味 |
|---|---|---|
| `hedgeSource` | `Hedge j -> Vert j` | ハーフエッジの根元の角 |
| `vertHedge` | `Vert j -> Hedge j` | 角から出るハーフエッジ（`hedgeSource` と互いに逆） |
| `vertPoint` | `Vert j -> Point j` | 角の載っている共有点 |
| `vertPrim` | `Vert j -> Prim j` | 角の属する面 |
| `twin` | `Hedge j -> Maybe (Hedge j)` | 対面のハーフエッジ。**境界辺は対面なし**＝`none` |
| `next` | `Hedge j -> Hedge j` | 同じ面内の次の角送り（常に定義） |
| `hedgeTarget` | `Hedge j -> Maybe (Vert j)` | 先端の角（`twin` 経由の導出・境界は `none`） |

境界の「無い」は `Maybe` で正規に表されます（番兵値や例外はない）。

**⚙ 実装メモ**：内部では vert と hedge は同一のインデックスで、`hedgeSource` / `vertHedge` は
恒等です。しかしそれは opaque 境界の内側の事実で、表層では別の型です（内部表現に依存した
コードは書けないし、書く必要もない）。

**⚙ 実装メモ（接続子はテーブル）**：`vertPoint` / `vertPrim` / `next` は native ではなく、
**接続テーブル（`Array n N`・非公開）を `at` で引いて `clampFin` で要素に包むライブラリ定義**
です。つまり接続子は「汎用の配列読み＋データ」にすぎず、mesh 専用の特別扱いなしで
そのままカーネル（Gather）に乗ります。`twin` だけは結果が `Maybe` なので native のままです。

## 生成系

| 名前 | 型 | 内容 |
|---|---|---|
| `grid` | `(m n : N) -> { j : Topo, coord : Point j -> Vec 2 N }` | m×n 四角形グリッド。点 (m+1)(n+1)・面 mn。`coord` は各点の格子座標 |
| `fromTriangles` | `(po pr : N) -> Array pr (Vec 3 N) -> Maybe Topo` | 三角形リストから。添字が範囲外なら `none` |
| `fromPolygons` | `(po pr nc : N) -> Array pr N -> Array nc N -> Maybe Topo` | 多角形リストから（ns=各面の角数・flat=角の point 添字を面順に平坦・Σns≠nc は `none`） |
| `tetra` | `Topo` | 四面体（閉多様体・全ハーフエッジに twin あり）の fixture |
| `quad` | `Topo` | 四角形ひとつ（境界つき・twin は全部 `none`）の fixture |

不正な入力は `Maybe` で honest に `none` になります（例外や部分的な構築はしない）。

`smooth` / `normals` / `subdiv` / `marchingCubes` / `tessellate` は設計済み・未実装です
（`mesh.sc0` にコメントとして型だけ書かれています）。

## Web での表示

`{ j : Topo, pos : Point j -> Vec 3 R, … }` 型のレコードは Web の 3D ビューアで
表示できます（属性関数の型 `Point j -> A` / `Vert j -> A` … からドメインを読み取って抽出する。
[Web 環境](user/web-ui.md)）。
