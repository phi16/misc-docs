# モジュールと incremental ビルド（module.rs / modules.rs / cook.rs）

対象ソース：`core/src/module.rs`（実体）・`core/src/modules.rs`（ビルド・native 供給・forest 操作）・
`core/src/cook.rs`（incremental エンジンの持ち主）

## module.rs：一級の Module と唯一のレジストリ

```rust
struct Module {
    members:   HashMap<String, MemberEntry>,  // 型＋定義＋（NbE 値 / eval IR の）遅延メモ
    nominals:  HashMap<String, NominalDef>,
    instances: Vec<(V, V)>,       // 単相=VNominal・parametric=VPi
    fixities:  FixityList,
    reexports: Vec<(String, Option<Vec<String>>)>,  // open import 先（選択再公開は Some）
    private:   HashSet<String>,
}
```

- **唯一のレジストリ** `MODULES: HashMap<String, Module>`（thread_local）。かつて
  eval/types/ast に散らばっていた並行 thread_local を統合したもの。`build_module` が 1 回
  書き、以後は read-only の固定事実。各サブシステムの accessor（`scope_of`・`base_fixities`
  など）はこの 1 つの値への **domain view**。
- **self-populating**：読み窓口 `with_module` が「未構築かつ既知なら建ててから読む」。
  明示の ensure 呼び出しは存在せず、**「未構築状態」を呼び出し側から観測できない**構造。
  再入対策として、ビルド開始（**パース前**）に `BUILDING` スタックへ積む——パース中の
  fixity 参照が依存モジュールのビルドをカスケードさせるため、パース後では間に合わない。
  スタックのトップ以外で自分に再入したら import 循環として明確に panic（Drop で pop するので
  panic 経路でもスタックが汚れない）。
- `MemberEntry { ty, def, v, ir }`：定義（メタ無し Core 項 / native 値 / extern 既定値）が
  真実で、NbE 値（型検査用）と eval IR（runtime 用）を**対称に遅延導出してメモ**します
  （[evaluation](dev/evaluation.md)）。build_decls の phase 3（force）が runtime 値（IR）を
  確定し、render の実行・計時・エラーはここで起きます。
- **`extern name : T = init;`** も ModDecl の一種です。init は宣言型に対して elaborate され、
  IR では大域入力スロット `Extern(k)`、NbE では `NMember`（型検査では opaque）になります。
  現在値は member_ir に**入らない**——host が環境から供給するので、値の変更では
  Cook の再ビルドが一切走りません（[dev/ir](dev/ir.md)）。
- `private` は名前解決（`member_origin`）を他モジュールから引けなくするゲート 1 点。
  値・型は members に普通に入っている（自モジュール内では使える）。
- ソース供給元は provider：既定 `DirModules`（ディスク読み・LSP は開いたファイルの親 dir）、
  Web は `BakedModules`（include_str! 焼き込み）。provider を差し替えたら必ず
  `reset_registry` を呼ぶこと（stale 判定を考えない素直な形）。

## modules.rs：build_decls（ビルドの共通核）

**プログラムとモジュールは完全に同じ経路**です。`build_decls(module, imports, decls)` が唯一の核。

3 フェーズ構成（位相が重要）：

1. **全宣言を elaborate**。メンバは**生（未 zonk）で格納**——後続宣言からの参照可用性の
   ためだけ。per-decl に solve も回す（§4 の強制：**トップレベル宣言は局所的に型が閉じる**。
   後続の使い方から決まってはいけない。これが incremental の前提でもある）。
2. **solve_problems を 1 度**。
3. **全成果物を solve 後に一括 zonk**して登録＋全メンバ force（型検査と評価の計時は
   ここで分かれる。is_top のみ Log へ）。

**不変条件（zonk は 1 箇所・solve の後だけ）**：この位相を崩して「解決前に zonk」すると
未解決の辞書メタを項に固めるバグのクラスが復活します。

**不変条件（decls は借用のまま）**：hover は elaboration の記録（NodeId キー）とパース時の
span テーブルを join します。`build_decls` が `decls` を clone・再構築すると join が
**静かに外れます**（型が出なくなるだけでエラーにならない）。呼び側は join が終わるまで
decls を生かすこと。

### collect モード（エラー多重化）

`build_decls_collect` は phase1 で宣言が落ちても打ち切らず、失敗した宣言（型注釈あり）を
「**本物の宣言型を持つ stuck メンバ**」（`NMember` 中立項）として登録して次へ進みます。
値を捏造しないので後続の型検査は本物で続き、eval も panic しません。phase1 に 1 つでも
エラーがあれば phase2/3 はスキップ。`BuildError { decl: Option<usize>, msg, chain }` の
`chain` が診断の位置付け（NodeId 内側→外側）です。

