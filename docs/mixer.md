# C と C++ から mixer を使う

mixer は外の入口から strip が選んだ ch を受け、send で bus へ送り、bus の ch を出口へ割り当てます。bus から別の bus への send も作れます。bus の輪と、同じ出口の同じ ch を二つの bus に割り当てる構成は拒否します。

## YAML で組み立てる

```yaml
defaults: {sample_rate: 48000, buffer_ms: 40, mode: exclusive}
clock: os
inputs:
  incoming: {receive: {address: 0.0.0.0, port: 7000}, channels: 2, format: float32}
outputs:
  outgoing: {send: {address: 127.0.0.1, port: 7001}, channels: 2, format: pcm24}
strips:
  voice:
    input: incoming
    first_channel: 0
    channels: 2
    format: stereo
    pan: 0
    input_gain_db: 0
    fader_db: 0
    mute: false
    inserts:
      - kind: eq
        high_pass: {enabled: true, frequency_hz: 80}
      - kind: dynamics
        compressor: {enabled: true, threshold_dbfs: -18, ratio: 4, attack_ms: 10, release_ms: 100}
        limiter: {enabled: true, ceiling_dbfs: -1, release_ms: 50}
buses:
  main: {channels: 2, format: stereo, output: outgoing, output_channels: [0, 1], fader_db: 0, mute: false, inserts: []}
sends:
  voice_to_main: {from: {strip: voice}, to: main, gain_db: 0}
```

`inputs`、`outputs`、`strips`、`buses`、`sends` は名前を key にする mapping です。五つの mapping は必須で、空でも構いません。外の口は一度だけ宣言し、strip と bus はその名前を参照します。同じ device または同じ方向の network address と port を二つの名前で宣言する設定は拒否します。port 0 であっても同じ address の二重宣言は拒否します。

strip は `first_channel` (省略時は 0) と `channels` で連続する範囲を選びます。複数の strip が同じ範囲を受けても構いません。bus は `output_channels` の n 番目に、bus の ch n の割当先を指定します。出口を持たない中間の bus は `output` と `output_channels` を省略します。

strip と bus の `format` は、`mono` (1 ch)、`stereo` (2 ch、ch 0 が左、ch 1 が右)、`multi-mono` (ch ごとに独立した mono の束) です。省略時は `multi-mono` となり、既存の設定も同じように動きます。形式と ch の数が合わない設定は拒否します。外の口の `format` は sample の形式で、strip と bus の音の形式とは別の値です。

send の `from` は `{strip: 名前}` または `{bus: 名前}`、`to` は bus の名前です。`connect` を省略した mono strip から stereo bus への send は、ch 0 を左右へ 0 dB でつなぎます。それ以外の組は、送り元と bus の ch の数が同じ場合だけ、同じ番号の ch を 0 dB でつなぎます。明示する時は `connect: [{source: 0, destination: 1, gain_db: -6}]` の形で、つなぐ組だけを書きます。`connect: []` はどの ch もつなぎません。

mono と stereo の strip は `pan` を持ち、有限の -100 (左) から +100 (右)、省略時は 0 (中央) です。multi-mono の strip と bus は PAN を持ちません。YAML で multi-mono に `pan` を書くと、0 でも拒否します。PAN は stereo bus へ送る各 send の、送り先 ch 0 と ch 1 の gain にだけ係数を掛けます。`θ = (pan + 100) / 200 × π/2` として、mono は左 `cos θ`、右 `sin θ` (中央は左右とも約 -3.01 dB)、stereo は左 `min(1, √2 cos θ)`、右 `min(1, √2 sin θ)` (中央は左右とも 0 dB) です。端では反対側が 0 になります。PAN は stereo 以外の bus への send と strip の meter に影響しません。

`defaults.sample_rate` は mixer の rate です。device と network input は `sample_rate` を上書きでき、境界の変換が公称 rate の違いと clock のずれを一緒に扱います。network output は mixer の rate と clock で、処理した block を sample の位置と期限に従って送ります。同じ clock の出口には、別の clock を回復する変換や buffer の待ちを重ねません。外の口は `buffer_ms` を上書きでき、mixer は 20〜1,000 ms を受け付けます。省略時は 40 ms です。block は各 rate の 4 ms に最も近い整数の frame 数です。

