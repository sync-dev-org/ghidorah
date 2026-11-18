# ブラウザと program から mixer を操作する

遠隔の制御の application `ghidorah-control` は、ghidorah の mixer と FastAPI の server を同じ process で動かします。利用者は、ブラウザの簡易 UI と REST で parameter を変え、WebSocket で音の大きさと値の変化を受け取れます。

## 起動する

利用者は [install の手順](install.md) に従って、同じ Release の `ghidorah` と `ghidorah_control` の wheel を Python 3.12 の環境に入れます。下の YAML の例を `mixer.yaml` に保存し、その環境で実行してください。

```sh
python -m ghidorah_control mixer.yaml
```

起動時に application が YAML から mixer を作って開始し、Ctrl-C で mixer を止めて破棄します。server の process は一つで動かしてください。

ブラウザで `http://127.0.0.1:8000/` を開きます。別の host から操作する時は待ち受ける address を指定します。

```sh
python -m ghidorah_control mixer.yaml --host 0.0.0.0 --port 8080 --interval-ms 100
```

既定の address は `127.0.0.1`、port は `8000`、WebSocket の読取りの間隔は `100` ms です。`--interval-ms` は有限の正の数を受け付けます。application は認証を持たないため、利用者は制御を許す network の範囲で application を守ってください。

## insert を持つ設定の例

次の YAML は、network の入口から strip と bus を経由して network の出口へ音を送ります。strip と bus はそれぞれ EQ と dynamics を持ち、insert の名前を省き、異なる親で同じ自動名を使っています。

```yaml
defaults: {sample_rate: 48000, buffer_ms: 40}
clock: os
inputs:
  incoming: {receive: {address: 127.0.0.1, port: 7000}, channels: 2, format: float32}
outputs:
  outgoing: {send: {address: 127.0.0.1, port: 7001}, channels: 2, format: float32}
strips:
  voice:
    input: incoming
    channels: 2
    format: stereo
    pan: 0
    inserts:
      - kind: eq
        high_pass: {enabled: true, frequency_hz: 80}
      - kind: dynamics
        compressor: {enabled: true, threshold_dbfs: -18, ratio: 4, attack_ms: 10, release_ms: 100}
buses:
  main:
    channels: 2
    format: stereo
    output: outgoing
    output_channels: [0, 1]
    inserts:
      - {kind: eq}
      - kind: dynamics
        limiter: {enabled: true, ceiling_dbfs: -1, release_ms: 50}
sends:
  voice_to_main: {from: {strip: voice}, to: main}
```

名前を省くと、insert は `eq`、`dynamics` のような自動名を持ちます。同じ親の二つ目の EQ は `eq-2` です。明示名との重なりと種類ごとの数え方は [mixer の文書](mixer.md) に記載しています。

EQ と dynamics の帯域、stage、値の範囲と音の処理の順は、[mixer の文書](mixer.md) を参照してください。

mono / stereo の strip は `pan` を -100〜100 で受け、stereo bus への音の左右の位置を変えます。省略時は中央の 0 です。

### mono、stereo、multi-mono と PAN を試す

利用者は次の YAML を `demo.yaml` に保存し、`python -m ghidorah_control demo.yaml` で起動できます。MIC は AUX へ送り、AUX は自分の出口へ渡すとともに MAIN へ送ります。MAIN の出口は localhost の UDP port 7001、AUX の出口は port 7002 です。

```yaml
defaults: {sample_rate: 48000, buffer_ms: 40}
clock: os
inputs:
  incoming: {receive: {address: 127.0.0.1, port: 0}, channels: 2, format: float32}
outputs:
  outgoing: {send: {address: 127.0.0.1, port: 7001}, channels: 2, format: float32}
  auxiliary: {send: {address: 127.0.0.1, port: 7002}, channels: 2, format: float32}
strips:
  MIC:
    input: incoming
    channels: 1
    format: mono
    pan: -50
    inserts: [{kind: eq}, {kind: dynamics}, {kind: eq}]
  MUSIC: {input: incoming, channels: 2, format: stereo, pan: 25}
  STEMS: {input: incoming, channels: 2, format: multi-mono}
buses:
  AUX: {channels: 2, format: stereo, output: auxiliary, output_channels: [0, 1]}
  MAIN: {channels: 2, format: stereo, output: outgoing, output_channels: [0, 1]}
sends:
  mic: {from: {strip: MIC}, to: AUX, gain_db: -6}
  music: {from: {strip: MUSIC}, to: MAIN, gain_db: -6}
  stems: {from: {strip: STEMS}, to: MAIN, gain_db: -6}
  aux_to_main: {from: {bus: AUX}, to: MAIN}
```