### per-decl solve 固有の失敗の reframe

リテラル曖昧・ambiguous instance で落ちたとき、旧・全体 solve なら後続宣言が確定していた
ケースがあるため、「トップ宣言は自分だけで型が閉じる必要がある」という理由をエラーに
添えます（複数宣言のときだけ。単一宣言では誤解を招くので触らない）。

## incremental（build_decls_incremental ＋ BuildCache）

full と incremental は**同じ関数** `build_decls_impl` です（green 判定と early-cutoff を
挿しただけ。別実装を持たない＝結果一致が構造的に保ちやすい）。

- **green 判定**：dirty でなく・ambient 変化なく・依存（forest_deps）が今回変わっていない
  decl は elaborate/solve/force を丸ごと省き、キャッシュ（型と前回の eval IR id）から
  再適用して登録（`MemberEntry::with_ir`）。
- **early-cutoff**：再計算した decl の結果が前回と一致すれば「変わっていない」として
  下流を green のまま保つ。一致判定は **eval IR の正準 id の比較 O(1)**（hash-cons の
  おかげで「同じ構造＝同じ id」・旧実装の正規形 quote 比較より速く共有も壊さない）。
  `#[node]` 注釈の変更（意味に効かない）はこれで無害化される。
- **結果は full ビルドと一致**することを differential テスト（cook 系）が担保。
  `incr_stats()`（再計算数 / green 数）で最小性も検証する。

## cook.rs：Web の生きた module

`Cook` は worker が**単一保持**する言語状態です（main は状態を持たない純クライアント）。

```
Cook { module: WebModule, cache: BuildCache, dirty, ambient, built, last_errors, broken }
```

- **編集**は真実（WebModule）を変異して dirty を印すだけ。**無効化は編集メソッドごとに
  手書きせず、編集前後の forest を pretty text（decl 粒度・NodeId 非依存）で比較して導く**
  （`apply_edit`）——どの編集でも漏れない。再 lex/再パースはしない（安い）。
- **ambient**：fixity / nominal / instance / enum / import の変化は精密な依存に乗らない
  （暗黙の型文脈・パース文脈）ので、粗く全再計算。opaque はメンバを足すだけなので
  ambient ではない。
- **query**（eval/diagnose/workspace/graph）は `Cook::build` で最小再計算してから読む
  （旧名 ensure_built——build のログ・計時を出す実体に合わせて改名。modules.rs の
  self-populating `ensure_built` は別物で健在）。
  1 フレームの複数 query は `built` フラグで **1 回のビルドを共有**。
- **`set_source` も `apply_edit` 経由**です（冷たい「キャッシュ全捨て」は廃止）。全文が
  来ても前後の decl text 差分で dirty が出るので、full setSource 後の hold swap 判定などが
  「本当に変わった decl」を見られる（[dev/stream](dev/stream.md) の編集検知）。
- **broken**：`set_source` の parse 失敗は module を前のまま触らず broken 印だけ立て、
  eval はそれを返す（**stale な旧値を返さない**・§6）。一方、**編集メソッドの失敗は
  broken にしない**（module は前の有効なまま＝他の decl の eval を毒さない）。
  過去に「失敗した編集が Cook を broken にして全 eval を毒す」バグがあった——
  この 2 つの失敗モードの区別は意図的です。
- `eval_typed(node)`：per-node 評価。node に紐づくビルドエラー・eval エラーチャネル・
  未解決メタ（ホール）はすべて honest に Err。

## forest（構造編集の core 側）

Workspace のノード＝トップレベル宣言。宣言間参照は `Member` **名前**なので、削除・挿入・
並べ替えは**純粋なリスト操作**で de Bruijn を壊しません。消した宣言への参照は
「未解決の名前」としてビルドが正直に報告します。依存辺（`forest_deps`）・宣言名集合
（`module_declared`）・ローカル fixity（`local_fixities`）は、それぞれ**唯一の導出関数**に
まとまっています（散在 scan をしない）。

## 落とし穴まとめ

- zonk の位相（solve 後 1 箇所）を崩さない（前述）。
- build_decls に渡した decls を clone しない（hover join が静かに外れる・前述）。
- BUILDING への push はパース前（import 循環検出のため・前述）。
- member_origin が自モジュールを解決するとき self-populating が自己ディスクビルドに
  再入しないよう、自モジュールは parser の declared 集合で解決する（過去に再入バグ）。