network の口は `channels`、`format`、`receive` または `send` を持ちます。`format` は `pcm16`、`pcm24`、`float32`、`max_datagram_bytes` は省略時 1200 です。device の口は `device: 'Windows が示す完全な名前'` を持ち、ch の数を指定しません。device を開く mode で分かった全 ch を使えます。`mode` は `exclusive` または `shared` で、device の値、defaults の値、exclusive の順で選びます。開けない時に別の mode を試すことはありません。

`clock` は `os`、`{input: 名前}`、`{output: 名前}` です。device を選ぶ時は宣言済みの device の口を指定します。省略時は strip の記載順で最初の録音 device を使い、そのような strip が無ければ OS の clock を使います。実行中に clock は変えられません。

strip の処理順は、ch の選択、input gain、strip meter、insert、fader、mute、PAN (mono と stereo だけ)、send です。bus の処理順は、send の足し合わせ、insert、fader、mute、bus meter、出口と send です。bus は input gain を持ちません。`inserts` は `kind` と任意の `name` を持つ順序付きの列です。`kind` は `eq` または `dynamics` で、未知の種類や欄、同じ node の中の重複した名前は拒否します。空の列はそのまま音を通します。

```cpp
#include <ghidorah/ghidorah.hpp>
auto mixer = ghidorah::Mixer::from_file("mixer.yaml");
auto levels = mixer.level_reader();
mixer.start();
mixer.update_strip("voice", {0, -6, false});
auto stats = mixer.input_statistics("incoming");
auto values = levels.read();
mixer.stop();
```

## YAML を使わずに組み立てる

```cpp
ghidorah::MixerConfig config(48000);
ghidorah::StreamConfig stream{48000, 2, GHIDORAH_FLOAT32, 40, 1200};
config.network_input("incoming", stream, "0.0.0.0", 7000);
config.network_output("outgoing", stream, "127.0.0.1", 7001);
auto eq = ghidorah::default_eq_parameters(48000);
eq.high_pass.enabled = 1; eq.high_pass.frequency_hz = 80;
auto dynamics = ghidorah::default_dynamics_parameters();
dynamics.compressor.enabled = 1;
dynamics.limiter.enabled = 1;
ghidorah::Strip strip{"voice", "incoming", 0, 2};
strip.format = GHIDORAH_STEREO;
strip.inserts = {{.kind = "eq", .eq = eq}, {.kind = "dynamics", .dynamics = dynamics}};
config.strip(strip);
ghidorah::Bus bus{"main", 2, "outgoing", {0, 1}};
bus.format = GHIDORAH_STEREO;
config.bus(bus);
config.sends({{"voice_to_main", GHIDORAH_NODE_STRIP, "voice", "main"}});
config.clock(GHIDORAH_CLOCK_OS);
ghidorah::Mixer mixer(config);
```

C ABI の対応する型は `ghidorah_mixer_config`、`ghidorah_strip_config`、`ghidorah_bus_config`、`ghidorah_send` です。`ghidorah_mixer_config_add_*` と `set_sends`、`set_clock` で設定を作り、`ghidorah_mixer_create` に渡します。YAML の入口は `ghidorah_mixer_from_yaml` と `from_file` です。設定は名前と配列をコピーするので、作成後に設定の handle を破棄できます。C++ は C ABI を包む header です。

C ABI と C++ は `ghidorah_audio_format` を使い、`GHIDORAH_MULTI_MONO = 0`、`GHIDORAH_MONO = 1`、`GHIDORAH_STEREO = 2` です。`ghidorah_strip_config`、`ghidorah_bus_config` と C++ の `Strip`、`Bus` は末尾の `format` に形式を受けます。0 初期化と省略は multi-mono になります。`ghidorah_strip_parameters` の末尾の `pan` は既定 0 で、multi-mono では使わない 0 だけを受け付けます。

これらの C ABI の struct を使う利用側は、新しい header と library に合わせて再 build してください。

insert の `name` は省けます。自動の名前は種類の名前 `eq` / `dynamics` から決め、同じ種類の二つ目からは `eq-2`、`eq-3` のように番号を付けます。明示名の insert も種類ごとの数に含めます。利用者の名前は先に全て確保し、自動名が重なる時は重ならない次の番号を使います。たとえば `[{kind: eq}, {name: eq-2, kind: eq}, {kind: eq}]` の名前は `eq`、`eq-2`、`eq-3` です。名前は strip / bus ごとに決まり、一覧と parameter の path にそのまま現れます。構成を置き換えた時も新しい insert の列から同じ規則で決めます。

