# 動かし方

## 最小の例

SC0 のプログラムは宣言（`let` など）の列です。まず関数を書き、実体化して画像にします。

```
-- 色 = (x, y, 0)。R² → R³ の純粋関数を 64×64 でサンプリングする。
rasterize {lo = [0.0, 0.0], hi = [1.0, 1.0], res = [64, 64]}
  (λp. [p.0, p.1, 0.0])
```

`λp. [p.0, p.1, 0.0]` の段階では**まだ画像ではなく**ただの関数で、
`rasterize` が `Buffer 2 [64, 64] (Vec 3 R)` に実体化します。
このファイルは `core/examples/gradient.sc0` にあります。

チュートリアルとしてはこれだけです。以降のページは機能の列挙です。

## CLI

`core/` が Rust のクレートです。

```sh
cd core
cargo run -- prog.sc0            # ファイルを評価して「値 : 型」を表示
cargo run -- render prog.sc0 out.png   # Buffer 2 [w,h] (Vec 3 R) を PNG に書き出し
cargo run                        # 引数なし＝組み込みデモを実行
cargo test                       # テスト
```

- 標準ライブラリのモジュールは `core/modules/*.sc0` からディスク読みされます
  （`std` は Prelude として常時取り込み＝`import std;` は不要）。
- PNG 書き出しの対象は `Buffer 2 [w, h] (Vec 3 R)` 型の値です（成分は 0–1 の RGB）。

## Web 環境

ブラウザで動くエディタ＋ノードグラフ＋ビューア群です。wasm をビルドしてから静的サーバで開きます。

```sh
# 1) wasm をビルドして web/ にコピー
./build-wasm.sh

# 2) 静的サーバを起動してブラウザで開く
node web/serve.js        # → http://localhost:8080
```

ブラウザなしの検証は `node web/test.js`（wasm ブリッジを Node で実行）。
操作の詳細は [Web 環境](user/web-ui.md) を参照してください。

## LSP / VS Code

`lsp/` に LSP サーバ（`sc0-lsp`）、`editors/vscode/` に VS Code 拡張があります。

- 診断（構文エラーは精密な位置、型エラーは宣言内の項に位置づけ）
- hover（項の型を実名の束縛子で表示）・定義ジャンプ
- `sc0-lsp check <file>` で CLI からの検査も可能
- 拡張のビルドは `editors/vscode/build.sh`（要検証：手順の詳細）

開いているファイルの親ディレクトリがモジュール探索パスになります
（同じディレクトリの `.sc0` を `import` できる）。

## このドキュメント自体

```sh
node docs/serve.js       # → http://localhost:8081
```
