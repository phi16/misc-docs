# SC0 開発ドキュメント — 概観

SC0 の実装に手を入れる人向けのドキュメントです。
設計判断の**経緯**はここではなくリポジトリ直下の一次資料にあります：
`philosophy.md`（7 原則）・`design.md`（決定レジスタ）・`design-workspace-tasks.md`（進行中の作業）・`TODO.md`。

各層の詳細ページ（それぞれ「責務 → 中心データ型 → 不変条件 → 落とし穴」の構成）：

- [Surface / Core と表示](dev/surface-core.md) — 型分割・CommonF・Deco・pretty・構造編集・graph/workspace
- [Elaboration](dev/elaboration.md) — 双方向検査・メタ・postpone fixpoint・instance 探索・リテラル解決・診断記録
- [NbE](dev/evaluation.md) — Value/Neutral・VForeign・メタ・評価エラーチャネル（型検査専用＋参照実装）
- [eval IR](dev/ir.md) — runtime の値モデル（hash-cons・closure conversion・KernelRender・extern）
- [kernel IR と GPU](dev/kernel.md) — 残差の lowering・CPU Runner・WGSL/wgpu（feature gpu）
- [モジュールと Cook](dev/modules-cook.md) — Module レジストリ・build_decls 3 フェーズ・incremental・forest
- [Web 層](dev/web.md) — worker/main 分担・wasm ABI・WebModule・view/placement システム

## リポジトリ構成

| パス | 内容 |
|---|---|
| `core/` | 言語本体（Rust クレート `sc0-core`）。CLI も同居 |
| `core/modules/` | 標準ライブラリの SC0 ソース（base / prim / std / algebra / math / mesh） |
| `wasm/` | wasm ブリッジ（生 C-ABI エクスポート。wasm-bindgen 不使用） |
| `web/` | ブラウザ編集環境（依存ゼロの素の JS＋vendor した CodeMirror） |
| `lsp/` | LSP サーバ `sc0-lsp`（`check <file>` CLI も） |
| `editors/vscode/` | VS Code 拡張（TextMate＋LanguageClient） |
| `docs/` | このドキュメント（docsify） |

## ビルドとテスト

```sh
./check.sh            # 一括：core cargo test → lsp build → wasm ビルド → web 回帰テスト
./build-wasm.sh       # core/wasm の Rust を変えたら（wasm → web/sc0_wasm.wasm 配置）
(cd core && cargo test)
(cd core && cargo test --features gpu)   # GPU≡CPU 差分テスト込み（wgpu headless・GPU 不在なら skip）
node web/test.js      # wasm ブリッジの回帰テスト（Node・ブラウザ不要）
node web/serve.js     # ブラウザで UI 確認 → http://localhost:8080
```

clippy は警告ゼロを保つ運用です。

## core のパイプライン

```
ソース文字列
  → lexer.rs    字句解析（Span つきトークン。Web のハイライトも同じ lexer が真実）
  → parser.rs   パース → Surface（位置つき表層項・de Bruijn）
  → types.rs    elaboration：双方向型検査（依存型・メタ変数・instance 探索・From 解決）
                 → Core Term（純粋・表示情報なし）
  ├→ eval.rs    NbE（Value・quote）＝型検査の正規化・定義的等価。**型検査専用**
  └→ ir.rs      eval IR＝runtime の値計算（hash-cons DAG・closure conversion・extern）
      → kernel.rs      render の座標残差を kernel IR へ lower（数値カーネル・CPU Runner）
        → kernel/gpu.rs  kernel IR → WGSL → wgpu 実行（feature gpu）
```

- **Surface / Core の型分割**：表層項（位置・装飾つき）と Core（純）は別の型で、
  共通の形は `CommonF` で共有。グラフは Surface 直結（各ノードが `src : NodeId` を持つ）。