C ABI は `ghidorah_insert_config.name` が `NULL` または空文字の時、C++ は空の `Insert.name` または `Insert{.kind = "eq"}` の時に名前を省いたものとして扱います。

## EQ と dynamics の設定

EQ は `high_pass`、`low_shelf`、`peak1`、`peak2`、`high_shelf` の順の 5 帯域です。各帯域は `enabled` と `frequency_hz` を持ち、shelf と peak は `gain_db`、peak は `q` を持ちます。high pass は 12 dB/oct です。既定は全帯域 off、gain は 0 dB、Q は 1、周波数は mixer rate を 600、240、48、12、6 で割った値です。

Dynamics は `expander`、`compressor`、`makeup`、`limiter` の決まった順で処理します。全 stage は既定で off です。

| stage | 欄と既定 |
|---|---|
| expander | threshold_dbfs=-60、ratio=2、gate=false、attack_ms=5、release_ms=50 |
| compressor | threshold_dbfs=-18、ratio=4、attack_ms=10、release_ms=100 |
| makeup | gain_db=0 |
| limiter | ceiling_dbfs=-1、release_ms=50。attack は 0 に固定 |

expander と compressor は各 ch の二乗の 10 ms の平均のうち最大の power を共通の RMS とし、全 ch に同じ gain を掛けます。`gate: true` は無限大の ratio の gate です。ratio の欄には有限の 1 以上を指定します。limiter は最大の peak に応じて先読みせずに動作します。on/off と ceiling の変更は 5 ms で移るため、その間は混合した音が新しい target ceiling を超えることがあります。有効への遷移が終わると target を守ります。有効な limiter は insert の出力を ceiling で制限します。利用者が insert の後の fader を上げたり、bus で複数の音を足し合わせたりすると、mixer の音の値が ceiling を超えることがあります。

周波数は有限で 0 より大きく Nyquist より小さい値、Q は有限の正の値です。time は有限の 0 以上です。gain と threshold は有限の dB で、makeup の linear gain と limiter の正の float32 ceiling、係数の安定性と表現可能性も検査します。一覧の scalar の範囲は必要条件で、値の組を検査する `Equalizer::prepare` / `Dynamics::prepare` を置き換えません。

C は `ghidorah_eq_parameters_init(rate, &eq)` と `ghidorah_dynamics_parameters_init(&dynamics)` で既定を得ます。`ghidorah_insert_config` の kind に応じた eq または dynamics の pointer を指定し、省略すると同じ既定を使います。settings と名前はコピーされます。

動作中は `Mixer::update_eq(kind, node, insert, parameters)` と `update_dynamics`、C ABI の `ghidorah_mixer_update_eq` / `ghidorah_mixer_update_dynamics` で一つの insert の全設定を変えます。処理の history を保ち、変更する係数や on/off の混合を 5 ms で移します。値の拒否では音と revision を変えません。insert の追加や種類の変更は、mixer を組み立てる時に指定します。

## 実行中の操作と一覧

`update_strip(name, StripParameters)`、`update_bus(name, BusParameters)` は該当する node の parameter 全体を置き換えます。`update_sends(sends)` は send の列全体を置き換えます。空の列は send を全て外します。gain、mute、PAN の左右の係数は 5 ms の間で新しい値へ移ります。不正な更新は音と parameter の一覧を変えません。一度使った send の名前は、その mixer の間、送り元と送り先を保持します。

`inputs()` と `outputs()` は、使われていない宣言済みの口も含め、宣言順で名前と解決済みの stream を返します。C ABI は `ghidorah_mixer_input_count/input_at` と `output_count/output_at` です。C の名前は次の controller 操作まで有効で、C++ と Python はコピーを返します。

`input_settings()` と `output_settings()` は、同じ順で `EndpointSettings` を返します。`kind` は `EndpointKind.NETWORK` または `DEVICE`、`stream` は ch 数、sample rate、sample の形式、buffer_ms、max_datagram_bytes を持ちます。network は `address` と宣言した `port`、device は `device` の名前と `mode` を持ちます。network の入口の `receive_port` は実際に受ける port で、宣言した `port: 0` と区別します。それ以外の口の `receive_port` は 0 です。device の `stream.channels` は開いた結果であり、設定として渡す数ではありません。該当しない文字列と port は空文字と 0、mode は device の時だけ意味を持ちます。

