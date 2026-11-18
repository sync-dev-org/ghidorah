# Python から音声を送受信する

package 名は `ghidorah` です。次の例は、同じ process の送り手と受け手を localhost でつなぎ、2 ch の音声を 3 block 送ります。

```python
import ghidorah

config = ghidorah.StreamConfig(
    sample_rate=48000,
    channels=2,
    sample_format=ghidorah.SampleFormat.FLOAT32,
    buffer_ms=40,
)
with ghidorah.Receiver(config, bind_address="127.0.0.1", port=0) as receiver:
    with ghidorah.Sender(config, destination="127.0.0.1", port=receiver.port) as sender:
        samples = [0.25, -0.25] * (3 * sender.block_frames)
        sender.write(samples)
        for _ in range(128):
            received = receiver.read(timeout_ms=1000)
            if all(abs(value - (0.25 if i % 2 == 0 else -0.25)) < 0.02
                   for i, value in enumerate(received)):
                break
        stats: ghidorah.ReceiverStats = receiver.statistics()
        print("received samples:", len(received))
        print("received packets:", stats.received_packets)

print("library version:", ghidorah.version())
```

`buffer_ms` は buffer の長さを ms で指定します。20 ms 以上 1,000 ms 以下の有限の値を受け、既定は 40 ms です。
block の長さは 4 ms で、frame の数は sample rate から最も近い整数に丸めます。
`sender.block_frames` と `receiver.block_frames` で、その frame の数を取得します。
`write()` に渡す list と `read()` が返す list は、各時刻の全 ch の値を順に並べた float の list です。
この例では、左と右の値を交互に並べ、3 block 分の 1,152 個の値を送ります。`read()` は毎回、1 block 分の 384 個の値を返します。
受け手は、送り手と読み手の clock のずれを状態を保持する sample rate の変換で吸収し、全 ch の時間関係を保ちます。起動時には buffer と補間のための音が溜まるまで 0 を返します。変換を通るため、返る sample の値と block の境界は、送った音と完全には一致しません。
`buffer_ms` は、補間の先読みの後に残る未読の音の量の時間平均の目標です。受け手は、最初の packet の到着が遅れた場合も、その後の音を受け、変換の比の範囲で有限の時間をかけて平均を目標に近づけます。
`port=0` は OS に空いている port を選ばせ、`receiver.port` でその番号を取得します。
`with` を抜けると送り手と受け手は閉じます。

| 名前 | 使い方 |
|---|---|
| `StreamConfig` | sample rate、ch の数、buffer の長さ (ms)、sample の形式を指定します |
| `SampleFormat` | `PCM16`、`PCM24`、`FLOAT32` から送る sample の形式を選びます |
| `Sender` | `block_frames` で block の frame の数を読み、`write(samples)` で音声を送り、`ping(timeout_ms)` で往復の時間を ms で取得します |
| `Receiver` | `block_frames` で block の frame の数を読み、`read(timeout_ms)` で 1 block を受け取ります。待ち時間の上限は ms です |
| `ReceiverStats` | `Receiver.statistics()` が返す統計です。`received_packets`、`missing_packets`、`late_packets`、`sender_sample_rate` を参照できます。`missing_packets` は、読んだ位置または保持できなくなった位置の欠けた fragment の数です。`late_packets` は、読んだ位置への到着、または保持範囲から外れた位置への到着として捨てた packet を数えます |
| `version` | `version()` で library の version を取得します |

## 経路の作成と更新

YAML と組み立ての口は、同じ設定の検証と mixer を使います。

```python
from ghidorah import Mixer, StripParameters

with Mixer.from_file("mixer.yaml") as mixer:
    levels = mixer.level_reader()
    before = mixer.parameters()
    mixer.update_strip("voice", StripParameters(fader_db=-6, mute=False))
    changes = mixer.parameters(before.revision)
    values = levels.read()
    stats = mixer.input_statistics("incoming")
```

YAML を使わずに組み立てる例は次のとおりです。

