---
title: "バッチファイルのクソ仕様にやられたので備忘録まとめ"
emoji: "⌨"
type: "tech"
topics: ["cmd", "windows", "cli"]
published: true
---

### 0. はじめに
業務で普段使いするバッチプログラムをまとめて書いてみた。
`AND`と`OR`と`break`と`continue`できないので不満が多い言語

#### 10分後の時刻を取得する処理
現在時刻を分に直し、10分足して剰余演算する処理
バッチは言語仕様によりif文が煩雑になりがち
如何にテスト工程数を減らすかが肝
``` bat:batchfile
@echo off
setlocal enabledelayedexpansion

:: 現在時刻をUNIX時間っぽく処理する
for /f "tokens=1-2 delims=:" %%a in ("%time%") do (
    set /a totalMin=%%a * 60 + %%b + 10
)

set /a hh=totalMin / 60 %% 24
set /a mm=totalMin %% 60

:: ゼロパディング
if !hh! LSS 10 set hh=0!hh!
if !mm! LSS 10 set mm=0!mm!

set starttime=!hh!:!mm!
echo %starttime%
```


#### タスクスケジューラに新規タスクを登録する処理
`schtasks /create`コマンド
これ自体は普通の処理
ターミナルに入力する場合は改行せずに1行で新規登録できるが、ファイルに書き込むときは改行しないとうまく動作しない場合もある

`/tr`の引数のパスにダブルクォートやスペースが入るとバッチが正しく認識しなくなる
しかしパスの部分には変数を使いたい場合が多く、いちいち考慮していられない

そこで、`/tr`にコマンドラインのパスを渡してあげる
``` bat:batchfile
rem タスク作成ログ出力	
schtasks /create /tn "MyTask" ^
 /tr "\"cmd.exe\" /c \"C:\path with spaces\my_script.bat\"" ^
 /sc once /st 12:00
```
↑`/tr`の解釈をコマンドラインに任せることで安定して動作できる


#### コマンド実行時にエラーメッセージがでる場合はスルーする
``` bat:batchfile
@echo off
setlocal enabledelayedexpansion

schtasks /query /tn !taskname! 2>nul
if !errorlevel! equ 0 (
    schtasks /delete !taskname! /f
)
```
1行目で`!taskname!`に一致するタスク名をクエリする。
タスクが存在しなければエラーメッセージがターミナルに表示されるが、
`2>nul`を末尾につけることでエラーメッセージをスルーできる。

`1>nul`にすると成功メッセージをスルーできる。

#### FOR文の書き方 + if文
残念ながらJavaのstreamやforEach,拡張for文に当たるものはない。
基本for文のみ解説する。

``` bat:batchfile
@echo off
setlocal enabledelayedexpansion

for /f "usebackq tokens=*" %%f in ("C:\User\sample.csv") do (
	echo /y %%f >> "C:\User\result.csv"
	if !ERRORLEVEL! neq 0 (
        echo %DATE% %TIME% ERR %DEF_NAME% "COPY 異常終了" >> %ALERT_LOG%
    )
)
```
`"C:\User\sample.csv"`のデータを1行ずつ`for`文を回して取得し、
`"C:\User\result.csv"`にコピーする処理。
`%%f`は変数。

#### おわり

```bat
cipher /w:C:\
```
ドライブ上の未使用領域（削除済みファイルの断片が残る可能性のある領域）を、00→FF→ランダムの順で3回上書き

#### Windowsバッチの記事
@[card](https://zenn.dev/nickelth/articles/setvarvariable)