## REST で読み書きする

| method | path | 操作 |
|---|---|---|
| GET | `/parameters` | entity と parameter の一覧、revision を読む |
| GET | 一覧の parameter の `path` | 一つの parameter を読む |
| PUT | 一覧の parameter の `path` | `{"value": 値}` で一つの値を変える |
| GET | `/sends` | 現在の send の定義の列を読む |
| PUT | `/sends` | send の定義の列をまとめて置き換える |
| GET | `/configuration` | 現在の構成と、宣言済みの入口・出口を読む |
| PATCH | `/configuration` | 名前を指定した構成の変更を一度に反映する |
| GET | `/statistics` | 外の入口と出口の統計と device の状態を読む |
| GET | `/configuration.yaml` | 現在の設定を ghidorah.yaml として download する |
| PUT | `/configuration.yaml` | UTF-8 の YAML を body で渡し、mixer を置き換える |

application は、native が返す `EntityInfo.path` と `ParameterInfo.path` の先頭に `/` を付けて HTTP の path にします。名前に日本語、空白、`/`、`%` が含まれる場合も、一覧の `path` は encode 済みです。program はその `path` をそのまま URL に使ってください。

entity の行は、strip の入口の名前 `input`、最初の ch `first_channel`、ch の数 `channels` と、bus の出口の名前 `output`、ch ごとの出口への割り当て `output_channels` を持ちます。出口を持たない bus は `output: ""` と `output_channels: []` を返します。該当しない欄は空文字、0、空配列です。`channels` は strip の入力範囲にだけ使い、bus の行では 0 です。REST と WebSocket は同じ欄を返し、動いている構成を変えた後は、library の一覧から新しい値を読みます。名前の欄は元の文字列で、`path` だけを percent encode します。

strip と bus の entity は `format` に `"mono"`、`"stereo"`、`"multi-mono"` を返します。send と insert は `format: ""` です。形式だけが変わった時も entity の差分を送り、PAN が使えなくなった strip には PAN の除去を通知します。

entity と parameter の行には整数の `order` もあります。一覧自体は path 順を保ちます。同じ親の insert を `order` で並べると、insert 枠の処理順になります。parameter は同じ entity の中で `order` を比較し、EQ の帯域や dynamics の stage の処理順に並べられます。番号はそれぞれ 0 から数え、異なる entity の parameter の番号は直接比較しません。insert の並びだけを変えた時も、WebSocket は変更した insert の entity と新しい番号を差分で送ります。

strip の parameter の順は input gain が 0、fader が 1、mute が 2、PAN がある時は PAN が 3 です。bus は fader が 0、mute が 1 です。send は全体の gain が 0、続いて送り元と送り先の ch の順に ch の組の gain が並びます。insert 以外の entity の `order` は 0 で、entity 同士の処理順を示しません。EQ は high pass、low shelf、peak1、peak2、high shelf の順に、帯域内で enabled、frequency、gain、Q を並べ、持たない値を省きます。dynamics は expander、compressor、makeup、limiter の順です。各 stage は enabled、threshold、ratio、gate、attack、release のうち持つ値を並べ、makeup は enabled、gain、limiter は enabled、ceiling、release を並べます。

上の設定では、次のような path が一覧に現れます。

```text
/strips/voice/fader_db
/strips/voice/pan
/buses/main/mute
/sends/voice_to_main/channels/0/0/gain_db
/strips/voice/inserts/eq/high_pass/frequency_hz
/buses/main/inserts/dynamics/compressor/ratio
```

send の ch の組は設定された組だけが一覧に現れます。上の設定の既定の対応では、`channels/0/0/gain_db` と `channels/1/1/gain_db` が使えます。

```sh
curl http://127.0.0.1:8000/parameters
curl -X PUT http://127.0.0.1:8000/strips/voice/inserts/eq/high_pass/frequency_hz \
  -H 'Content-Type: application/json' -d '{"value": 120}'
```

parameter の一覧は、`entity_kind`、`entity_name`、`name`、`path`、`type`、`minimum`、`maximum`、`minimum_inclusive`、`maximum_inclusive`、`value`、`removed`、`revision` を持ちます。`minimum` が `null` の時は下限が無く、`maximum` が `null` の時は上限がありません。insert の entity は `parent_kind`、`parent_name`、`insert_kind` も持ちます。

