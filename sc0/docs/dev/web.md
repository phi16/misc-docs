# Web 層（wasm / worker / main）

対象ソース：`core/src/web.rs`・`wasm/`・`web/*.js`

## 役割分担

```
main（UI スレッド）          worker                     wasm（sc0_wasm.wasm）
 ├ タイル窓・各パネル          ├ 単一の Cook（生きた module）  ├ core を C-ABI で export
 ├ 言語状態を持たない          ├ 編集オペ受信→最小再計算      └ sc0.js が手書きグルー
 └ 編集を送り query を引く     └ per-node 評価＋Log drain
```

- 言語状態の所有者は **worker の Cook ただ 1 つ**（[modules-cook](dev/modules-cook.md)）。
  main は純クライアントで、Workspace/Graph などの軽い派生 view は main 側の
  （キャッシュ無し・純関数の）wasm インスタンスが担います（同じ真実からの再導出＝
  二つの真実ではない、という整理）。**main と worker は別々の wasm インスタンス**である
  ことに注意——main 側の呼び出し（`workspace`/`graphMember`/`diagnose`…）はすべて
  `(src, …) → 結果` の純関数形で、状態を持ちません。
- **wasm-bindgen は使いません**。生の C-ABI エクスポート＋手書きグルー `sc0.js`
  （ブラウザ / Node 共通）。wasm には `std::time` が無いので、時刻は **JS 提供の import**
  `env.sc0_now_ms` をすべてのインスタンス化に供給します（Log の計時が動く条件）。

## worker プロトコル（worker.js）

| メッセージ | 意味 |
|---|---|
| `setSource { src }` | 冷たい resync（初期化・構造編集・Source 自由編集）。唯一の全文経路 |
| `editDecl { name, text }` | hot path＝1 宣言差し替え（slider・Code）→ 下流だけ再評価 |
| `eval { id, node }` | per-node 最小評価。結果＋この評価の Log を返す |
| `glyphs { config }` | pretty の glyph 設定を main と揃える |

- メッセージは FIFO 処理なので「編集 → eval」の順序は自然に保たれます。
- `editDecl` の失敗（drift）は**握りつぶします**——Cook は前の有効な状態のまま・crash しない。
  main が次の `setSource` で追随する設計です。
- buffer / mesh の `ArrayBuffer` は **transfer** で返してコピーを避けます（大きい）。
  戻り値に配列を足すときは transfer リストへの追加を忘れないこと。

## main.js の構造

中心は `rebuild(refit)`。編集・選択変更のたびに呼ばれ、次を順にやります：

1. `sc0.workspace(src)` → Workspace レイアウト・選択の維持（`pickNode`）
2. `sc0.graphMember(src, selectedNode)` → Graph パネル（パースのみ＝軽いので main で同期）
3. Code / Info / Workspace Info の更新
4. **評価集合の決定**（placement 駆動）：選択ノード ∪ view 付きノード ∪ window placement の
   ノード。ただし `graph` kind は eval でなく `graphMember` の計算（データ源が違う）
5. `syncCook()` → worker の Cook を現ソースへ追随させてから、`evalNode` を投げる

### cookSync：worker への追随プロトコル（3 値）

main は「次の rebuild で worker にどう追随するか」を変数 `cookSync` に積みます。
**編集の種類ごとにどれになるかが性能の要**です：

| 値 | 意味 | 例 |
|---|---|---|
| `"full"` | 全文 `setSource` | Source の打鍵・構造編集（add/delete/rename/connect/Code 実行）・Examples ロード・Reformat |
| `{decl: name}` | その 1 宣言だけ `editDecl`（hot path） | slider のドラッグ |
| `null` | 追随不要（eval に効かない） | ノード選択・`pos` 注釈のドラッグ永続化 |

`pos` の永続化（`persistPos`）が `cookSync = null` に**打ち消す**のは意図的です：
注釈は意味に効かない透明層なので worker に送る必要がなく、送っても early-cutoff で
無害ですが、そもそも送らないのが正しい形。

### 評価結果のルーティング

- `evalNode` は連番 id を振り、`nodeSeq`（ノード → 最新 id）で**古い結果を捨てます**
  （編集の途中に届いた stale な評価が UI を巻き戻さない）。
- 結果は `nodeResults`（ノード → 直近結果）に入り、(a) 選択ノードなら Result＋レンズ、
  (b) box の view（`wsViews`）、(c) window placement（`winPlacements`）へ push されます。