- **NbE / eval IR の分離**：型検査は NbE、runtime は eval IR（quote の共有破壊を
  runtime から構造的に排除）。NbE の parallel は per-sample の独立参照実装として残り、
  IR≡NbE の差分テストが両者の一致を担保する。
- **モジュール**：`module.rs`（一級の `Module` 表現・唯一のレジストリ）＋
  `modules.rs`（ビルド `build_decls`・self-populating＝参照時に建てる）。
  プログラムとモジュールは完全に同じ経路（`build_decls`）。
- **メンバの二重メモ**：`MemberEntry`＝定義（Core 項）から NbE 値（型検査用）と
  eval IR（runtime 用）を対称に遅延導出してメモ化。
- **incremental（Cook）**：`cook.rs`。永続 `Cook` が WebModule と BuildCache を持ち、
  編集は dirty を印すだけ・query が最小再計算（green＋early-cutoff）。
  full ビルドと incremental の一致は differential テストで保証。

## core のファイル別責務

| ファイル | 責務 |
|---|---|
| `lexer.rs` | 字句解析（`Span` 付与） |
| `parser.rs` | パース・fixity 適用・束縛子列（Π/λ/nominal 共通） |
| `surface.rs` | Surface 項（位置つき表層） |
| `ast.rs` | Core Term・共通形 |
| `types.rs` | elaboration（双方向型検査・メタ・instance 探索・リテラル born/From 解決） |
| `eval.rs` | NbE（Value・quote・VForeign）＝型検査専用＋差分テストの参照実装 |
| `ir.rs`（＋`ir/tests.rs`） | **eval IR**＝runtime の値モデル（hash-cons DAG・closure conversion・KernelRender・extern） |
| `kernel.rs` | **kernel IR**＝数値カーネル（eval IR 残差の lowering・CPU Runner） |
| `kernel/gpu.rs` | kernel IR → WGSL 翻訳・wgpu headless 実行（feature `gpu`） |
| `prim.rs` | プリミティブ実装（`by_name`・簡約規則） |
| `module.rs` / `modules.rs` | Module 表現／ビルド・ソース供給（DirModules / BakedModules） |
| `cook.rs` | incremental ビルドエンジン |
| `graph.rs` | ノードグラフ表現・構造編集 |
| `workspace.rs` | Workspace（自動レイアウト規則） |
| `analysis.rs` | hover / 診断の共通解析（`Overlay`・エラー位置） |
| `mesh.rs` | ハーフエッジ実体（`Topo`） |
| `log.rs` | 汎用ログ（thread_local・Web の Log パネルへ drain） |
| `png.rs` | 依存ゼロの PNG エンコーダ |
| `web.rs` | Web 向けエントリポイント（WebModule・op ディスパッチ） |
| `lib.rs` | 公開 API（`run` / `render_png` / `diagnose` …） |
| `main.rs` | CLI |
| `tests/` | 統合テスト（analysis / cook / incremental / modules / typeclass） |

## web の構成

- `sc0.js` — wasm への薄いグルー（ブラウザ / Node 共通。`env.sc0_now_ms` などの import 供給）
- `worker.js` — 評価と Cook は worker 上の単一 `Cook` が持ち、main は非同期クライアント
- `main.js` ほか — タイル窓（`tilewin.js`）・Graph（`graphview.js`）・Workspace（`workspaceview.js`）・
  エディタ（`editor.js`＝CodeMirror）・各 view（`imageview.js` / `curveview.js` / `viewer3d.js` / …）・
  Log（`logpanel.js`）・Info（`infopanel.js`）
- `test.js` — Node での回帰テスト（ブラウザ不要）

## 書き方の規約（抜粋）

- コメント・ドキュメントは日本語、ユーザーに見える出力（Log・UI ラベル）は英語。
- match はワイルドカード `_` を避けて網羅する（variant 追加で漏れを炙り出す）。
- 失敗は隠さない：値の捏造・サイレントフォールバックをしない（philosophy.md §6）。