C ABI の `ghidorah_mixer_input_settings_at` と `ghidorah_mixer_output_settings_at` は `ghidorah_endpoint_settings` を返します。件数は従来の input_count / output_count を使います。C の文字列は次の controller 操作まで有効で、C++ と Python は所有する値を返します。選ばれた clock の種類と対象の名前は、既存の `clock()` (`ghidorah_mixer_clock`) で読みます。

`parameters()` は strip、bus、send、insert の entity と、parameter の名前、型、範囲、現在の値、revision を返します。strip は `input_gain_db`、`fader_db`、`mute` と、mono / stereo でだけ `pan`、bus は `fader_db`、`mute`、send は `gain_db` と `channels/<source>/<destination>/gain_db` を持ちます。現在の値は受理した目標値で、ramp の途中の係数ではありません。

strip の entity は、外の入口の名前を `input`、受ける最初の ch を `first_channel`、受ける ch の数を `channels` に返します。ch は 0 から数えます。bus の entity は、出口の名前を `output`、bus の ch ごとの出口の ch を `output_channels` に返します。たとえば `[3, 1]` は、bus の ch 0 を出口の ch 3 へ、bus の ch 1 を出口の ch 1 へ渡します。出口を持たない bus は空の名前と空の割り当てを返します。該当しない entity の欄は空の名前、0、空の割り当てで、bus の `channels` も 0 です。名前の欄は encode せず、構成で指定した名前を返します。

C ABI の `ghidorah_entity_info` は、`input` と `output` を `const char*`、`first_channel` と `channels` を `uint16_t`、`output_channels` を `const uint16_t*` と `output_channel_count` (`uint32_t`) で返します。割り当てが空の時は pointer が `NULL`、長さが 0 です。文字列と配列は snapshot の破棄まで有効です。C++ の `EntityInfo` は名前を `std::string`、割り当てを `std::vector<std::uint16_t>` にコピーし、snapshot と mixer の破棄後も保持できます。動いている構成の入口や出口だけを変えた時も、その entity が新しい revision を持ち、`parameters(previous.revision)` に現れます。

strip と bus の entity は `format` に同じ `ghidorah_audio_format` の値を返します。send と insert の `format` は 0 で、その種類では意味を持ちません。形式だけを変えた時も entity の revision を更新します。PAN を持たない形式へ変えると PAN の除去を通知し、mono / stereo に変えると既定 0 の PAN を持たせられます。

insert の entity は `insert_kind`、`parent_kind`、`parent_name` を持ちます。別の node で同じ insert 名を使えます。insert の parameter の local な name は `high_pass/frequency_hz` や `compressor/ratio`、path は `strips/voice/inserts/eq/high_pass/frequency_hz` などです。

entity と parameter は `order` も持ちます。C ABI は `uint32_t`、C++ は `std::uint32_t` です。配列自体は従来の path 順で返します。同じ `parent_kind` と `parent_name` を持つ insert を `order` の小さい順に並べると、その親の insert 枠の処理順になります。番号は枠の中で 0 から数えます。insert 以外の entity の番号は 0 で、異なる strip や bus の順を表しません。

parameter の `order` は、同じ entity の中で 0 から数えます。EQ は high pass、low shelf、peak1、peak2、high shelf の順で、各帯域では enabled、frequency、gain、Q のうち存在する欄をこの順で返します。dynamics は expander、compressor、makeup、limiter の順です。expander と compressor の中は enabled、threshold、ratio、gate、attack、release のうち存在する欄、makeup は enabled と gain、limiter は enabled、ceiling、release の順です。strip は input gain、fader、mute、PAN (mono と stereo だけ) の順に 0、1、2、3、bus は fader、mute の順に 0、1 とします。send は gain が 0、続く ch の組の gain は送り元 ch と送り先 ch の番号順です。parameter の番号を insert の枠の番号と直接比較しません。

`reconfigure` で insert の順だけを変えた時も、変更した insert の entity が新しい revision を持ち、差分に現れます。insert の中の parameter の順と値が同じ時は、parameter の差分は空です。以前の snapshot は以前の番号を保持します。

各 parameter の `path` は `strips/voice/fader_db`、`buses/main/mute`、`sends/voice_to_main/channels/0/0/gain_db` などの名前の階層です。`name` は従来の local な名前を保ちます。`Mixer::set_parameter(path, double)` と `set_parameter(path, bool)` は、その一覧にある一つの値を変えます。C ABI は型と値を持つ `ghidorah_parameter_value` を `ghidorah_mixer_set_parameter` に渡します。知らない path、型の違う値、無効な値は、設定と revision を変えずに拒否します。send の有無は `update_sends` で変えます。