- 静的（型）エラーは main 側で `diagnose` を引いて `行:列` を前置します（エディタの
  下線と同じ位置になる）。
- cook ログは**全ノード分**を Log へ流します。以前は選択ノード限定で、ビルドを起こした
  ノードが別だとログが消える（「Log でなくなる」）バグがありました。

### placement レジストリ（box / window）

view の統一モデル（`design-workspace-tasks.md` §4・完了）の実装です。

- **box**（`wsViews`: ノード → {el, view, kind, metaKey}）＝**導出**。描画のたび
  （`onViewport`）に `syncWsOverlays` が「view 注釈つきの箱」から作成・配置・カリング・
  撤去まで全部やる純関数的な同期。view メタ（slider の min/max 等）が変わったら作り直し
  （`metaKey` 比較）。
- **window**（`winPlacements`: 配列）＝**命令的**。Workspace Info の Pop out で開き、
  ユーザーが閉じるまで残る。ノードが module から消えたら **null を push して空表示**
  （窓は壊さない・§6）。プログラムのロード時は全 popup を閉じる（別プログラムの同名
  ノードへ名前で誤接続しないため）。
- 供給は **host が push・view は受け身**：`view.setValue(state)`。view は sc0 を呼ばない。
  `lastRes` 比較で同じ値の再 push を避ける。

### syncWsOverlays の描画詳細（見た目の整合が壊れやすい所）

- **枠は canvas が描く**（○ノードと同一ラスタライザ＝太さ・位置が完全一致）。
  overlay の DOM は「中身だけ」を stroke の内縁まで inset して重ねる。
  DOM の border で枠を描く案は device-pixel 丸めのズレで却下済み。
- overlay は `transform: scale(vp.scale)` で world→screen に追従。canvas 系 view には
  `setScale(dpr×倍率)` で backing store を焼き直させ、拡大時のボケを防ぐ
  （表示中・倍率変化時のみ再レンダ）。
- 画面外はカリング（`display:none`）。表示復帰時にサイズ確定後 `setValue` を再供給
  （image の fit が正しいサイズで走るように）。
- 埋め込み view は基本 `pointer-events: none`（操作は下の canvas）。slider のような
  対話要素だけ auto にし、**その上で起きた wheel は canvas へ転送**して Workspace の
  ズームが途切れないようにしている。

### 編集の最小反映（applySourceEdit）

構造編集の結果テキストをエディタへ当てるとき、全文置換ではなく
`sc0.minimalEdit(old, new, name)`＝「宣言 name を起点とした最小編集」を使います。
1 ブロックで済む編集（Code / set_let / annotate / add）は該当ブロックだけ差し替え、
複数ブロックに波及する編集（delete / rename の cascade）は全文置換にフォールバック。
どの行範囲を触ったかは Log に出ます（`edit <名前>: [12, 15)`）——「その宣言しか
変わっていない」ことを観測可能にするため。

slider（`setNodeValue`）はさらに軽く、`memberBodySpan` で**値式の char span だけ**を
`editor.replaceRange` します。ドラッグ中の連続適用は rAF で束ねて最後の値だけ反映
（re-parse 過多の回避）。CodeMirror 側は programmatic 編集を `external` フラグで
onChange 非発火にしてあり、エコーバックしません。

### 選択（selectedNode）の 3 状態

`undefined`＝初回（既定選択する：`output` → 末尾）、`null`＝**明示的に未選択**
（勝手に選び直さない）、文字列＝選択中。空所クリックで null になり、それは維持されます。
「何も選択しない」を正規の状態として扱うための区別なので、潰さないこと。

## web.rs：WebModule（core 側の編集境界）

`WebModule { imports, decls, annos }`＝Web が編集する module（名前 `self`）の一級の値。

- **唯一の parse 境界**（`WebModule::parse`）と**唯一の pretty 境界**（`to_source`）。
  Source テキストは module の**忠実な直列化**なので、操作ごとの再パースは無損失＝
  在メモリ保持は最適化であって correctness 要件ではない。
- 断片（1 宣言の body など）の再パースは**モジュール文脈**（宣言名集合＋fixity）で行う。
  孤立でパースすると兄弟参照・ローカル演算子が unbound になります（Code パネルが
  ungraph でなく `memberDecl`/`memberBody` を使うのも同じ理由）。
