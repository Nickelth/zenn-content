---
title: "【Windows】メモリ自動クリーンタスクの設定【PowerShell】"
emoji: "🗑️"
type: "tech"
topics: ["Windows"]["PowerShell"]
published: false
---

## 動機
Microsoftが提供するPC管理ツール「[MicroSoft PC Manager](https://pcmanager.microsoft.com/ja-jp)」を使用するとPCのメモリが簡単に解放できます。
ベータ版の頃から愛用していましたが、開発に使用しているアプリが重すぎて5分に1回はボタンを押してます

毎回ボタンを押しているとさすがに業務効率が下がるので、この度自動化しようと思いました

## 要件
- Windowsのメモリ使用量を監視し、80%を超えた場合に自動でメモリクリーニングを行う
- タスクの実行ログとエラーログを出力し、指定のフォルダに保存する

## 実装方法

### 1.メモリ使用量を監視するスクリプトの記述

MonitorMemory.ps1
``` powershell
param(
    [string]$logFilePath = "C:\path\to\logs\memory_monitor_log.txt",
    [string]$errorFilePath = "C:\path\to\logs\memory_monitor_error.txt"
)

# ログファイルに書き込む関数
function Write-Log {
    param (
        [string]$message
    )
    $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    $logMessage = "$timestamp - $message"
    Add-Content -Path $logFilePath -Value $logMessage
}

# エラーログファイルに書き込む関数
function Write-ErrorLog {
    param (
        [string]$errorMessage
    )
    $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    $errorLogMessage = "$timestamp - ERROR: $errorMessage"
    Add-Content -Path $errorFilePath -Value $errorLogMessage
}

try {
    $mem = Get-CimInstance Win32_OperatingSystem
    $usedMemory = [math]::Round((($mem.TotalVisibleMemorySize - $mem.FreePhysicalMemory) / $mem.TotalVisibleMemorySize) * 100, 2)

    if ($usedMemory -gt 80) {
         Write-Log "Memory usage is $usedMemory%, cleaning up memory..."
        # メモリクリーニングを行うコマンド（例: Windowsのページファイルをクリア）
        & "C:\Scripts\Clear-PageFile.ps1"  #必ずCドライブからのフルパスで記述する
        Write-Log "Memory cleanup completed."
    } else {
        Write-Log "Memory usage is $usedMemory%, no action needed."
    }
} catch {
    Write-ErrorLog $_.Exception.Message
}

```

### 2.メモリクリーニングを行うコマンドの記述
Clear-PageFile.ps1
``` powershell
[System.Diagnostics.Process]::GetProcesses() | Where-Object { $_.PrivateMemorySize64 -gt 0 } | ForEach-Object { $_.MinWorkingSet = 1 }
```
これは、現在実行中のプライベートメモリを所有するすべてのプロセスを取得し、最小ワーキングサイズを1にしてメモリ開放するコマンドです。
ちなみに"$_.MinWorkingSet"を0にすることはシステムに影響がでるので、たとえ管理者権限であっても拒否されます。

しかし上記のコマンドだと、アクセス拒否エラーが出ることがあります。
``` powershell
ERROR: "MinWorkingSet" の設定中に例外が発生しました: "アクセスが拒否されました。"
```

今回実装するタスクは管理者権限ですら実行できないのでアクセス拒否が発生します。
そこで、例外処理を追加してアクセス拒否を回避します。
``` powershell
[System.Diagnostics.Process]::GetProcesses() | Where-Object { $_.PrivateMemorySize64 -gt 0 } | ForEach-Object {
    try {
        $_.MinWorkingSet = 1
    } catch {
        Write-Host "Failed to modify MinWorkingSet for process $($_.ProcessName): $_"
    }
}
```
これでアクセス拒否が発生してもスクリプトの実行を続けることができます。
エラーが発生した場合は、詳細をコンソールに出力します。

### 3.タスクスケジューラの設定

①「タスクスケジューラ」を開き、右側のタスクの作成をクリックします。
![](https://storage.googleapis.com/zenn-user-upload/0243e63f3cc0-20240902.png)

②「全般」タブでタスクの名前を入力し、「最上位の特権で実行する」にチェックを入れます。
![](https://storage.googleapis.com/zenn-user-upload/99a759989dd4-20240902.png)

③「トリガー」タブで「新規」ボタンを押します。
![](https://storage.googleapis.com/zenn-user-upload/042dda38e768-20240902.png)

④「タスクの開始」は「スケジュールに従う」、「繰り返し間隔」は「5分間」を選択
![](https://storage.googleapis.com/zenn-user-upload/76910c160fe4-20240902.png)

⑤「操作」タブも同様に「新規」ボタンを押し、以下の3点を記入します。
    - 操作: プログラムの開始
    - プログラム/スクリプト: powershell
    - 引数の追加: -ExecutionPolicy Bypass -File "C:\path\to\MonitorMemory.ps1"
        ⇧フルパスを記入
![](https://storage.googleapis.com/zenn-user-upload/4593efdecf74-20240902.png)

⑥設定値が追加されているのを確認し、「OK」ボタンを押します。
![](https://storage.googleapis.com/zenn-user-upload/8522c2118585-20240902.png)

⑦「アクティブなタスク」欄にタスクが追加されていたらOK
![](https://storage.googleapis.com/zenn-user-upload/42e8eb328df1-20240902.png)

### 実行確認
④で設定した時間間隔で、PowerShellが起動します。
この時、メモリ使用率が80%を超えていると、画像のようにアクセス拒否のメッセージが表示されます。
![](https://storage.googleapis.com/zenn-user-upload/131926587462-20240902.png)

メッセージ表示後にPowerShellが閉じるとタスク実行完了です。
画面が閉じない場合はEnterキーを押してください。
タスクマネージャを確認すると、メモリ使用率が急激に落ちているのが分かります。
![](https://storage.googleapis.com/zenn-user-upload/ce74c93b6f8b-20240902.png)

今回は81%→59%でしたが、場合によっては48%まで落ちます。

これで自動メモリ開放タスクの設定は完了です。