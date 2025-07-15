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
```
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