`bool` の値は JSON の `true` または `false`、`real` の値は有限の数を渡します。型と範囲の違う値や不正な body は HTTP 422 と理由を返し、知らない parameter の path は 404 を返します。拒否された変更では値と revision は変わりません。

`PUT /sends` は、一度に渡した send の列を一つの変更として扱います。空の列 `[]` を渡すと全ての send を外します。`GET /sends` で得た以前の定義を渡すと接続を戻せます。定義は `name`、`source_kind` (`strip` または `bus`)、`source`、`bus`、`gain_db`、`channels` を持ちます。`channels` の各要素は `source_channel`、`destination_channel`、`gain_db` です。`channels: null` は native の既定の ch の対応を使い、空の列は ch をつなぎません。

### 動いている構成を変える

`GET /configuration` は `generation`、`sample_rate`、`clock`、`strips`、`buses`、`sends`、`inputs`、`outputs` を返します。`generation` は運転を識別する文字列で、mixer の置き換えが成功するたびに変わります。同じ YAML の読み直しでも変わります。`clock` は選ばれた `kind` (`os`、`input`、`output`) と対象の `name` (OS なら空文字) を持ちます。

入口と出口は未使用の物も含み、名前 `name`、種類 `kind` (`network` または `device`)、ch の数 `channels`、`sample_rate`、`sample_format`、`buffer_ms`、`max_datagram_bytes` を持ちます。network は `address`、宣言した `port`、実際の `receive_port` (出口では 0) を、device は `device` の名前と `mode` を持ちます。device の ch 数と receive_port は観測値で、変更する設定ではありません。

この口の bus の `channels` は、bus 自身の ch の数です。strip は `name`、`input`、`first_channel`、`channels`、`format`、`inserts` を、bus は `name`、`channels`、`format`、`output`、`output_channels`、`inserts` を持ちます。insert の列は処理順の `name` と `kind` を持ち、parameter の値は `/parameters` で読みます。send の欄は `/sends` と同じですが、native の既定の ch の対応を使う send は `channels: null` のまま返します。

`PATCH /configuration` の body は、`strips`、`buses`、`sends` の object を必要な物だけ持ちます。その中の key は URL encode していない元の名前です。値の object は追加または変更を、`null` はその名前の削除を指します。指定しない項目はそのまま保ちます。

```json
{"buses": {
  "main": {"output": "auxiliary", "output_channels": [1, 0]},
  "monitor": {"output": ""}
}}
```

strip は `input`、`first_channel`、`channels`、`format`、`inserts` を、bus は `channels`、`format`、`output`、`output_channels`、`inserts` を変えられます。新しい strip は `input` と `channels`、新しい bus は `channels` を指定します。新しい物の省略した欄は native の既定値を使います。`output: ""` は出口を外し、`output_channels` を省略した時はその割り当ても空にします。

`inserts` は枠の中の全ての insert を処理順に並べる列です。要素は `kind` (`eq` または `dynamics`) と任意の `name` を持ちます。今と同じ名前・種類の insert は今の parameter を保ち、新しい insert は既定値を使います。名前を省いた時は native が名前を決めます。send は `source_kind`、`source`、`bus`、`gain_db`、`channels` を変えられ、新しい send は最初の三つを指定します。send の `channels: null` は既定の ch の対応を使い、`[]` は ch をつなぎません。それ以外の欄の明示的な `null` は拒否します。strip や bus を削除する時は、それを指す send の変更や削除も同じ body に入れます。

strip、bus、send だけの変更では、application は native から現在の構成のコピーを取り、コピーだけを編集して、`Mixer.reconfigure()` を一度呼びます。宣言済みの外の入口と出口、transport、device、clock、保持する parameter の現在値は引き継ぎ、YAML を読み直しません。

成功は HTTP 200 で、`GET /configuration` と同じ形の変更後の構成を返します。不正な型や範囲、存在しない名前、bus の輪、出口の ch の重複、構成を反映できない理由がある時は HTTP 422 と `detail` を返します。一度に受けた変更の一部だけを反映することはなく、拒否した時は構成と revision を保ちます。受理した後の entity は従来の WebSocket でも届きます。[mixer の文書](mixer.md) は、所有権を持つ `MixerConfig` のコピーを Python から直接編集する方法も説明します。

### 外の口、sample rate、clock を変える

