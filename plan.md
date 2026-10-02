# frdt プロジェクト引き継ぎ

新しいリポジトリ `frdt` を作る。既存の `brson.js` の改造ではない。
決定済みの前提を再検討せず、この文書を仕様の起点にする。

## このライブラリがすること

Resonite のオブジェクトファイル（拡張子 `.brson`、中身は FrDT）を、
JSON へ全部展開せずに操作する Rust ライブラリ。

用途は3つ。

1. 編集。複数の brson を入れ子に結合して新しいオブジェクトを作る。値を書き換える。
2. 分析。特定コンポーネントの数、スロットの数を数える。Dynamic Variable を抽出する。
3. 保存。すぐ使える形で残す。Resonite が開く `.brson` と、次回ツールがすぐ開く作業用データ。

## 名前

リポジトリ名、クレート名、npm パッケージ名は `frdt`。

ファイル先頭の4バイトがマジックナンバーで、ASCII にするとこの4文字になる。

```text
46 72 44 54  00 00 00 00  03
F  r  D  T   予約4バイト  圧縮種別
```

- `0x46` = `F`
- `0x72` = `r`
- `0x44` = `D`
- `0x54` = `T`

Resonite のエンジンは FrooxEngine、シリアライズ木は DataTree。
`FrDT` はその略として読む。仕様書に正式名称の展開が書いてあるわけではなく、
ファイルに入っているマジックがこの4文字である。
2026-10-02 時点で npm と crates.io に `frdt` は無かった。

`brson` は拡張子側の通称。npm の `brson.js` 1.0.5 が既にあるので、新しいライブラリを `brson` と呼ばない。

## 言語と連携

変換エンジンは Rust クレート1つ。実装を Go と Rust に分けない。

- 今のプロダクトは JS のまま。そこは書き換えない。
- JS からは napi-rs のネイティブアドオンで呼ぶ。`Document` は Rust オブジェクトへの参照。JS に返すのは件数、文字列、小さな抽出結果、書き出した `Buffer` だけ。スロット木を JS オブジェクトにしない。
- 将来、軽量コンテナ上の変換 HTTP API を作る。同じクレートを axum で包んだ Rust バイナリにする。コンテナに Node は入れない。
- Go でサーバーを書く案は、変換エンジンも Go で書き直す場合だけ。Rust エンジンを cgo で Go から呼ばない。

JS 側の呼び出しイメージ:

```js
import { open } from "frdt";

const avatar = open(avatarBytes);
const hat = open(hatBytes);
const head = avatar.root().findSlot((s) => s.name() === "Head");
head.append(hat.root());
const out = avatar.writeBrson(); // Node の Buffer
```

既存 JS で置き換えるのは、`brson.js` の `DeCompress` / `Compress` を呼んでいる箇所。

## 既存 brson.js の実態

リポジトリは薄いラッパ。実装は `src/index.ts` の2関数。

- `Compress(json: string)`: `JSON.parse` → `BSON.serialize` → `zlib.brotliCompressSync` → 9バイトヘッダを前置
- `DeCompress(buffer)`: 先頭9バイトを捨てる → `brotliDecompressSync` → `BSON.deserialize` → `JSON.stringify`

ヘッダは次で固定されている。

```js
Buffer.from([0x46, 0x72, 0x44, 0x54, 0x00, 0x00, 0x00, 0x00, 0x03])
```

依存は `bson-fork`（`rassi0429/js-bson`、MongoDB の bson 6.1.0 のフォーク）。
公式 bson は内部バッファが 17MB（`1024 * 1024 * 17`）で大きな文書が失敗するため、フォークに替わっている。
フォークは `MAXSIZE = 1024 * 1024 * 128` で、import 時に 128MB を常駐確保する。

Node の `brotliCompressSync` のデフォルトは品質 11、ウィンドウ 22。品質 11 は最も遅い。
書き出しは品質 4〜6 を使う。

