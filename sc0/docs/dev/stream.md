# Stream（stream.rs）——音のランタイムと WebAudio lowering

対象ソース：`core/src/stream.rs`＋`web/streamplay.js` / `web/streamproc.js`。
**設計の真実文書は `design-stream.md`**（確定事項・排除した案 7 件とその理由・未解決の核が
全部そこにある）。このページは構造と不変条件の要約です。

## 表現：Stream 値＝eval IR の Tag 木

Stream は **`Tag("osc",[…])` / `Tag("mul",[…])` / `Tag("biquad",[…])` … の木**として
eval IR に載ります（専用の Node variant は作らない＝既存の一般機構 Tag の一例・哲学 §5）。
**spec（周波数など・静的）も state（位相・フィルタ履歴）も同じ構造に**入ります。

この表現に至るまでに 2 つの代替案が**原理的に**死んでいます（design-stream.md §5）：

- **opaque な step closure で持つ**——`Osc` / `Biquad` というノード情報が消えて、
  WebAudio に落とせなくなる（木は WebAudio の認識対象）。
- **VForeign（head+args で intern・state は side-table）で持つ**——`osc 440`（位相 0）と
  peel 後の rest（位相進行済み）が head+args 同一 → **同じ id に intern → state を clobber**。
  状態を持つ値を hash-cons IR に side-table 経由で入れるのは根本的に不整合。
  Tag 木なら state が構造に入るので、hash-cons が状態違いを自然に別ノードにする。

`Trigger` は**別表現**です：内部データ Record（`metro` なら `{interval, phase, index, step}`）で
**木を持ちません**——Tag 木は WebAudio native 認識のための構造で、Trigger に native 対応は
ない（poly は worklet / native ミキサ経由）ため。`trigger_peel` が Record を読み、
`scanT` / `merge` は Record の入れ子を再帰します。

## peel（renderStream）

`peel` は木を歩いて各ノードの block-op を合成します——**pointwise（osc・mul・add）は
parallel、recurrent（biquad・delay・echo）は block 内 scan**（「block 内=parallel」は
一般に嘘・biquad の y[i] は y[i-1] に依存する）。rest には**全ノードの状態が揃って進んで**
入ります。

重要な帰結：**renderStream は WebAudio の話ではありません**。native ノード
（OscillatorNode / BiquadFilterNode）からは位相も履歴も読み出せないので、rest を返す
eliminator は CPU reference / オフライン専用。live はこれと別の姿（木を丸ごと sink に
渡す）です。

## live の 2 経路（自動選択）

1. **native**：core が木から **descriptor（JSON）**を出し、JS（streamplay.js）が
   WebAudio graph を組む——osc→OscillatorNode・mul→GainNode・biquad→IIRFilterNode・
   stereo→ChannelMerger・add→fan-in・constS→ConstantSourceNode・
   **poly→GainNode ミキサ＋イベントごとに AudioBufferSourceNode**（voice＝有限クリップ＝
   AudioBuffer が正しい表現、というのが native 化の理由。軽さは帰結）。
2. **worklet**：native に落ちない形（`mkStream` の user ノード等）を含む木は、worker が
   `stream_open` / `pull` / `close` で剥がし続け、AudioWorkletProcessor（streamproc.js）へ
   MessagePort 直結で流す（~85ms 先読み）。**開いた stream の先端は Cook の GC root 必須**
   ——忘れると eval IR の GC に食われます。

click 対策は master gain の 6ms ramp。underrun は streamproc が検知して worker へ送り、
worker が負荷内訳（voice-gen / mix）を drain して Log に 1 行で出します（起きたときだけ）。

## 走行中の所有と live 更新

- **走行中インスタンスは（ノード, 値の同一性キー＝extern 適用済み木の hash-cons id）が
  所有**。再評価のたび view が比較し、**値が変わったら stop** が正直な既定。