同じ PATCH の body に `inputs`、`outputs`、`sample_rate`、`clock` も渡せます。外の口への変更、または sample_rate / clock の指定がある時は `Mixer.replace()` で新しい運転を作り、作り直す間は音が止まります。同じ body の strip、bus、send の変更もまとめて扱うため、口の宣言とそれを指す名前を一度に変えられます。

`inputs` と `outputs` は元の名前を key にし、値の `null` は削除、object は追加・編集です。共通の欄は `kind`、`sample_rate`、`sample_format`、`buffer_ms`、`max_datagram_bytes`。network はさらに `address`、`port`、`channels`、device は `device`、`mode` を受けます。新しい口は種類、sample の形式と、network なら address / port / channels、device なら device の名前を指定します。省略時の値は mixer の rate、buffer_ms 40、max_datagram_bytes 1200、device の exclusive mode です。既存の共通の設定は種類を変えても引き継ぎます。device の channels と種類に合わない欄は拒否します。object の中の name と receive_port は受け付けません。

clock は `{"kind":"os"}`、`{"kind":"input","name":"capture"}`、`{"kind":"output","name":"playback"}`、`{"kind":"default"}` の形です。input / output の clock は宣言済みの device を指します。os / default に名前は付けません。network の出口の rate は mixer の rate と一致する必要があるため、mixer の rate を変える時は出口の変更も同じ body に入れます。保持する insert の値も新しい rate で検査します。

```json
{"sample_rate":96000,"clock":{"kind":"os"},
 "outputs":{"outgoing":{"sample_rate":96000}},
 "inputs":{"extra":{"kind":"network","address":"127.0.0.1","port":0,
                    "channels":2,"sample_rate":48000,"sample_format":"float32"}}}
```

成功は HTTP 200 と新しい構成・generation を返します。拒否や新しい口の open / start の失敗は HTTP 422 と detail を返し、元の構成と generation を保ちます。

### YAML の download と読み込み

`GET /configuration.yaml` は現在の設定を `application/yaml; charset=utf-8` で返し、`Content-Disposition: attachment; filename="ghidorah.yaml"` を付けます。確定した insert の名前と設定値、宣言した受信 port、選ばれた clock を含み、計測値や device の ch 数は含みません。

`PUT /configuration.yaml` は一つの UTF-8 の YAML 文書を body で受けます。Content-Type は `application/yaml` を指定してください。成功すると HTTP 200 と新しい構成・generation を返します。不正な UTF-8 / YAML、設定の拒否、口を開けない・開始できない場合は HTTP 422 と detail を返し、元の運転を保ちます。読み込んだ YAML は起動時の設定 file へ上書きしません。

### 外の口の統計

`GET /statistics` は generation と inputs / outputs の列を返します。各行は name、累計の missing_frames と late_packets、jitter_margin_ms、device と clock_device を持ちます。device と clock_device は unavailable、name、reason を持ちます。late_packets は network の入口の値で、他の口では 0 です。

REST の読取りは累計を消費しません。受け手は同じ generation の前回の累計を引いて増分を求めます。generation が変わった時、口が消えて戻った時、または累計が減った時は、前回の値を使わず新しく数え始めます。WebSocket は接続ごとの増分を直接返します。

## WebSocket で受け取る

同じ host と port の `/ws` へ接続します。全ての message は generation を持ちます。最初の `type: "parameters"` は `reset: true` と、`/parameters` と同じ全体の一覧を持ちます。置き換えが成功した後も、接続ごとに LevelReader を取り直し、統計の前回値を消して、全体の一覧を reset: true で送ります。受け手は既存の一覧を全て置き換えます。native の parameter の revision が変わらない場合も同じです。

それ以外の parameters message は `reset: false` で、その接続の前回の revision から変わった entity と parameter を載せます。`removed: true` の記録は、読む側の一覧から取り除きます。複数の変更が最新の値にまとまる場合があります。`type: "configuration"` は、接続時と構成が変わった時に GET /configuration と同じ全体の値を送ります。出口の無い bus の ch 数など、parameter の revision を変えない変更も含みます。

`type: "levels"` の message は、`levels` に strip、send、bus、output の ch ごとの測定を載せます。

```json
{"type":"levels","generation":"...","levels":[
  {"kind":"bus","name":"main","channel":0,
   "peak_dbfs":-12.0,"rms_dbfs":-15.0,"samples":4800,
   "exceeded_samples":0,"total_exceeded_samples":0}
]}
```

