# VC++6 (MSVC 1200) 互換対応

このディレクトリのコードは VC++6 でビルドする必要がある。
VC++6 は C89 準拠のみで C99 機能を認識しない。

## tiffconf.h による互換定義

`tiffconf.h` の末尾（`#endif` 直前）に以下が必要:

```c
#if defined(_MSC_VER) && (_MSC_VER < 1400)
#  ifndef inline
#    define inline __inline
#  endif
#  ifndef snprintf
#    define snprintf _snprintf
#  endif
#endif
```

## C89 準拠の注意点

新規コードを書く場合・修正する場合は以下を厳守すること。

**ブロック先頭以外での変数宣言は禁止**
- `if`/`for`/`while` ブロック内でも、最初の文（代入・関数呼び出し等）より後に宣言を置いてはならない
- `(void)x;` のような suppression 文もブロック先頭に宣言がある場合はその後で良いが、宣言より前に他の文があってはならない

**`for` ループ内の変数宣言は禁止**
- `for (int i = 0; ...)` は使用不可。事前に宣言すること

**`inline` は `tiffconf.h` の define で対応済み**

**`snprintf` は `tiffconf.h` の define で対応済み**

## マクロでの C99 機能使用禁止

`FILL`/`ZERO` マクロの例のように、マクロ内でも `for (int32_t i = ...)` のような C99 for-init は不可。
`memset` 等で代替すること。
