# ghidorah

ghidorah は、Windows の音声 device と network の音の流れをつなぐための音声 tool です。

## 今できること

Windows と Linux で、独自の protocol による UDP の音声の送受信を C++ と Python から使えます。
C ABI、C++ header、Python の binding は、strip、bus、send と YAML の設定 file から mixer を作り、動かせます。
Windows の exe は、YAML の mixer を動かし、外の口ごとの統計と音の大きさを表示します。device の一覧と設定の検査も行えます。
遠隔の制御の application `ghidorah-control` は、同じ process の mixer を REST で操作し、WebSocket で音の大きさと値の変化を送ります。ブラウザの簡易 UI から strip、bus と insert の値も変えられます。利用者は Release の `ghidorah` と `ghidorah_control` の wheel を同じ Python の環境に入れて使えます。
今は Linux の音声 device を扱いません。

## 使い始める

[install](install.md) で配布物の取得方法を確認できます。[C と C++ から mixer を使う](mixer.md) で mixer を組み立て、[Python から使う](python.md) の例で音声を送受信できます。[Windows の command line](cli.md) は exe の使い方を説明します。
[ブラウザと program から mixer を操作する](control.md) は、遠隔の制御の application の起動、REST、WebSocket と insert の操作を説明します。

## License

ghidorah の C++ の library、exe、Python の wheel と、遠隔の制御の application は、[MIT License](https://github.com/sync-dev-org/ghidorah/blob/main/LICENSE) で配ります。

配布物には、次の第三者の物が入ります。

- yaml-cpp (MIT License): C++ の library、exe、Python の wheel
- miniaudio (Public Domain と MIT No Attribution から選ぶ形): Windows の C++ の library と exe、Python の wheel
- nanobind (BSD-3-Clause): Python の wheel

wheel に同梱する license の文書は、install 先の `ghidorah-<version>.dist-info/licenses/` にあります。Windows と Linux の archive では `licenses/` にあります。
遠隔の制御の application の MIT License の文書は、`ghidorah_control-<version>.dist-info/licenses/LICENSE` にあります。

| 文書 | 対象 |
|---|---|
| `LICENSE` | ghidorah |
| `yaml-cpp.txt` | yaml-cpp |
| `miniaudio.txt` | miniaudio |
| `nanobind.txt` | nanobind |

再配布する際も、該当する著作権表示と license の文書を含めてください。