- `member_body_span`：宣言の値式のソース **char span**。surface の span はトークン添字
  なので、再 lex したトークンの char span で変換します（slider の surgical 更新の土台）。
- `#[node]` 注釈（annos）は宣言名 → 生テキスト。**core は中身を解釈しません**
  （透明層・JS が JSON として解釈）。

## タイル窓（tilewin.js）

- 木のモデル：`leaf { key }` ／ `cont { layout: h|v|tab, children, sizes, active }`。
  分割とタブは**同一ノード種**（i3 流）。デスクトップは `{ tree, geom, z, full }` の窓を複数持つ。
- パネルは静的（CATALOG＝`#dock` 内の wrap 要素を葉に移送）と動的（`panelHost`＝
  ビューアや popup が後から登録・**transient で永続化しない**）の 2 種。
- レイアウトは localStorage に保存。復元時は `sanitize` が未知キー・重複・空コンテナを
  落として正規化する（CATALOG の変更やバグで保存値が腐っても起動できる）。

## レンズ（lenses.js）と view（views.js）

- **レンズ**＝Result のバッファ `{shape, comps, data}` を「何として見るか」のボタン列。
  `fits(shape, comps)` が真のものだけ並ぶ（自動判定しない）。画像は buffer が x-major
  なので **y-major へ転置**してから viewer へ渡す。長い 1D の wrap、mono/stereo audio、
  surface、curve 3D など。mesh はバッファでなく専用経路（worker が Topo＋属性関数から
  positions/normals/colors/tris/edges を抽出して transfer）。
- **view**（`createView(kind, opts)`）＝box / window に埋め込む統一インターフェース
  （`mount / setValue / naturalSize / unmount`、任意で `setScale` / `mountControls`）。
  各ビューアは host 非依存の content（`createXContent`）と、lenses 用の窓ラッパ
  （`createXViewer`）に分離済み。操作 UI は `mountControls`＝**窓のときだけ**（暫定。
  将来は box 単体で 3D 回転・再生等ができる方向が明言されている）。

## 主な JS ファイル

| ファイル | 責務 |
|---|---|
| `sc0.js` | wasm グルー（初期化・文字列 marshalling・`env.sc0_now_ms` 供給・API 面） |
| `worker.js` | 上記プロトコル |
| `main.js` | 全体の配線（rebuild・cookSync・placement・評価ルーティング・構造編集キー） |
| `tilewin.js` | タイル窓エンジン（floatwin.js は el/drag ヘルパの残骸） |
| `views.js` / `lenses.js` | view kind レジストリ／レンズボタン |
| `imageview.js` / `curveview.js` / `viewer3d.js` / `audioview.js` | 各 content 実装 |
| `graphview.js` | Graph 描画（layoutGraph / renderGraph / hitTest・THEME・mini モード） |
| `workspaceview.js` | Workspace レイアウト（forest 規則）・`parseAnno` |
| `canvasview.js` | pan / zoom / DPR / ResizeObserver つき canvas（Graph と Workspace で共有） |
| `editor.js` | CodeMirror 6 ＋ textarea 互換アダプタ（vendor は npm＋esbuild で `web/vendor/`） |
| `infopanel.js` | Info（項ノードの編集・fold/unfold・let ヘッダ） |
| `logpanel.js` / `theme.js` / `syntax.js` / `samples.js` / `reference.js` | Log／テーマ／構文辞書／サンプル／Library・Examples 窓 |
| `serve.js` / `test.js` | 依存ゼロ静的サーバ／Node 回帰テスト（ブラウザ不要） |

## 落とし穴

- **Log の計時**：型検査と評価の時間は build_decls 内で phase を分けて計っています。
  黒箱で外から計ると lazy メンバのせいで「型検査時間に評価が混入」します。
- **glyph 設定**は main と worker の両インスタンスに送る（片方だけだと pretty の出力が
  食い違い、往復比較が壊れる）。
- **graph view のデータ源分岐**：popup を開いた直後の供給も kind で分岐すること
  （eval 結果を graph view に流しても何も描けない）。
- **sc0（wasm）は非同期ロード**：editor が `getSc0` で即参照するため、`let sc0 = null` の
  宣言位置に TDZ の罠がある（過去に踏んだ）。ロード完了前の UI は disabled にしておく。
- **display:none 中の canvas / CodeMirror**：隠れたパネルは fit を保留して表示時に消化
  （canvasview）、CM6 は復帰後に `requestMeasure` が要る。