`brson.js` は圧縮種別 `0x03`（Brotli）だけ。ヘッダ9バイト固定。
圧縮種別は可変長バイトで、`0x03` は1バイトに収まるので9バイトで合っている。
`0x01` は LZ4 frame、`0x02` は LZMA（`.lz4bson` / `.7zbson`）。v1 では Brotli 以外を拒否してよい。

履歴:

- 最初は公式 `bson`
- `fix: bson max size` で `bson-fork` に切り替え
- 2024-10-13 の `update bson for NaN, Infinity`（brson.js 1.0.5）は `src/index.ts` を変更していない。`bson-fork` を `^6.1.1` に上げただけ
- 同じ日のフォーク側コミット `feat: support double Inf, NaN`（`23c7072`）と `max size to 128MB`（`720d1ee`）が本体
- 2025-07 にフォークへ `stringifyStream` が追加された。brson.js は使っていない

## Infinity が壊れていた理由

JSON は `Infinity` / `NaN` を数値として持てない。

- `JSON.stringify(Infinity)` は JSON の `null` になる
- `JSON.parse('{"x":Infinity}')` は `SyntaxError`
- BSON の double は IEEE 754 なので無限大を保持できる。`serializeNumber` は有限な安全整数を int32、それ以外を `setFloat64` で書く。JS の数値 `Infinity` を渡せば double の無限大になる

作者の修正はデシリアライズだけ。

```ts
value = dataview.getFloat64(index, true);
if (isNaN(value)) value = "NaN";
if (value === Infinity) value = "Infinity";
if (value === -Infinity) value = "-Infinity";
```

公開物 `bson-fork` 6.1.1〜6.1.5 の `lib/bson.cjs` に入っている。
読み出すと数値ではなく文字列 `"Infinity"` になる。`JSON.stringify` は `"Infinity"` という JSON 文字列にするので、昔の「`null` になる」症状は止まる。

残る問題:

- 数値の無限大には戻らない
- その文字列を `Compress` すると BSON string（型 0x02）になり、元の double（型 0x01）に戻らない
- 呼び出し側が先に `JSON.stringify` すると、ライブラリに届く前に `null` になる
- `promoteValues: false` だと `Double` のままになり、`Double.toJSON()` が生の数値を返す。`JSON.stringify` は再び `null` にする。文字列化はデフォルトの `promoteValues: true` だけ

frdt では double の非有限値を IEEE のまま BSON に置く。JSON 文字列を中間形式にしない。

`stringifyStream` は、全オブジェクト化の後に文字列化するだけなので、frdt の方式の代わりにしない。

## メモリの結論

壁の原因は「バイト列を保持すること」ではなく、「全ノードを JS オブジェクトと JSON 文字列にすること」。
`DeCompress` が1本の文字列を返す API 自体が全展開を約束している。

DataTree は小さな辞書が多いので、V8 のオブジェクトは BSON の数倍から十数倍になる。
経路上に、圧縮バイト、展開 BSON、オブジェクト木、JSON 文字列、128MB の常駐バッファが重なる。
BSON の walk は JS 側で、API は同期なのでイベントループも止まる。
非圧縮 BSON が 128MB を超えると、フォークのバッファでは書けない。

Brotli と `JSON.parse` / `JSON.stringify` はネイティブ実装。遅い本体は、品質 11 の圧縮と、オブジェクト木の展開である。

方式は次の2段。

- 分析だけなら、Brotli を流しながら1回歩き、件数と抽出結果だけ残して捨てる。ピークは入れ子の深さと、いま読んでいるフィールドと、型テーブル。巨大な1フィールドが下限になる。
- 編集と再解析があるなら、展開した BSON を1枚と、ノードごとの ID・バイト位置・長さ・型添字の索引を持つ。開いたスロットだけ読む。保存して次回の入力にする。メモリはおおよそ非圧縮 BSON の1倍と索引。

純ストリームだけでは、後から触る編集に戻れない。Brotli は途中へ安くシークできない。
編集用の実体は「展開 BSON 1枚 + 索引」。
長さプレフィックスがあるので、開いていない部分木は中をパースせず飛ばせる。

ハンドルはコピーではなくポインタ。

```ts
type SlotHandle = { doc: Document; offset: number }
type ComponentHandle = { doc: Document; offset: number; typeIndex: number }
```

