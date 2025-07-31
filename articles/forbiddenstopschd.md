---
title: "タスクスケジューラ全体を停止することは禁止されている"
emoji: "⏰"
type: "tech"
topics: ["powershell", "windows", "cli"]
published: false
---

## 停止ダメ。ゼッタイ。

### 0. はじめに

業務中、たまに耳にする。

>「タスクスケジューラ、止めとけばよくない？」
>…うん、それ、止められたら苦労してないよね。

サーバーメンテ中にバッチ処理が動くと困る。だから**「タスクスケジューラのサービスごと止めよう」とする人**が出てくる。

だけど現実は甘くない。Windowsはタスクスケジューラを“停止できない存在”として扱っている。
止めようとすると、`Stop-Service`は黙って拒否。`-Force`も効かない。

というわけで、この記事は：

- なぜタスクスケジューラは停止できないのか
- その代わりにどうすればいいのか
- PowerShellでのタスク一時停止＆復元レシピ

をお届けする、**世にも珍しい「止まらないものと戦う記事」**です。

### 1. なぜ止められないのか（Stop-Serviceに怒られたログ付き）

タスクスケジューラのサービスは Windows に組み込まれたシステム・タスクを多数実行しており、**それ自体が他のサービスの前提**となっている。

Microsoft の公式ガイドや技術ブログでは、サービスを停止・無効にすると**スケジュールされたバックアップやログ整理などの重要な処理が実行されず、依存サービスも起動できなくなるためシステムに重大な影響が出る** と明記している。

画像は、`Stop-Service -Name Schedule -Force`を実行してコマンドでタスクスケジューラを止めようとしたもの。

<画像>

以下は、公式ドキュメントや主要なブログから抜粋した情報。

