# 配布物と install

配布物は [GitHub Release](https://github.com/sync-dev-org/ghidorah/releases) から取得できます。

| OS | 配布物 |
|---|---|
| Windows | exe、C++ の library と header、`ghidorah` と遠隔の制御の application の wheel |
| Linux | C++ の library と header、`ghidorah` と遠隔の制御の application の wheel |

C++ の library は、C ABI の関数を公開する共有 library と、C ABI と C++ の header で配ります。
Windows では DLL `ghidorah-<major>.dll` と import library `ghidorah.lib`、Linux では `libghidorah.so` を配ります。DLL の名前の `<major>` は C ABI の major (version の先頭の数、たとえば `0.2.0` なら `0`) で、C ABI の major が変わると DLL の名前も変わります。CMake から使う時は、DLL の名前ではなく CMake の target で link してください。
package `ghidorah` の wheel は Windows と Linux の x86_64 向けで、Python 3.12 以上で使える abi3 です。
`ghidorah` の wheel には `.pyi` と `py.typed` が入り、IDE や型の検査の tool から API の型を参照できます。

利用する OS の wheel を取得し、次の command の `WHEEL_FILE` を取得したファイルの path に置き換えて install してください。

```console
python -m pip install WHEEL_FILE
```

## 遠隔の制御の application を入れる

利用者は Python 3.12 を用意し、同じ Release から次の二つの wheel を取得してください。`VERSION` は選んだ Release の version (tag の先頭の `v` を除いた値) に置き換えます。application は同じ version の `ghidorah` を必要とします。

| OS | 取得する file |
|---|---|
| Windows x86_64 | `ghidorah-VERSION-cp312-abi3-win_amd64.whl` と `ghidorah_control-VERSION-py3-none-any.whl` |
| Linux x86_64 | `ghidorah-VERSION-cp312-abi3-linux_x86_64.whl` と `ghidorah_control-VERSION-py3-none-any.whl` |

Windows の利用者は、取得した file のある directory の PowerShell で実行します。

```powershell
py -3.12 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install .\ghidorah-VERSION-cp312-abi3-win_amd64.whl .\ghidorah_control-VERSION-py3-none-any.whl
```

Linux の利用者は、取得した file のある directory で実行します。

```sh
python3.12 -m venv .venv
. .venv/bin/activate
python -m pip install ./ghidorah-VERSION-cp312-abi3-linux_x86_64.whl ./ghidorah_control-VERSION-py3-none-any.whl
```

pip は FastAPI、Uvicorn などの Python の依存も取得します。C++ の build の準備は要りません。application の wheel は Python の code と HTML、JavaScript、CSS を含み、Windows と Linux で同じ file を使います。

利用者は [制御の文書の YAML の例](control.md) を `mixer.yaml` に保存し、wheel を入れた環境で起動できます。

```sh
python -m ghidorah_control mixer.yaml
```

ブラウザで <http://127.0.0.1:8000/> を開きます。起動の option と操作は [ブラウザと program から mixer を操作する](control.md) に記載しています。
