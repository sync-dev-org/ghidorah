# Windows の command line

`ghidorah.exe` は、YAML の設定 file の経路を動かします。exe と同じ directory に、配布物の `ghidorah-<major>.dll` を置きます。Linux 向けの配布物には exe を含めません。

```text
ghidorah version
ghidorah devices
ghidorah check mixer.yaml
ghidorah run mixer.yaml --interval-ms 1000
```

`version` と `--version` は library の build version を示します。引数を付けずに起動した時も version を示します。`--help` は command の使い方を示します。

`devices` は録音と再生の device の名前、ch の数、sample rate を表示します。名前は Windows の名前との完全一致で指定します。登録された形式と device が受け付ける形式が食い違う device は、両方の形式と Windows の endpoint を作り直す案内を表示します。形式を調べられない時は、unknown の状態と取得時のエラーを表示します。device の一覧を調べても音は流れません。

`check` は YAML と経路を library で検査します。device を選んだ mode で開き、名前、sample rate、全 ch の数と経路の ch の対応も検査します。device は開始せず、音を流しません。network の endpoint は検査中に bind するため、指定した port が使用中なら失敗します。

`run` は起動時の経路と gain を使い、Ctrl+C または停止 signal まで動き続けます。既定の表示間隔は 1000 ms です。`--interval-ms` には正の整数を指定します。経路の変更には library の API を使います。

定期の表示は `key=value` の行です。`input` と `output` の行は名前、`missing_frames`、`late_packets`、`jitter_margin_ms` を返します。使われていない宣言済みの口も表示します。`device_unavailable` と `clock_device_unavailable` は停止中に 1 となり、同じ接頭辞の `_name` と `_reason` は名前と理由を示します。

level の行は `strip`、`send`、`bus`、`output` と名前、ch を示します。`peak_dbfs`、`rms_dbfs`、`samples`、`exceeded_samples` は、前回表示した後に記録した音についての値です。`total_exceeded_samples` は運転開始からの累計です。無音の peak と RMS は `-inf` です。

device が止まっても他の音は続き、同じ名前と mode の復帰を試みます。clock の device が止まった間は network へ送りません。[mixer の説明](mixer.md) に YAML と復帰の扱いを記載しています。

## insert を使う YAML の例

次の設定を `mixer.yaml` に置き、`ghidorah check mixer.yaml` で検査し、`ghidorah run mixer.yaml` で起動できます。

```yaml
defaults: {sample_rate: 48000, buffer_ms: 40}
clock: os
inputs:
  incoming: {receive: {address: 0.0.0.0, port: 7000}, channels: 2, format: float32}
outputs:
  outgoing: {send: {address: 127.0.0.1, port: 7001}, channels: 2, format: float32}
strips:
  voice:
    input: incoming
    channels: 2
    inserts:
      - {name: tone, kind: eq, high_pass: {enabled: true, frequency_hz: 80}}
      - {name: protect, kind: dynamics, limiter: {enabled: true, ceiling_dbfs: -1, release_ms: 50}}
buses:
  main: {channels: 2, output: outgoing, output_channels: [0, 1]}
sends:
  voice_to_main: {from: {strip: voice}, to: main}
```

省略した帯域と stage は内部の既定で埋めます。未知の種類と欄、範囲の外の値は check の段階で拒否します。