`name()` や `get` はその位置のフィールドを1つ読む。`set` はそこを書き換え、祖先の BSON 長を更新する。
変わらない長い文字列やバイナリは、元バッファの範囲をコピーする。

パーサだけ WASM にして結果を全 JS オブジェクトにすると、壁が戻る。
見えているフィールドだけを JS に出す。

## ファイル形式

参照: https://wiki.resonite.com/User:Kisaragi_marine/brson

形式は変わることがある、と wiki 自身が書いている。実ファイルのフィクスチャで確認する。
以下は wiki の理解であり、フィールドの入れ子は実データで突き合わせる。

圧縮を解くと BSON。ルートオブジェクトのフィールド:

- `VersionNumber`
- `FeatureFlags`（wiki にある例: `ColorManagement`、`ResetGUID`、`ProtoFlux`、`TEXTURE_QUALITY`、`TypeManagement`。意味は未確定のものがある）
- `Types`: 使われているコンポーネント型の文字列配列。CLI の正準名。例: `[FrooxEngine]FrooxEngine.FinalIK.VRIKAvatar`
- `TypeVersions`: キーは `Types` にある文字列、値は数値。全コンポーネントが入っているとは限らない
- `Object`: Slot
- `Asset`: IAssetProvider の配列

正準名は、アセンブリ名を角括弧で囲み、その直後に完全修飾名を続ける。型引数があればアリティと型パラメータを続ける。

`PackageNodeIdentifier` はパッケージ内で一意な不透明 ID。

`UniquePtr<T>` は次の形。

- `ID`: `PackageNodeIdentifier`
- `Data`: T

`WorldElementRef` は、対応する `PackageNodeIdentifier` のノードを指す参照。
wiki では `UniquePtr(PackageNodeIdentifier)` のエイリアスとされている。実レイアウトはフィクスチャで確認する。

`ComponentTableIndex` は `Types` への非負整数の添字。コンポーネントの `Type` フィールドがこれ。
`Types[42]` がそのコンポーネントの型名。`Types` は `Object` より前に来るので、単一パスで型テーブルを覚えてからコンポーネントを解釈できる。

Slot（`Object` は常に Slot から始まる）:

- `ID`: `PackageNodeIdentifier`
- `Components`: `UniquePtr(Array(IComponent))`
- `ParentReference`: `PackageNodeIdentifier`
- `Children`: `Array(Slot)`
- `Name` (string)
- `Tag` (string?)
- `Position` (float3)
- `Rotation` (floatQ)
- `Scale` (float3)
- `OrderOffset` (long)

親子階層自体は `Children` の入れ子なので、文書の順番で出力できる。

IComponent:

- `Type`: `ComponentTableIndex`
- `Data`: コンポーネント依存

`[NameOverride]` のフィールドは別名で出る。`[NonPersistent]` は出ない。
wiki では、`Sync<T>` は T に解決できる `WorldElementRef`、`SyncList<T>` は配列に解決できる `WorldElementRef`、`SyncRef<T>` は T に解決できる `WorldElementRef` とされている。実バイトで確認する。

## 結合で必ずやること

2つの brson をバイト列のまま連結してはいけない。

`append` の中で対応表を作り、部分木を書き出す。

- 子文書の全 `PackageNodeIdentifier` を、結合先で一意な新しい ID にする
- 子のルートの `ParentReference` を、くっつけ先スロットの ID にする
- 子の `Types[i]` を結合先 `Types` へ足し、コンポーネントの `Type` 添字を新しい番号に振り直す。これを怠ると別の型を指す。例: 帽子側の `Types[0] = Spinner` で、アバター側に型が既に2つあるなら、書き出した `Type` は `0` ではなく `2`
- 部分木内の `WorldElementRef` も同じ ID 対応表を通す
- `Asset`、`TypeVersions`、`FeatureFlags` もマージする
- 親の `Children` に書き出した BSON を挿入し、祖先の文書長を更新する
- 変わらない長い文字列やバイナリは、元バッファの範囲をそのままコピーする