ch の番号は 0 から始まります。peak と RMS の単位は dBFS で、無音や sample が無い間の値は `null` です。sample の数と出口の超過数は、その接続の前回の読取りからの値です。`total_exceeded_samples` は mixer が動き始めてからの累計です。

接続ごとに LevelReader と revision を持つため、別の接続の読取りの間隔は互いの測定に影響しません。遅い接続の送信を待つ間も、mixer の音の処理と他の接続は動き続けます。

`type: "statistics"` は GET /statistics と同じ累計・状態を一定の間隔で送ります。各行にはさらに missing_frames_delta、late_packets_delta、reset があります。増分はその接続の前回の統計との差です。最初の読取り、generation の変更、累計の減少では reset を true にし、両方の増分を 0 にします。他の接続や REST の読取りはこの前回値を変えません。受け手は四種類の message を type と generation で区別してください。

## ブラウザで操作する

利用者は、上端の Add strip / Add bus で列を追加できます。strip と bus が無い状態でも同じ操作を使えます。各列の Edit は ch の数と音の形式を変え、strip では宣言済みの入口と最初の ch も選びます。Remove は、その列を指す send も一度の変更で外します。+ Inserts は処理順の一覧を開き、名前を省いた insert の追加、名前と種類の編集、上下への移動、削除を Apply で反映します。各 insert の key は従来どおり parameter の子の window を開きます。

各列の + Send は、その strip または bus からの send を追加します。送り元の種類と名前、送り先の bus、既定の ch の対応か明示的な ch の組を選べます。send の子の window は peak と RMS を示し、Edit send から接続、ch の組の増減、削除を操作できます。拒否された変更では子の window に理由が現れ、元の構成を保ちます。成功後は library の返す今の構成から描き直します。別の program が REST で変えた構成と値も、開いている画面に反映します。

画面は、同じ卓の中に strip、bus、出口の順で列を並べます。宣言済みの出口は、その出口を使う bus が無い時も一つの列として示します。出口の列は、上端に出口の名前を、本体に ch ごとの peak と RMS の縦のレベルメーターを示し、meter の目盛りを strip と bus の列に揃えます。出口の列は fader、mute、insert、send、PAN、input gain の control を持ちません。画面は、strip、bus、出口の三つのまとまりの間に、卓の外周と同じ金属の枠を描きます。

strip と bus の各列の Stats は、その入口または出口の統計を子の window に示します。欠けた frame と遅れて捨てた packet は前回の WebSocket の読取りからの増分と累計を区別し、揺れを吸収できる幅、device と clock の device の状態と理由も示します。device が止まっている間は列に Device stopped が現れ、復帰すると消えます。出口の無い bus は外の口の統計を持ちません。

External I/O は外の入口と出口の宣言の一覧、追加、編集、削除を受けます。実際の receive port と device の ch の数は観測値として示します。Mixer settings は sample rate と clock を変え、network の全ての出口の sample rate も同じ変更で mixer の値に揃えます。どちらの子の window も、反映の前に音が止まる事を示します。Download YAML は現在の設定を download し、Load YAML は file または貼り付けた YAML を読み込みます。読み込みの前にも音が止まる事を示し、拒否された時は元の構成のまま理由を示します。

簡易 UI は、mixer を物理の卓の ch strip の形で示します。strip、bus、出口の順に縦の列を並べます。strip と bus の各列は、上から下へ音の処理の順に、次の物を示します。

- LCD: 名前、音の形式 (MONO / ST / MULTI)、fader の値、保持した peak の値
- 音の入る所: strip では外の入口の名前と受ける ch の範囲を示す LCD、bus では送り元の strip と bus の名前を並べる LCD
- input gain の knob (strip だけ)
- insert の key
- PAN の knob (mono と stereo の strip だけ)
- mute の key
- 縦の fader と刷った目盛り、その横の ch ごとの縦の level meter
- この列から音を送る先を示す区画 (send の key を並べ、その下に bus の出口の名前と ch の割り当てを一行で示す)

画面は、send と出口の行が最も多く並ぶ列に、全ての列の下端の区画の高さを合わせます。出口の行は send の行と同じ高さ、文字の大きさ、配置を使い、`OUT auxiliary ch 0–1` の一行で示します。出口を割り当てた時は、send と同じく接続があることを lamp の点灯で示します。出口を持たない bus も `No output` の行を持ち、その lamp は消灯します。どちらの行も、押すと出口の子の window を開きます。長い名前や ch の割り当ては省略記号で切り、tooltip と読み上げ用の label に全文を残します。