```python
from ghidorah import (Mixer, MixerConfig, StreamConfig, SampleFormat, AudioFormat, Strip, Bus, Send, NodeKind,
                      EqParameters, DynamicsParameters, Insert)

stream = StreamConfig(48000, 2, SampleFormat.FLOAT32, buffer_ms=40)
config = MixerConfig(48000)
config.network_input("incoming", stream, "0.0.0.0", 7000)
config.network_output("outgoing", stream, "127.0.0.1", 7001)
eq = EqParameters(48000)
eq.high_pass.enabled = True
eq.high_pass.frequency_hz = 80
dynamics = DynamicsParameters()
dynamics.compressor.enabled = True
inserts = [Insert(kind="eq", eq=eq), Insert(kind="dynamics", dynamics=dynamics)]
config.strip(Strip("voice", "incoming", 0, 2, inserts=inserts, format=AudioFormat.STEREO))
config.bus(Bus("main", 2, "outgoing", [0, 1], format=AudioFormat.STEREO))
config.sends([Send("voice_to_main", NodeKind.STRIP, "voice", "main")])
with Mixer(config) as mixer:
    print(mixer.inputs(), mixer.outputs(), mixer.parameters())
```

`Strip` と `Bus` の末尾の引数 `format` は `AudioFormat.MONO` (1 ch)、`AudioFormat.STEREO` (2 ch、左と右)、`AudioFormat.MULTI_MONO` (独立した mono の束) を受けます。省略時は `MULTI_MONO` です。形式と ch の数が合わない設定は拒否します。`SampleFormat` は外の音の sample の形式を示す別の列挙です。

`StripParameters(input_gain_db=0, fader_db=0, mute=False, pan=0)` と `BusParameters(fader_db=0, mute=False)` は決まった位置の parameter を扱います。mono / stereo の strip の PAN は -100 (左) から +100 (右)、既定 0 (中央) です。`mixer.set_parameter("strips/voice/pan", -50)` で一つの値を変えられます。PAN は stereo bus への send にだけ効き、mono は等電力で振り、stereo は左右の釣り合いを変えます。係数と meter の位置は [mixer の説明](mixer.md) に記載しています。multi-mono は PAN を公開せず、共通の `StripParameters.pan` には使わない既定値 0 だけを受け付けます。

`update_strip` と `update_bus` は該当する parameter 全体を置き換えます。`update_sends` は send の列全体を置き換え、空の list で全ての send を外します。

send の `channels=None` は mono strip から stereo bus へ左右とも 0 dB でつなぎ、それ以外の組は同じ数の ch を同じ番号でつなぎます。`channels=[ChannelGain(source_channel, destination_channel, gain_db=0)]` は並べた組だけをつなぎます。空の list は ch をつなぎません。`inserts` は `Insert(name="", kind=種類, eq=None, dynamics=None)` の順序付きの列です。kind は `eq` または `dynamics` で、省略した settings は内部と同じ既定を使います。未知の種類は拒否します。

`name` を省く時は `Insert(kind="eq")` のように種類を keyword で渡します。`Insert("eq")` と、種類を指定しない呼び出しは拒否します。明示名も含む種類ごとの出現順から `eq`、`eq-2`、`eq-3` と名付け、全ての明示名との重なりを避けます。strip と bus は別々に数えます。`reconfigure` でも変更後の列から名前を決め、一覧と REST に同じ名前を返します。

`EqParameters(sample_rate)` は high_pass、low_shelf、peak1、peak2、high_shelf を持ち、`DynamicsParameters()` は expander、compressor、makeup、limiter を持ちます。動作中の全設定の置換は `update_eq(NodeKind.STRIP, "voice", "eq", eq)` と `update_dynamics(...)` です。一つの値なら `set_parameter("strips/voice/inserts/eq/high_pass/frequency_hz", 120)` を使えます。一覧の path と setter は同じ parameter の定義を使います。

設定と snapshot は値をコピーするため、組み立てに渡した EqParameters や DynamicsParameters を後から変えても、動いている mixer は変わりません。明示した update または set_parameter で反映します。範囲と limiter の切り替えの扱いは [mixer の説明](mixer.md) に記載しています。

`parameters()` の snapshot は entity、parameter の名前・型・範囲・目標値、revision を持ちます。`parameters(previous.revision)` は最新の変更と除去の記録を返します。読む側はそれぞれ自分の revision を保持します。読み手と GIL の待ちで音の処理を待たせず、読む間隔を独立させるため、callback は使いません。

strip と bus の `EntityInfo.format` は読み取り専用の `AudioFormat` です。send と insert では使わず、値は `MULTI_MONO` (0) です。mono / stereo strip の PAN は型 REAL、範囲 -100〜100、`order=3` として一覧に現れ、input gain 0、fader 1、mute 2 の後になります。PAN の変更は左右の係数を 5 ms で移し、変更後の目標値を revision に載せます。