参照はバイト位置ではなく ID なので、前方のノードを指していても対応表があれば書き出し時に解決できる。
結合先の頭以外の部分は、元のバイト列のまま残してよい。

呼び出し側のイメージ:

```rust
let mut avatar = Document::open(&avatar_bytes)?;
let hat = Document::open(&hat_bytes)?;
let head = avatar.root().find_slot(|s| s.name()? == "Head")?;
let grafted = avatar.append(head, hat.root())?;
grafted.set("Scale", float3(2.0, 2.0, 2.0))?;
avatar.write_brson(&mut file)?;
```

`append` の戻り値が、結合先文書の中にできたスロット。元の `hat` のハンドルは以降使わない。
`find_slot` は名前だけを読む。`walk_components` は対象の部分木だけを見る。

単体の値変更は、索引で位置を見つけ、そのフィールドを差し替え、祖先の長さを更新する。

`append` の内部イメージ:

```ts
append(child: SlotHandle): SlotHandle {
  const idMap = new Map<string, string>()
  for (const node of child.walkIndex()) {
    idMap.set(node.id, newId())
  }
  const typeMap = this.doc.mergeTypes(child.doc)
  const bytes = rewriteSlot(child, idMap, typeMap)
  const offset = insertIntoChildren(this, bytes)
  return { doc: this.doc, offset }
}
```

## 分析

ストリームのまま1回歩く。結果以外は残さない。

- Slot 数は `Children` の入れ子を数える
- コンポーネント数は、先に現れる `Types` を覚えて `Type` 添字を数える
- Dynamic Variable は、型名に `DynamicValueVariable`、`DynamicReferenceVariable`、`DynamicVariableSpace` を含むものを拾い、`Data` から変数名と値を控える。フィールド名（`VariableName`、`Value`、`Reference`、`SpaceName` を想定）は実ファイルで確定する

同じファイルを何度も調べるなら、展開 BSON と索引を保存し、2回目以降はそれを開く。

## 保存

- Resonite 用: 索引から BSON を流し、Brotli 品質 4〜6 で圧縮し、`FrDT` + 予約4バイト + `0x03` を付ける。ファイルは少し大きくなり、圧縮時間は大きく減る
- ツール用: 展開済み BSON と索引をそのまま保存する。次の分析と結合はここから始める
- 人が読む JSON は、見ている部分の抜粋だけにする

## 実装の層

1. コンテナ。FrDT ヘッダと Brotli。LZ4 と LZMA は後回し
2. BSON。カーソル、部分木コピー、長さの付け直し。文書と配列は先頭4バイトが自分の長さ
3. DataTree。`Types`、Slot、コンポーネント、参照 ID
4. 操作。数える、変数を抜く、結合する、フィールドを書く、書き出す

FrooxEngine の全コンポーネント型は写さない。触るものから足す。形式の変更に追随しやすくするため。

## v1 でやること

フィクスチャで次を確認してから API を広げる。

- スロット数とコンポーネント数
- Dynamic Variable の抽出
- `Infinity` / `-Infinity` / `NaN` が double のまま往復する
- 2文書の結合で、型添字と ID が衝突せず、`ParentReference` が結合先を指す

結合が一番壊れやすい。このテストを先に置く。

## v1 でやらないこと

- JSON を入出力の本体にすること
- `bson-fork` と `brson.js` に依存すること
- LZ4 / LZMA
- 全コンポーネントのスキーマ
- GUI
- 公式 bson の 17MB バッファや、フォークの 128MB 常駐バッファを持ち込むこと

## 関連リンク

- brson.js: https://github.com/rassi0429/brson.js （npm `brson.js` 1.0.5）
- bson-fork: https://github.com/rassi0429/js-bson
- 形式メモ: https://wiki.resonite.com/User:Kisaragi_marine/brson
- 対応形式: https://wiki.resonite.com/Supported_Formats
- C# の変換ツール（エンジン同梱）: https://github.com/Lexevolution/Resonite-DataTree-Converter
- Rust の展開のみ: https://github.com/KisaragiEffective/frooxengine-data-tree
