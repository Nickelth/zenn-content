---
title: "我的バッチ言語のアイデア帳"
emoji: "⌨"
type: "tech"
topics: ["cmd", "windows", "cli"]
published: false
---

### 概要


:::details 10分後の時刻を取得する処理
``` bat:batchfile
for /f "tokens=1-2 delims=:" %%a in ("%time%") do (	
    set /a hh=%%a	
    set /a mm=%%b + 10	
)

:: 分が60以上になったら繰り上げ	
if !mm! GEQ 60 (
    set /a mm-=60
    set /a hh+=1
)

:: 24時を超えたら0にする
if !hh! GEQ 24 (
    set /a hh-=24
)

:: ゼロパディング
if !hh! LSS 10 set hh=0!hh!	
if !mm! LSS 10 set mm=0!mm!

:: 時刻フォーマット	
set starttime=!hh!:!mm!
```
:::


:::details タスクスケジューラに新規タスクを登録する処理
``` bat:batchfile
rem タスク作成ログ出力	
set taskcmd="\"C:\EXECUTE.bat\"
schtasks /create ^
/tn "!taskname!" ^
/tr !taskcmd! ^
/sc once ^
/st !starttime! ^
/f
```
:::

:::details コマンド実行時にエラーメッセージがでる場合はスルーする
``` bat:batchfile
schtasks /query /tn !taskname! 2>nul
if !errorlevel! equ 0 (
    schtasks /delete !taskname! /f
)
```
1行目で!taskname!に一致するタスク名をクエリする。
タスクが存在しなければエラーメッセージがターミナルに表示されるが、
2>nulを末尾につけることでエラーメッセージをスルーできる。

1>nulにすると成功メッセージをスルーできる。

:::

:::details FOR文の書き方 + if文
残念ながらJavaのstreamやforEach,拡張for文に当たるものはない。
基本for文のみ解説する。

``` bat:batchfile
for %%f in ("C:\User\sample.csv") do (
	copy /y %%f "C:\User\result.csv" >> %LOGFILE%
	if !ERRORLEVEL! neq 0 (
        echo %DATE% %TIME% ERR %DEF_NAME% "COPY 異常終了" >> %ALERT_LOG%
    )
)
```
"C:\User\sample.csv"のデータを1行ずつfor文を回して取得し、
"C:\User\result.csv"にコピーする処理。
%%fは変数。
:::

::: 変数名の定義と呼び出し
``` bat:batchfile
@echo off
setlocal enabledelayedexpansion

set "PATH=C:\User\Project"
mkdir %PATH% "DIR_TASK"
if !ERRORLEVEL! neq 0 (
    echo %DATE% %TIME% ERR %DEF_NAME% "MKDIR 異常終了" >> %ALERT_LOG%
)

```
:::