`snapshot.entities` の `EntityInfo` は、strip の入口を `input: str`、受ける ch の範囲を `first_channel: int` と `channels: int` に返します。bus の出口は `output: str`、bus の ch n を出口のどの ch へ渡すかは `output_channels: list[int]` です。ch は 0 から数えます。出口を持たない bus の値は `""` と `[]` です。該当しない欄は `""`、`0`、`[]` で、bus の `channels` も 0 です。名前は構成で指定した元の文字列です。構成変更後は新しい snapshot が変更後の値を返し、以前の snapshot は以前の値を保持します。接続の欄だけが変わった時も、変更した entity が revision に含まれます。

`EntityInfo.order` と `ParameterInfo.order` は読み取り専用の `int` です。一覧自体は path 順です。同じ親を持つ insert を `order` の小さい順に並べると、insert の枠の処理順になります。番号は枠内で 0 から数え、insert 以外の entity では 0 です。parameter の番号は同じ entity の中で 0 から数え、EQ の帯域と dynamics の stage は [mixer の説明](mixer.md#実行中の操作と一覧) の処理順になります。同じ名前の insert が別の親にもある時は、entity の `path` を prefix にして parameter を選んでください。動いている間に insert の並びだけを変えた時も、差分の entity に新しい番号が現れます。以前の snapshot の番号は変わりません。

各 parameter は `path` も持ちます。`mixer.set_parameter("strips/voice/fader_db", -6)` や `mixer.set_parameter("buses/main/mute", True)` で一つの値を変えられます。Python の bool は BOOL、float と bool 以外の int は REAL として扱います。型の違う値、一覧に無い path、無効な値は拒否し、revision を保ちます。

`DeviceConfig(name, sample_rate, sample_format, buffer_ms=40, mode=DeviceMode.EXCLUSIVE)` は全ての native ch で device を開きます。ch の数は指定しません。`device_input` と `device_output` で一度宣言してから、strip と bus が範囲を選びます。`list_devices()` は名前、方向、全 ch の数、rate、登録された形式と device が受ける形式の診断を返します。

`start`、`stop`、`close` は開始、停止、破棄です。`with` は開始し、抜ける時に破棄します。stop 後の start は新しい運転を作り、以前の LevelReader は以前の値を保持します。`check()` は回復できない非同期の失敗を例外にします。`receive_port(input)` は port 0 で開いた input の実際の port を返します。

`input_statistics` と `output_statistics` は欠け、遅い packet、揺れを吸収できる幅、device の状態を返します。`LevelReader.read()` は、読み手が前回読んだ後に記録した音について、strip、send、bus、output の ch ごとの peak、RMS、超過数を返します。取得した snapshot、統計、level の値は mixer の破棄後も有効です。

controller 操作は一つの mixer ごとに直列化してください。独立した LevelReader は音と並行して読めます。[C と C++ の mixer](mixer.md) に YAML、clock、device の復帰、meter の位置を記載しています。

## 動いている mixer の構成を変える

`Mixer.reconfigure(config)` は、既存の `MixerConfig` で作った変更後の構成全体をコピーして受けます。strip と bus の追加・削除と ch 数と音の形式、入口と出口の割り当て、insert の追加・削除・並びを変えられます。

```python
replacement = MixerConfig(48000)
replacement.network_input("incoming", stream, "0.0.0.0", 7000)
replacement.strip(Strip("voice", "incoming", 0, 2))
replacement.bus(Bus("main", 2))
replacement.sends([Send("voice_to_main", NodeKind.STRIP, "voice", "main")])
mixer.reconfigure(replacement)
```

`reconfigure` は、同じ名前の物も、parameter、send の有無、gain の表を渡した構成の値で置き換えます。今の値を残す時は `mixer.parameters()` の一覧を読み、渡す構成の値を揃えてください。構造が同じ物について、mixer は処理状態と各 reader が前に読んだ後の計測値を保持します。値だけを変える時は、既存の 5 ms の移り方を使います。構造が変わる strip と bus は 5 ms で下げてから構成全体を切り替え、5 ms で戻します。

呼び出しは準備と検査が成功して構成全体の反映を受け付けた時点で戻り、音への反映の完了は待ちません。続けて来た変更は順に扱います。`parameters(before.revision)` は、追加・削除した物と値の変化を一つの revision で返します。mixer の sample rate と clock は変更できず、不正な構成は `ValueError`、新しい口の open または start の失敗は `RuntimeError` で理由を返します。失敗した変更は元の構成を保ちます。同じ device の mode を変える時の制約と、C ABI に対応する動作は [mixer の説明](mixer.md#動いている構成を変える) を参照してください。