path は先頭の `/` を持ちません。名前は UTF-8 の一つの path の要素として percent encode し、名前に含む `/` と `%` も encode します。REST の application は一覧の path に `/` を付けて使えます。範囲の端を含むかは `minimum_inclusive` と `maximum_inclusive` で示します。

次に `parameters(previous.revision)` を読むと、前回以降に変わった値と、外した send の `removed` の記録を得られます。同じ parameter の複数の変更は最新値へまとめます。各読み手は自分の revision を保持します。callback は使いません。音の処理を読み手や Python の GIL の待ちから切り離し、読み手ごとに間隔を選ぶためです。C ABI の snapshot は `ghidorah_mixer_parameters` で作り、`ghidorah_parameter_snapshot_*` で読み、destroy で破棄します。

## meter、統計、device の復帰

各 LevelReader は、登録した後、または前回読んだ後に記録した音を独立して数えます。strip、send、bus、output の名前と ch ごとに peak と RMS (dBFS)、sample 数、出口の超過数と運転開始からの累計を返します。strip meter は input gain の後・insert の前、send meter は gain と PAN の係数の後・足し合わせの前、bus meter は insert・fader・mute の後です。PCM は出口で一度だけ範囲の端に揃え、float32 は超過した値もそのまま出します。無音の時と、前回読んだ後に音を記録していない時は、peak と RMS が `-inf` になります。

`input_statistics(name)` と `output_statistics(name)` は欠けた frame の数、network input の遅い packet の数、揺れを吸収できる幅 (ms)、`device` と `clock_device` の状態を返します。device の状態は名前、理由、`unavailable` を持つ所有する値です。

入力の幅は、変換が実際に使う layout と入力の frame の単位から求めます。独立した clock の出口も、変換の入力となる mixer の frame の単位で、buffer から固定の上乗せを引きます。再生 device の出口は、開いた backend が一度に求める frame の数を、短い callback に分ける前に使います。差が負になる時は 0 ms を示します。mixer と同じ clock の出口は変換の固定の上乗せを持たないので、幅はその出口の `buffer_ms` です。

device が止まっても、残る device の callback で mixer は動作を続けます。止まった入力は無音、止まった出力は捨て読みとし、同じ名前と mode での復帰を試みます。clock の device が止まった間は network へ送らず、復帰した後の callback から新しい stream を始めます。device の全 ch が変わった時は復帰を待ち、異なる ch の数で使う場合は Mixer を作り直します。

mixer の network input と独立の Receiver は、受けている stream の有効な音の packet が `buffer_ms` より長く届かない時、次に届いた stream を受け直します。送り手の address と port が変わっても受け直します。受け直した音は buffer の長さの遅延で出し、確定した欠けを後から差し替えません。有効な packet が届いている間は、別の stream を受けません。最初の packet の到着が遅れた時も、その後の音を受け、変換の比の範囲で未読の量の時間平均を buffer の長さへ近づけます。

作成時は口を開き、`start()` で callback と worker を始めます。`stop()` は全 callback と thread を join して socket を閉じます。stop 後の start は新しい運転を作り、以前の meter reader は以前の運転の値を保持します。controller 操作は呼び手で直列化してください。独立した meter reader は音と並行して読めます。

## 動いている構成を変える

C ABI の `ghidorah_mixer_reconfigure(mixer, config)` と、C++ の `Mixer::reconfigure(config)` は、変更後の構成全体を既存の組み立ての型で受けます。strip と bus の追加・削除と ch 数と音の形式、strip の外の入口と ch の範囲、bus の出口と ch の割り当て、insert の追加・削除・並びを変えられます。

```cpp
// replacement は変更後の外の口、strip、bus、send、insert の全体を持つ。
ghidorah::MixerConfig replacement(48000);
replacement.network_input("incoming", stream, "0.0.0.0", 7000);
replacement.strip({"voice", "incoming", 0, 2});
replacement.bus({"main", 2});
replacement.sends({{"voice_to_main", GHIDORAH_NODE_STRIP, "voice", "main"}});
mixer.reconfigure(replacement);
```

