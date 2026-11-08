# Python から音声を送受信する

package 名は `ghidorah` です。次の例は、同じ process の送り手と受け手を localhost でつなぎ、2 ch の音声を 1 block 送ります。

```python
import ghidorah

config = ghidorah.StreamConfig(
    sample_rate=48000,
    channels=2,
    block_samples=512,
    sample_format=ghidorah.SampleFormat.FLOAT32,
    buffer_samples=2048,
)
samples = [0.25, -0.25] * config.block_samples

with ghidorah.Receiver(config, bind_address="127.0.0.1", port=0) as receiver:
    with ghidorah.Sender(config, destination="127.0.0.1", port=receiver.port) as sender:
        sender.write(samples)
        received = receiver.read(timeout_ms=1000)
        assert received == samples
        stats: ghidorah.ReceiverStats = receiver.statistics()
        print("received samples:", len(received))
        print("received packets:", stats.received_packets)

print("library version:", ghidorah.version())
```

`block_samples` と `buffer_samples` は、1 ch あたりのサンプル数です。
`write()` に渡す list と `read()` が返す list は、各時刻の全 ch の値を順に並べた float の list です。
この例では、左と右の値を交互に並べ、512 サンプル × 2 ch の 1024 個の値を送ります。
`port=0` は OS に空いている port を選ばせ、`receiver.port` でその番号を取得します。
`with` を抜けると送り手と受け手は閉じます。

| 名前 | 使い方 |
|---|---|
| `StreamConfig` | sample rate、ch の数、block と buffer のサンプル数、sample の形式を指定します |
| `SampleFormat` | `PCM16`、`PCM24`、`FLOAT32` から送る sample の形式を選びます |
| `Sender` | `write(samples)` で音声を送り、`ping(timeout_ms)` で往復の時間を ms で取得します |
| `Receiver` | `read(timeout_ms)` で 1 block を受け取ります。待ち時間の上限は ms です |
| `ReceiverStats` | `Receiver.statistics()` が返す統計です。`received_packets`、`missing_packets`、`late_packets`、`sender_sample_rate` を参照できます |
| `version` | `version()` で library の version を取得します |