出口の子の window は、現在の出口の名前の全文と ch の割り当て、宣言済みの出口の選択欄、bus の ch ごとの割り当てを持ちます。出口と ch を選んで Apply を押すと構成を変え、No output を選んで Apply を押すと出口を外します。変更が拒否された時は構成を保ち、子の window の中に理由を示します。表には、選択した出口の ch ごとの、前回の WebSocket の読取りから範囲を超えた sample の数と累計を示し、受け取る測定値で更新します。

PAN の区画は insert と MUTE の間に置きます。PAN は GAIN と同じ作りの knob を使い、中央の目盛りと `L50`、`C`、`R100` の値で位置を示します。double-click で中央へ戻ります。PAN を持たない列は同じ区画を空けます。

mute は fader の後で働きますが、操作しやすいように mute の key を fader の上に置きます。各列は全ての区画を持ち、区画は同じ余白と上端の細い線を持ちます。そのため、区画の線は全ての列で同じ高さに揃い、列が使わない区画は線に囲まれた空きになります。

画面の ch の番号は、YAML と API と同じく 0 から数えます。出口の割り当ての `0→3` は、bus の ch 0 の音を出口の ch 3 へ渡すことを示します。strip の meter は input gain の後で insert の前の音を、bus の meter は insert、fader、mute の後の音を測ります。meter は、RMS を濃い棒、peak を薄い棒、保持した peak を線で示します。LCD の中の細い meter は最も大きい ch の peak を、strip と bus の LCD の下端の横の棒は fader の位置を示します。

画面は control surface の見た目を持ちます。strip は金属の枠の中の一枚の暗い面に並び、列と列の間を細い線で分けます。LCD は黒い帯の中に並びます。key は小さな四角い lamp で、文字は刷った形で示します。insert の key は、その insert の stage か帯域が一つでも on の時に光ります。send の key は、つながっている間は光ります。mute の key は、mute が掛かっている時に光ります。fader のつまみは銀色の鞍の形で、陰影は断面の形と上からの光から求めています。目盛りの数は、卓と同じく符号を付けずに刷ります。0 より上の目盛りは音を大きくし、下の目盛りは音を小さくします。

insert の key は種類 (`EQ`、`Dynamics`) を大きく示し、種類の既定名と違う名前だけを小さく添えます。同じ種類の二つ目の EQ では `eq-2` が小さく現れます。insert の dialog も種類を大きく、名前を小さく示します。送る側の列の下には、送り先の bus の名前を `→ bus` の形で、send の level と一緒に示します。受ける bus の列の上には、strip の入口と同じ作りの LCD に送り元の名前を並べます。bus の送り元の LCD には send の key を置きません。LCD は二行に収まる名前を示し、送り元の種類と収まらない名前を含む全体は tooltip と読み上げ用の label に保持します。利用者が欄を押すと、場所を取る設定を示す dialog が開きます。insert の dialog は parameter を stage や帯域ごとにまとめて示し、send の dialog は send の level と ch ごとの gain を示します。Escape、閉じる button、dialog の外の click で dialog は閉じます。parameter の名前は、単位を分けた語で示します (例: `compressor/attack_ms` は `attack` と `ms`)。

簡易 UI は、列、欄、dialog を一覧から作ります。動いている間に strip、bus、send、insert が増減すると、列と欄も増減します。knob、fader、button には strip と bus の決まった parameter (`input_gain_db`、`fader_db`、`mute`、`pan`) を使い、他の parameter にも一覧の型に合う入力を作ります。EQ の帯域や dynamics の stage を application が決め打ちすることはありません。簡易 UI は、insert とその parameter を一覧の `order` に従って並べます。

fader と knob は、縦に drag するか、矢印の key と Page Up / Page Down で動かします。Shift を押すと細かく動き、double-click で 0 dB になります。drag は pointer を押した位置からの移動の量で値を変えるため、click だけでは値は変わりません。fader は +10 dB から -90 dB、input gain の knob は -24 dB から +24 dB の間を動きます。数の入力欄は parameter の範囲の任意の値を受け付けます。drag している間、簡易 UI は最新の位置を送り、受け取った値でその control を動かしません。狭い画面では、列を横に scroll して見ます。画面の他の値は WebSocket の変化に従って更新します。

application は HTML、JavaScript、CSS を同じ server から配信します。外部の CDN や JavaScript の build の準備は要りません。
