# kernel IR と GPU（kernel.rs / kernel/gpu.rs）

対象ソース：`core/src/kernel.rs`（数値カーネル・CPU Runner・lowering）＋
`core/src/kernel/gpu.rs`（WGSL 翻訳・wgpu 実行・feature `gpu`）

## 責務と全体の流れ

`parallel size f` の f をサンプルごとにインタプリタで評価する代わりに、
**型なしの数値カーネル（kernel IR）**にコンパイルして N 回回します（design.md §4）。

1. **部分評価**：eval IR のコンパイル（`ir::compile`）が `parallel` に出会うと、
   f を **symbolic な入力スロット `Coord(k)`** に適用して**座標残差**を作ります。
   座標に依存しない部分（型・辞書射影・定数・静的 `map`/`iterate` の unroll）は
   その場で畳まれ、専用の specialization 機構はありません——「inline の結果として創発」。
2. その残差（eval IR の Id）を `kernel::compile(residual, coord_dim)` が kernel IR へ
   lower します。戻りは `(Kernel, captures)`——**落とせない演算に出会ったら全体が失敗**
   （部分成功はない）。
3. strict な `parallel` は lower 失敗を**実行時エラー**にします。`parallel_fallback` だけが
   per-sample の NbE 評価に落ちます（黙る silent fallback は存在しない）。

かつては NbE の Value からも lowering する経路（Value カーネル）がありましたが、
**撤去済み**です。lowering の入口は eval IR 一本で、NbE 側の `parallel` は常に
per-sample（差分テスト用の独立参照実装）です。

## kernel IR の設計

- **単一アリーナの平坦な Op 列**。子は必ず自分より小さい index を指し、実行は
  index 昇順の 1 パス線形。各ノードは 1 サンプルにつき一度だけ評価（CSE）。
- **分岐は eager**（両枝計算）。木の遅延性は失うが CSE で重複が消え、GPU の
  predication モデルにそのまま合う。
- 値ドメインは **f64 スカラ / Bool / タグ / 静的長ベクトル**。
- `sampler` は**バッファを定数テクスチャとして閉じ込め**（texture fetch 対応）、
  `at`・mesh の接続子は整数添字の Gather（範囲外は実行時クランプ）。
  mesh の接続子が特別扱いなしでここに乗るのは、ライブラリ側が
  「テーブル（`Array n N`）を `at` で引く」形に書かれているからです
  （mesh 専用の recognizer は撤去済み・[user/mesh](user/mesh.md)）。
- 動的回数の `iterate`/`for` は Loop ノード＋サブカーネル。静的回数は eval IR 段階で
  unroll されるので Loop は出ない。
- **入力スロット `Var(k)`**：内訳は「座標軸（0..d）／ループの累算器・添字／
  **捕捉入力（captures）**」。captures は外側 λ の引数や `extern` 由来のスカラ式で、
  カーネルはそれらについて symbolic——**値が変わっても再コンパイルしない**
  （層 B・[dev/ir](dev/ir.md) の KernelRender 参照）。slider がライブなのはこの構造。
- CPU 実行は `Runner`：レジスタ列をカーネルごとに 1 回確保して全サンプルで使い回す
  （per-sample alloc なし・ループのスクラッチもネスト込み事前確保）。

**⚙ 歴史メモ**：旧実装の lowering メモは `Rc::as_ptr` キーで、一時値のアドレス再利用による
誤 CSE（テストの稀フレーク）を踏み、キーの Rc を生かす `keep` で塞いでいました。
現在はメモのキーが **eval IR の正準 id** なのでこの罠は構造的に消えています
（ポインタキーのメモ一般の教訓としては生きている）。

## GPU バックエンド（feature `gpu`）

kernel IR は元々 GPU 前提の形（index 昇順 1 パス・eager 分岐＝predication・
定数バッファ＝テクスチャ）なので、実 GPU へは素直に載ります。

- **ターゲット＝wgpu / WGSL**（native の headless テストにも wasm/Web にも同じコード）。
  精度は **f32**（WebGPU は f64 実質未サポート）。依存は **feature `gpu`** で optional
  （wgpu / pollster / bytemuck——普段のビルドは軽いまま）。
- `to_wgsl`：kernel IR →「`(coord, caps)` を受けて `array<f32, comps>` を返す WGSL 関数」。
  ベクトルは成分スカラに分解（alignment 問題の回避）。Case は入れ子 select。
- **動的 Loop** も翻訳できます（`var acc`＋WGSL `for`・サブカーネルは scope 接頭辞つきで
  再帰翻訳）。mandelbrot が——Complex の環インスタンスまで完全にインラインされた——
  綺麗な 1 枚の WGSL になります。
- `build_entry`：`@compute` エントリポイント化。flat な gid を shape で多次元添字に分解。
  **shape はリテラル焼き**（解像度変更時のみ再翻訳）、**caps（slider 等）は焼かない**
  （recompile-free を GPU でも維持）。
- `run_gpu`：wgpu headless 実行（device キャッシュ・storage バッファ・dispatch・readback）。
  GPU が無い環境では None で skip。
- **検証は UI 非依存**：`cargo test --features gpu` が **GPU 出力 ≡ CPU Runner 出力**の
  差分テストを回します。Sample/Gather は CPU と同じ算術で bit 一致。Rand だけは
  hash 実装が別（u64 の無い f32 世界で murmur3）なので値一致を要求しません。
- 既知の制限：Sample/Gather はトップレベルのみ（ループ body 内は buf の scope 衝突で未対応）。
- **残り＝Web の WebGPU 配線**（同じ WGSL をブラウザで dispatch して viewer へ・
  CPU フォールバックは残す）。

## 計測

実行経路（kernel IR / per-sample fallback）・次元・サンプル数・所要時間は `log.rs` 経由で
Web の Log パネルに出ます。IR 化の成否を疑うときはまずここを見る。
