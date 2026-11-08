# ghidorah

ghidorah は、Windows の音声 device と network の音の流れをつなぐための音声 tool です。

## 今できること

Windows と Linux で、独自の protocol による UDP の音声の送受信を C++ と Python から使えます。
Windows の exe は、現在は library の version を表示します。
今は Linux の音声 device を扱いません。

## 今後の予定

Windows の音声 device と network の音の流れを n 対 n でつなぐ経路と、YAML の設定 file による経路の指定を追加する予定です。

## 使い始める

[install](install.md) で配布物の取得方法を確認し、[Python から使う](python.md) の例で音声を送受信できます。

## License

ghidorah の C++ の library、exe、Python の wheel は、[MIT License](https://github.com/sync-dev-org/ghidorah/blob/main/LICENSE) で配ります。

Python の wheel には nanobind (BSD-3-Clause) が入り、Windows の配布物には RtAudio (MIT License と同じ形) が入ります。
wheel に同梱する license の文書は、install 先の `ghidorah-<version>.dist-info/licenses/` にあります。

| 文書 | 対象 |
|---|---|
| `LICENSE` | ghidorah |
| `nanobind.txt` | nanobind |
| `RtAudio.txt` | RtAudio |

再配布する際も、該当する著作権表示と license の文書を含めてください。
