# ブランチ運用と VC++6 互換確認

## ブランチの役割

- `origin/autobuild`: upstream (libtiff) の変更を自動で取り込むブランチ。独自コミットは持たない。
- `master`: `autobuild` の上に VC++6 対応コミット(「fix for old compilers」)を載せたもの。

`libtiff/CLAUDE.md` に VC++6 向けの C89 コーディング規約がある。

## autobuild から master への rebase 手順

1. `git fetch origin` で `origin/autobuild` を最新にする。ローカルの `origin/autobuild` は古いことがある。
2. `git log --oneline origin/master..origin/autobuild` で取り込まれる upstream の変更を確認し、
   `git diff <旧 merge-base> origin/autobuild` で差分を読む。
3. `git branch backup/master-before-rebase master` でバックアップを作る。
4. `git rebase origin/autobuild` を master 上で実行する。衝突したら VC6 対応側の変更を壊さないように解く。
5. 下記の手順で VC6 コンパイルを確認する。
6. 問題がなければ `git push --force-with-lease origin master`。履歴を書き換えるので、実行前にユーザーへ確認する。

## VC6 でのコンパイル確認

VC6 は `C:\Program Files (x86)\Microsoft Visual Studio\VC98` にある。
PowerShell で `VCVARS32.BAT` の環境変数を取り込み、`libtiff\*.c` を 1 ファイルずつ `cl -c -W3` でコンパイルする。

```powershell
$vcvars = 'C:\Program Files (x86)\Microsoft Visual Studio\VC98\Bin\VCVARS32.BAT'
foreach ($l in (cmd /c "`"$vcvars`" >nul 2>&1 && set")) {
    if ($l -match '^([^=]+)=(.*)$') { Set-Item -Path "env:$($matches[1])" -Value $matches[2] }
}
$gh  = 'C:\D_DRIVE\MyDocuments\GitHub'
$src = "$gh\libtiff-mirror\libtiff"
$out = "$env:TEMP\vc6out"
New-Item -ItemType Directory -Force $out | Out-Null
$skip = 'tif_unix.c','mkspans.c','tif_jpeg_12.c','tif_jbig.c','tif_lerc.c','tif_lzma.c','tif_webp.c','tif_zstd.c'
Get-ChildItem $src -Filter *.c | Where-Object { $skip -notcontains $_.Name } | ForEach-Object {
    cl.exe -nologo -c -W3 "-I$src" "-I$gh\libjpeg-mirror" "-I$gh\zlib" "-Fo$out\$($_.BaseName).obj" $_.FullName
}
```

- `tif_jpeg.c` は `..\libjpeg-mirror`、`tif_zip.c` は `..\zlib` のヘッダが必要。`-I` を付けないと `jerror.h` / `zlib.h` が無くて失敗する。
- 除外しているファイルは、現在の `tiffconf.h` では使わない機能のもの。
- error と warning が 0 件であることを確認する。rebase 前の master(バックアップ)と同じ手順で比べると、今回の変更が原因かどうか切り分けられる。
- Git Bash から `cl` を呼ぶと `/` 始まりのオプションがパスに変換される。PowerShell から呼ぶこと。
- 新規コードで見落としやすい VC6 の障害は `libtiff/CLAUDE.md` を参照(ブロック先頭以外の宣言、`for` 内の宣言、`1ULL` などの C99 構文)。
