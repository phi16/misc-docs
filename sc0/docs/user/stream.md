# 音とストリーム

時間で変化する値の抽象が `Stream` です（`stream` モジュール）。音の合成・ライブ再生・
ライブコーディングがこの上に載っています。バッファ領域の信号処理（FFT など）は
別モジュール `dsp` です。

## 考え方

- **`Stream a` ＝状態を持つ、終わらないサンプル列**（opaque 型）。実行時はブロック単位の
  状態機械です。`Buffer` が「有限に実体化されたデータ」なら、Stream は「無限に続く生成」。
- **フィルタやゲインは、専用の型ではなくただの関数** `Stream a -> Stream b` です
  （Faust のようなブロック図代数は要らない——関数がその役をする）。
- **Stream は rate-less**（純粋なサンプル列で、実時間を知らない）。実時間（Hz）に触れる
  `osc` だけが `sampleRate` を明示に取り、再生レートは再生側（view）の持ち物です。
  隠れた既定レートはありません。
- **スカラから Stream への持ち上げは常に可視の `from`** です（`From R (Stream R)`）。
  `from 440.0` は定数（DC）ストリーム。暗黙に持ち上がることはありません。

```
-- 440Hz のサイン波にローパスをかけ、音量を半分に
let sr = 48000.0;
let tone = osc WaveForm.sine sr (from 440.0);
let out  = from 0.5 * biquad (lowpass sr 1200.0 0.7) tone;
```

- **信号の和と積は演算子**です：`+`（ミックス）・`*`（ゲイン / VCA / リングモッド）。
  `Stream R` どうしの `AddOp` / `MulOp` インスタンスが定義されています。
  「gain」という特殊な名前はありません——両腕が Stream になった時点で、ゲインとはただの積です。
- **周波数などの制御入力も `Stream R`** です。`from 440.0` の代わりに LFO や
  `extern hold` を挿せば、そのまま FM・ライブ制御になります。

## 語彙（stream モジュール）

| 名前 | 型（implicit 省略） | 意味 |
|---|---|---|
| `Stream` | `U -> U` | ストリーム型 |
| `from` | `R -> Stream R`（From 経由） | 定数ストリーム（DC） |
| `WaveForm` | `enum { sine, square, saw, tri }` | 波形 |
| `osc` | `WaveForm -> R -> Stream R -> Stream R` | オシレータ（波形・sampleRate・時変周波数） |
| `+` / `*` | `Stream R -> Stream R -> Stream R`（stereo 版も） | ミックス／ゲイン |
| `biquad` | `{b0,b1,b2,a1,a2} -> Stream R -> Stream R` | 生係数の 2 次フィルタ（DF・漸化式そのもの） |
| `lowpass` / `highpass` | `R -> R -> R -> {係数}` | RBJ の係数導出（**純関数**・sr, freq, q） |
| `stereo` | `Stream R -> Stream R -> Stream (Vec 2 R)` | mono 2 本 → ステレオ |
| `delay` | `N -> Stream R -> Stream R` | 純遅延 z⁻ᵈ（d サンプル） |
| `echo` | `N -> R -> Stream R -> Stream R` | フィードバックエコー（自己参照は言語内で書けないため prim） |
| `compress` | `{threshold, ratio, attack, release} -> Stream R -> Stream R` | コンプレッサ |
| `convolve` | `(k : N) -> Array k R -> Stream R -> Stream R` | 畳み込み（IR リバーブの素） |
| `mkStream` | `S -> ((n : N) -> S -> {state, values}) -> Stream a` | **自前ストリーム（escape hatch）** |
| `renderStream` | `(n : N) -> Stream a -> { head : Buffer 1 [n] a, rest : Stream a }` | n サンプル剥がす（後述） |

多タップの FIR やエコー列は prim なしで書けます：
`x + from 0.5 * delay 12000 x + from 0.25 * delay 24000 x`。

## renderStream：観測は「剥がす」

```
renderStream 128 s   ==>   { head : Buffer 1 [128] R, rest : Stream R }
```

先頭の 128 サンプル（`head`）と、**状態が進んだ続きのストリーム（`rest`）**が返ります。
`rest` にもう一度 `renderStream` すれば次のブロック——これがオフライン実体化と
ライブ再生の共通の骨です。最初の一回から `rest` には状態（オシレータの位相・フィルタの
履歴）が乗っています。

## イベントとポリフォニー（Trigger / poly）

疎なイベント列は `Stream` とは**別の型** `Trigger a` です（毎サンプル `Maybe` を持つ
密な表現はしない）。時刻はユーザーに出しません——イベントごとに使えるのは
**序数 `index`**（何番目か）です。

| 名前 | 型（implicit 省略） | 意味 |
|---|---|---|
| `Trigger` | `U -> U` | 疎なイベント列 |
| `metro` | `(interval : N) -> ((index : N) -> a) -> Trigger a` | interval サンプルごとに発火 |
| `midiIn` | `(channel : N) -> Trigger { note : N, velocity : R }` | Web MIDI 入力（note-on。生の値＝周波数への変換はユーザーが書く） |
| `poly` | `Trigger a -> (a -> { len : N, buffer : Buffer 1 [len] b }) -> Stream b` | イベントごとに voice を焼いて重ねる（ポリフォニー） |
| `scanT` | `s -> (s -> a -> {state, payload}) -> Trigger a -> Trigger b` | Trigger の一般状態機械 |
| `merge` | `Trigger a -> Trigger a -> Trigger a` | イベント列の統合 |
| `mapT` / `indexed` | | `scanT` からの導出（ライブラリ定義） |