> Lets users configure and schedule automated tasks on this computer. The service also hosts multiple Windows system-critical tasks. If you stop or disable this service, these tasks don't run at their scheduled times, and any services that explicitly depend on it become unable to start.
> このサービスを使用すると、ユーザーはこのコンピューター上で自動化されたタスクを構成およびスケジュールできます。また、このサービスは複数の Windows のシステムにとって重要なタスクをホストしています。このサービスを停止または無効にすると、これらのタスクは予定された時刻に実行されず、このサービスに明示的に依存している他のサービスは起動できなくなります。
@[card](https://learn.microsoft.com/en-us/windows-server/security/windows-services/security-guidelines-for-disabling-system-services-in-windows-server)

> in the background of a modern Windows machine, dig through the Task Scheduler, it’s an eye-opener! To remove these, don’t do what I did and simply disable the Task Scheduler service – that’s a surefire way to kill a Windows 10 desktop.
> 実行中のスケジュールされたタスクを削除したい場合でも、タスクスケジューラサービスを単純に無効化するのは絶対にやめるべきだ。そうすると Windows 10 デスクトップが使い物にならなくなる
@[card](https://www.htg.co.uk/blog/windows-10-part-4-telemetry#:~:text=in%20the%20background%20of%20a,kill%20a%20Windows%2010%20desktop)

> システム管理のためにタスクを使用しているため、このサービスは必要です。デフォルトの"自動"のままにします。
> そもそも管理ツール(services.msc)ではメニューがグレイアウト化されていて停止したり無効化したりできないようです。
@[card](https://tooljp.com/Windows10/doc/Service/Task_Scheduler.html)


### 2. 回避策：バックアップ → 無効化 → 再有効化（PowerShell）

- **必ずStop→Restartの順に実行すること**

#### タスクスケジューラ登録の際の注意事項
- 「全般」での入力事項
    - **「最上位の特権で実行する」にチェックを入れる**
:::message
チェックしないとタスクの有効/無効化、Cドライブ以下のディレクトリ操作ができなくなる
:::
- 「操作」での入力事項
    - プログラム/スクリプト(P): `powershell.exe`
    - 引数の追加(オプション)(A): `-ExecutionPolicy Bypass -File "C:\YOUR_PATH\Stop_SchdTasks.ps1"`

想定されるディレクトリ構成(実行前に準備すること)
```plaintext
C:\
└── YOUR_PATH
    ├── Stop_SchdTasks.ps1
    ├── Restart_SchdTasks.ps1
    ├── log
    │   ├── Stop_SchdTasks
    │   └── Restart_SchdTasks
    ├── xml
    └── csv
```

#### ソースコード仕様
##### Stop_SchdTasks.ps1
- `C:\YOUR_PATH\log\Stop_SchdTasks\`配下に実行ログを出力
- 停止前に実行タスクの一覧をバックアップし、`\xml\`に出力
    - 実行タスク一覧は`XML`で出力後、`CSV`に変換され`\csv\`に出力される。
- 停止前に無効化されているタスクの一覧をバックアップし、`\xml\`に出力
    - 無効化タスクの一覧は`XML`で出力後、`CSV`に変換され`\csv\`に出力される。
- 処理終了後、`Restart_SchdTasks`を除き、すべてのタスクが無効化される。

```powershell:Stop_SchdTasks.ps1
# 非致命的なエラーもキャッチ
$ErrorActionPreference = "Stop"

try {
    # タスク基本情報の設定
    $timestamp = Get-Date -Format "yyyyMMddHHmmss"
    $taskName = "Stop_SchdTasks"
    $scriptName = "$taskName.ps1"
    $logFileName = "$taskName" + "_" + $timestamp + ".log"
    $logFilePath = Join-Path -Path "C:\YOUR_PATH\log\$taskName" -ChildPath $logFileName

    Add-Content -Path $logFilePath -Value "$(Get-Date -Format 'yyyy/MM/dd HH:mm:ss') INF $taskName $scriptName 処理開始"

    # 実行タスク一覧をバックアップ
    Get-ScheduledTask | Export-Clixml -Path ".\xml\scheduled_tasks_backup.xml"

    #　無効化されているタスクをバックアップ
    Get-ScheduledTask | Where-Object { $_.State -eq 'Disabled' } | Export-Clixml -Path ".\xml\disabled_tasks_before_maintenance.xml"

    # タスクスケジューラ中のタスクをすべて停止
    Get-ScheduledTask | Where-Object { $_.TaskPath -notlike "\Microsoft\Windows\*" } | Disable-ScheduledTask

    $tasks = Import-Clixml -Path ".\xml\scheduled_tasks_backup.xml"
    $tasks | Select-Object TaskName, TaskPath, State | Export-Csv -Path .\csv\disabled_tasks_list.csv -NoTypeInformation
    $tasks = Import-Clixml -Path ".\xml\disabled_tasks_before_maintenance.xml"
    $tasks | Select-Object TaskName, TaskPath, State | Export-Csv -Path .\csv\disabled_tasks_before_maintenance.csv -NoTypeInformation

    # Restart_SchdTasks.ps1を有効化
    Enable-ScheduledTask -TaskName "Restart_SchdTasks"

    Add-Content -Path $logFilePath -Value "$(Get-Date -Format 'yyyy/MM/dd HH:mm:ss') INF $taskName $scriptName 処理完了"
}
catch {
    $timestamp = Get-Date -Format "yyyy/MM/dd HH:mm:ss"
    $errorMessage = "$timestamp - エラー: $($_.Exception.Message)`n$($_.ScriptStackTrace)"

    Add-Content -Path $logFilePath -Value "$(Get-Date -Format 'yyyy/MM/dd HH:mm:ss') ERR $taskName $scriptName 異常終了"
    Add-content -Path $logFilePath -Value $errorMessage
}
```

##### Restart_SchdTasks.ps1
- `C:\YOUR_PATH\log\Stop_SchdTasks\`配下に実行ログを出力
- 処理開始時、タスクスケジューラのすべてのタスクを有効化
- その後、バックアップされた無効化されているタスク一覧XMLを元にタスクを無効化し、停止前の状態を再現


```powershell:Restart_SchdTasks.ps1
# 非致命的なエラーもキャッチ
$ErrorActionPreference = "Stop"

try {
    # タスク基本情報の設定
    $timestamp = Get-Date -Format "yyyyMMddHHmmss"
    $taskName = "Restart_SchdTasks"
    $scriptName = "$taskName.ps1"
    $logFileName = "$taskName" + "_" + $timestamp + ".log"
    $logFilePath = Join-Path -Path "C:\YOUR_PATH\log\$taskName" -ChildPath $logFileName

    Add-Content -Path $logFilePath -Value "$(Get-Date -Format 'yyyy/MM/dd HH:mm:ss') INF $taskName $scriptName 処理開始"

    #タスクスケジューラ中のタスクをすべて有効化
    Get-ScheduledTask | Where-Object { $_.TaskPath -notlike "\Microsoft\Windows\*" } | Enable-ScheduledTask

    # 復旧前に無効化だったタスクを再び無効化する
    $disabledTasks = Import-Clixml -Path ".\xml\disabled_tasks_before_maintenance.xml"
    $disabledTasks | ForEach-Object {
        Disable-ScheduledTask -TaskName $_.TaskName -TaskPath $_.TaskPath
    }

    Add-Content -Path $logFilePath -Value "$(Get-Date -Format 'yyyy/MM/dd HH:mm:ss') INF $taskName $scriptName 処理完了"
}
catch {
    $timestamp = Get-Date -Format "yyyy/MM/dd HH:mm:ss"
    $errorMessage = "エラー: $($_.Exception.Message)`n$($_.ScriptStackTrace)"

    Add-Content -Path $logFilePath -Value "$(Get-Date -Format 'yyyy/MM/dd HH:mm:ss') ERR $taskName $scriptName 異常終了"
    Add-content -Path $logFilePath -Value $errorMessage
}
```

### 3. よくある罠

- 「全般」での入力事項
    - **「最上位の特権で実行する」にチェックを入れる**
:::message
チェックしないとタスクの有効/無効化、Cドライブ以下のディレクトリ操作ができなくなる
:::
- 「操作」での入力事項
    - プログラム/スクリプト(P): `powershell.exe`
    - 引数の追加(オプション)(A): `-ExecutionPolicy Bypass -File "C:\YOUR_PATH\Stop_SchdTasks.ps1"`

### 4. おわり
タスクスケジューラは「止められない」という事実を知っておくだけでも、トラブルシューティングやメンテナンス時の判断に差が出る。
本記事が、実運用での混乱や「止まると思って止まらなかった問題」の予防に役立てば幸いだ。
また、上司が「サービスごと止めればいいのでは？」と言い出す前に、この記事をそっと差し出して思いとどまらせてほしい。