- 走行中に動かしたいものだけが専用の口を通ります——どちらも「**木を不変に保ったまま
  中身が動く**」形なので streamKey が安定し、走行が止まりません：
  - **extern hold**：`apply_externs` が値を焼かず恒等 leaf `Tag("held",[slot])` を残す。
    native＝ConstantSourceNode に slider→setTargetAtTime、worklet＝pull ごとに現在値読み。
  - **hold（共有セル）**：`hold X` は走行セル 1 つ（現在の rest）で、参照は inline せず
    **`holdref(k)`** にコンパイル。**peel は block ごとにセルを memoize**——共有参照が
    同じ block を受け取るために必須で、**二重 peel すると状態が 2 回進んで壊れます**。
    swap 前に `peels_ok` で試し peel し、壊れた定義には swap しない（last-good 継続。
    試し peel は独立 block なので **memo をクリアして試し、復元する**——前回 pull の
    memo を使い回すと offset 不整合で underflow した実バグ）。

## hold の一般化と規則（coalgebra 一般・design-stream.md §13）

hold の意味は「状態があるか」でなく「**更新時に受け継ぐべきか**」。機構は Stream 特化を
外して coalgebra 一般です：

- **hold できるのは Stream か Trigger**（ルール①・型 solve 後に判定——`a + a` 等は
  elaborate 時点で未解決メタなので、incremental ループの後で見る）。
- **hold を参照する宣言は hold**（ルール②・効果のように伝播）。decl レベルの
  surface 参照 × kind で判定（elaboration 順に依らない）。`renderStream` で値化する参照も
  例外にしない（例外を作らない・§5）。
- Trigger の hold は `trigger_hold_peel`（per-block memo は Stream/Trigger **別**）。
  編集 swap は種類で分岐：**Stream＝equal-power crossfade（xfade ノード・~15ms）／
  Trigger＝plain 差し替え**（離散イベントは補間できない・state リセット）。
- **holdref は不透明なので、構造解析が中を見忘れる**：`uses_midi` / `collect_trigger_slots`
  が holdref で止まり、hold の下の midiIn / trigger が見えず無音になった。
  `resolve_holdref` で参照を辿ること——「参照で止まる」クラスのバグです。

## 編集検知＝依存グラフの affected 集合（member_ir id 比較は不可）

どの hold を swap するかの判定は、かつて member_ir の id 比較でしたが**撤去**されました。
member_ir は再パースで非決定です（同じコードでも kernel handle や closure が NodeId 依存で
新 id）——full setSource は全 decl が fresh NodeId になるので、「依存で再計算されただけの
hold」の id が変わって誤 crossfade しました（livesynth で音が化けた実バグ）。

現在は：

- `set_source` も `apply_edit` に畳まれ（冷たい全 reset を廃止）、**前後の decl text 差分**で
  「自分のテキストが本当に変わった decl」が取れる。
- `Cook::affected_decls` が dirty から依存を辿って swap 対象を出す。伝播は
  **let（インライン）辺だけで、hold 参照で cut**——holdref は runtime slot への参照なので、
  中身が変わっても consumer のコンパイル結果は変わらない。
- 帰結：hold を編集→それを参照する下流は swap されない（音継続）。上流の let を編集→
  それを取り込む hold が crossfade。
- **プログラム切替は crossfade でなく全停止**（`cook_stop_all_live`・同名メンバへの
  誤接続防止）。最後の live stream が閉じたら `hold_reset`。

## Live パネル（cook_live_state）

走行状態のダッシュボード（design-stream.md §14）。worker が holds（name・kind・source
hash・依存の逆矢印・topo 順・xfade 中か）／native poly（voice 数）／worklet streams／
MIDI・trigger・extern held 値を返し、main が native 再生分を足す。**push 型**です——
event push（edit/open/close/midi/…）＋ pull ごとの変化検出 push（前回送値と比較・
steady state では送らない）。パネルが見えている間だけ subscribe。

## 落とし穴

- **design-stream.md §5 の「排除した案」7 件を再提案しないこと**。特に
  「block 内は parallel」（3 回踏んだ）と VForeign aliasing。
- worklet / hold の**先端・セルは GC root**（eval IR は mark-sweep GC を持つ）。
- **`log::take` は drain**（読んで空にする）なので、per-eval の `clear` は不要かつ有害
  （eval の外で走った build——hold swap 等——のログを捨てる）。区切りは take が受け持つ。
- CPU reference と native の既知の差異：native osc は band-limited（高域エイリアスが
  僅かに違う・許容済み）。compress の細部も DynamicsCompressorNode と reference で異なる。