- **voice ＝有限クリップ**＝依存レコード `{ len : N, buffer : Buffer 1 [len] b }`
  （buffer の型が前のフィールド `len` を参照する telescope）。合成 voice は
  `renderStream` で焼き、サンプル素材ならバッファそのもの。
- granular synthesis の骨格：

```
poly (metro 12000 (λi. from (rand [i])))
     (λf. { len = 4800, buffer = (renderStream 4800 (osc WaveForm.sine 48000.0 f)).head })
```

`Unit = enum { tt }`（std）が「payload 無し」の正直な表現として使われます
（ボタンの Trigger は `Trigger Unit`）。

## 宣言の 2×2：let / hold / extern / extern hold

時間と外部入力が絡むので、宣言が 2 軸で分かれます——**値の出所**（コード / 外部）×
**持続するか**（純値 / 走行状態を持つ）。

| | コードが値を定義 | 外部（UI・runtime）が値を供給 |
|---|---|---|
| 純値 | `let x = …;` | `extern x : R = 0.5;`（観測ごとに凍る scalar） |
| **持続**（走行状態が編集を越える） | **`hold x = …;`** | **`extern hold x : Stream R = from 440.0;`** |

- **`hold`**（ライブコーディング用）：構文は let と同じで `hold` を前置。
  hold の意味は「状態があるか」ではなく「**更新（編集）のとき、走行状態を受け継ぐべきか**」
  です。走行中の実体は**名前つきの共有セル**として生き続け、複数のストリームから
  参照されても状態は一つ。**コードを編集すると滑らかに繋がります**——`Stream` は
  crossfade（約 15ms）、`Trigger` は即時の差し替え（離散イベントは補間できないため。
  状態はリセット）。編集がコンパイルできない間（parse エラー・型エラー）は
  **直前の正常な音が鳴り続け**、有効になった時点で反映されます。

  hold には**規則が 2 つ**あります（どちらも検査されます）：
  1. **hold できるのは `Stream` か `Trigger` だけ**。純粋な値の持続に意味はないので let で。
  2. **hold を参照する宣言は、自分も hold でなければなりません**——「持続する identity に
     依存する」性質は効果のように伝播します。`let` が hold を参照すると「完全に再導出できる」
     という let の意味が破れるのでエラーです（`renderStream` で値化する参照も例外なし）。

  編集の反映は**編集が本当に届く hold だけ**です：hold 自体を編集すればそこが crossfade、
  上流の `let` を編集すればそれを取り込んでいる hold が crossfade——しかし
  **hold を参照している下流は据え置かれます**（参照は走行セルへのポインタなので、
  中身が入れ替わっても下流のコンパイル結果は変わらない＝音が途切れない）。
  なお crossfade は同一プログラム内の編集だけで、**別プログラムのロードは停止**します
  （たまたま同名の宣言に繋がってしまわないように）。
- **`extern hold`**：走行中に値が動く外部入力の正規の道。型が `Stream R` なら
  スライダで音が**鳴りながら**動き、`Trigger Unit` なら**ボタン**（`init` は不要）。
  素の `extern` は「観測ごとに凍る」ので、ライブ再生には追従しません——
  render（フレームごとに観測）では動いて見え、音（一回の長い観測）では
  `extern hold` を使う、という使い分けです。

## ライブ再生

`#[node] {view:"stream"}` を付けたノードに ▶ / ⏹ が出ます（[Web 環境](user/web-ui.md)）。

**⚙ 実装メモ（アクセラレータ）**：WebAudio は GPU と同じ**透明アクセラレータ**として
使われます。意味は常に自前定義（renderStream の CPU reference が真実）で、Web で
鳴らすときだけ認識できる形を native ノードに落とします——osc → OscillatorNode、
`*` → GainNode、biquad → IIRFilter、stereo → ChannelMerger、`+` → fan-in、
poly → AudioBufferSourceNode 群。native に落ちない部分（`mkStream` など）を含む
ストリームは worker が剥がし続けて AudioWorklet に流します（経路は自動選択）。
既知の差異：native のオシレータは band-limited なので、高域のエイリアスが CPU reference と
僅かに違います（許容済み）。

**⚙ 実装メモ（走行中の同一性）**：走行中のストリームは「ノード × 値の同一性」が所有します。
編集で**値が変わったら止まる**のが正直な既定で、走行したまま動かしたいものだけが
`hold` / `extern hold` の口を通ります（木が不変のまま中身が動く設計）。

- 再生が実時間に間に合わないと **underrun** として Log に 1 行出ます
  （欠けたフレーム数と負荷の内訳つき）。
- `midiIn` を使うストリームの再生開始時にだけ、ブラウザの MIDI 許可を求めます。

## dsp モジュール（バッファ領域）

| 名前 | 型 | 意味 |
|---|---|---|
| `dft` | `(n : N) -> Buffer 1 [n] (Vec 2 R) -> Buffer 1 [n] (Vec 2 R)` | 離散フーリエ変換（複素 `[re, im]`・n は 2 の冪・radix-2 FFT） |
| `idft` | 同上 | 逆変換（1/n 正規化・`idft (dft x) ≡ x`） |

時間領域（Stream）とは別世界の Buffer → Buffer 変換です。`convolve` / `dft` / `idft` の
長さが明示引数なのは、`Buffer` の添字からの `?k` 推論に既知の穴があるため（`parallel` の
size と同じ明示の流儀）。