mixer は渡された構成をコピーします。同じ名前の物も、parameter、send の有無、gain の表を渡した構成の値で置き換えます。今の値を残す時は、parameter の一覧を読み、渡す構成の値を揃えてください。値だけを変える時は既存の parameter の変更と同じ 5 ms で移ります。構造が変わった strip と bus は 5 ms で 0 へ下げ、block の境界で構成全体を切り替え、5 ms で戻します。変えた strip を受けるだけの bus は下げず、その bus が受ける他の strip の寄与を保ちます。新しい入口の音は、届くまで 0 です。

呼び出しは、新しい device と network の口を開いて開始し、検査と準備が成功して、構成全体の反映を受け付けた時点で戻ります。音への反映の完了を待ちません。続けて渡した構成と parameter の変更は受け付けた順に扱います。同じ名前で構造を変えない物について、mixer は変換と insert の処理状態、進行中の ramp、各 reader が前に読んだ後の計測値を保持します。追加・削除した物と parameter の変化は、既存の `parameters(after_revision)` の一つの revision で取得できます。controller の呼び出しは直列に実行してください。

mixer は設定 file と同じ決まりで構成全体を検査します。音の形式と ch の数の不一致、不正な gain の表、bus の輪、出口の ch の重なり、外の口の二重宣言、重複した名前を含む構成は拒否します。mixer の sample rate と clock は変えられません。clock の外の口の削除、clock device の mode や stream の変更も拒否します。automatic の構成を渡しても、clock を strip の新しい並びから選び直しません。

新しい口の open または start に失敗した時、mixer は今回準備した口を片付け、元の構成で音を流し続けます。C ABI は `ghidorah_status` と `ghidorah_last_error()`、C++ は `std::invalid_argument` または `std::runtime_error` で理由を返します。device の理由には外の口の名前、device の名前、mode、backend の理由が入ります。元の口は構成の反映を終えるまで閉じません。同じ device の mode を変える変更は、その device を二つ同時に開けない時は拒否します。その場合、clock 以外の device は一度外す変更と、異なる mode で足す変更の二回に分け、最初の変更が反映されて device が閉じた後に足してください。

## YAML への保存と mixer の置き換え

`mixer.to_yaml()` は、受理済みの構成と parameter の設定値を YAML の文字列にします。mixer の sample rate、選ばれた clock、未使用の物も含む外の入口と出口、strip、bus、send、insert と、その名前、音の形式、割り当て、処理順、設定値を持ちます。insert の名前は確定した名前、network の入口の port は宣言した値です。計測値、buffer の中身、音の frame の位置、device の ch 数は書きません。同じ外の資源を使える環境で読み直すと同じ設定を作れます。

`MixerConfig.from_yaml(text)` と `MixerConfig.from_file(path)` は、外の口を開かずに YAML を検査して所有権を持つ構成を返します。構成自体も `config.to_yaml()` で書き出せます。

```python
saved = mixer.to_yaml()
replacement = ghidorah.MixerConfig.from_yaml(saved)
mixer.replace(replacement)
# 文字列または file を直接渡す形もある。
mixer.replace_yaml(saved)
mixer.replace_file("recall.yaml")
```

`replace` は新しい運転を作り、sample rate、clock、外の口も渡した構成のとおりに置き換えます。動いていた mixer は新しい構成で動き、開始していなかった mixer は開始しません。作り直す間、音は止まります。YAML と構成の検査は、元の音を止める前に行います。同じ exclusive device や受信 port を新しい mixer が使えるように、元の I/O を一時停止してから新しい I/O を開きます。

新しい口の open / start が失敗した時は、新しく開いた口を全て閉じてから元の I/O を復旧し、理由を返します。元の mixer の処理状態、parameter、revision、既存の level reader を保持し、network の入口は元の実際の port に戻します。新しい構成の受理に成功した時は、新しい運転の計測を読むために level reader を取り直してください。以前の reader は以前の運転の記録を保持し、安全に破棄できます。

C ABI の読み込みは `ghidorah_mixer_config_from_yaml` / `ghidorah_mixer_config_from_file`、書き出しは `ghidorah_mixer_to_yaml` / `ghidorah_mixer_config_to_yaml`、置き換えは `ghidorah_mixer_replace` です。書き出した文字列は mixer または config が所有し、次の controller 操作または破棄まで有効です。C++ と Python は文字列をコピーします。controller 操作は、他の controller 操作と同じく直列に行ってください。既存の関数、struct、DLL の major は変えていません。
