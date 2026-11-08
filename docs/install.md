# 配布物と install

配布物は [GitHub Release](https://github.com/sync-dev-org/ghidorah/releases) から取得できます。

| OS | 配布物 |
|---|---|
| Windows | exe、C++ の library と header、Python の wheel |
| Linux | C++ の library と header、Python の wheel |

C++ の library は、C ABI の関数を公開する共有 library と、C ABI と C++ の header で配ります。
Windows では `ghidorah.dll` と import library `ghidorah.lib`、Linux では `libghidorah.so` を配ります。
Python の wheel は Windows と Linux の x86_64 向けで、Python 3.12 以上で使える abi3 です。
wheel には `.pyi` と `py.typed` が入り、IDE や型の検査の tool から API の型を参照できます。

利用する OS の wheel を取得し、次の command の `WHEEL_FILE` を取得したファイルの path に置き換えて install してください。

```console
python -m pip install WHEEL_FILE
```
