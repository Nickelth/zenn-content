---
title: "タスクスケジューラ全体を停止することは禁止されている"
emoji: "⏰"
type: "tech"
topics: ["powershell", "windows", "cli"]
published: true
---

## 停止ダメ。ゼッタイ。

### 0. はじめに

業務中、

>「タスクスケジューラ、全体停止してね」

とか言って**「タスクスケジューラのサービスごと止めよう」とする輩**が出てくる。

サーバーメンテ中にバッチ処理が動くと困るのはわかる。
だけど現実は甘くない。Windowsはタスクスケジューラを“停止できない存在”として扱っている。
止めようとすると、`Stop-Service`は黙って拒否。`-Force`も効かない。

というわけで、この記事は：

- なぜタスクスケジューラは停止できないのか
- その代わりにどうすればいいのか
- PowerShellでのタスク一時停止＆復元レシピ

という内容の **「止まらないものと戦う解説記事」**です。

### 1. なぜ止められないのか（Stop-Serviceに怒られたログ付き）

タスクスケジューラのサービスは Windows に組み込まれたシステム・タスクを多数実行しており、**それ自体が他のサービスの前提**となっている。

Microsoft の公式ガイドや技術ブログでは、サービスを停止・無効にすると**スケジュールされたバックアップやログ整理などの重要な処理が実行されず、依存サービスも起動できなくなるためシステムに重大な影響が出る** と明記している。

画像は、`Stop-Service -Name Schedule -Force`を実行してコマンドでタスクスケジューラを止めようとしたもの。

![Stop-Service -Name Schedule -Force](/images/Stop-Service%20-Name%20Schedule%20-Force.png)

以下は、公式ドキュメントや主要なブログから抜粋した情報。

> Lets users configure and schedule automated tasks on this computer. The service also hosts multiple Windows system-critical tasks. If you stop or disable this service, these tasks don't run at their scheduled times, and any services that explicitly depend on it become unable to start.
> このサービスを使用すると、ユーザーはこのコンピューター上で自動化されたタスクを構成およびスケジュールできます。また、このサービスは複数の Windows のシステムにとって重要なタスクをホストしています。このサービスを停止または無効にすると、これらのタスクは予定された時刻に実行されず、このサービスに明示的に依存している他のサービスは起動できなくなります。
@[card](https://learn.microsoft.com/en-us/windows-server/security/windows-services/security-guidelines-for-disabling-system-services-in-windows-server)

### 2. 回避策：バックアップ → 無効化 → 再有効化（PowerShell）

タスクスケジューラは止められないので中のタスクを1個ずつ無効化する作戦。
このとき再開用のタスクのみ有効化させるのがポイント。
また、停止/再開タスクはともにタスクスケジューラに登録しておく。
どうせタスクスケジューラは止まらないので外部ツールを使って自動実行予約する必要はない。

#### タスクスケジューラ登録の際の注意事項
:::message alert
必ずStop→Restartの順に実行すること
メンテ時間中に実行されることのないよう実行時刻を調整すること
:::

:::message
- 「全般」での入力事項
    - **「最上位の特権で実行する」にチェックを入れる**
:::

※チェックしないとタスクの有効/無効化、Cドライブ以下のディレクトリ操作ができなくなる

- 「操作」での入力事項
    - プログラム/スクリプト(P): `powershell.exe`
    - 引数の追加(オプション)(A): `-ExecutionPolicy Bypass -File "C:\YOUR_PATH\Stop_SchdTasks.ps1"`

上記をヒントに、AIなどでPowerShellコーディングを試してほしい。

### 3. おわり
タスクスケジューラは「止められない」という事実を知っておくだけでも、トラブルシューティングやメンテナンス時の判断に差が出る。
本記事が、実運用での混乱や「止まると思って止まらなかった問題」の予防に役立てば幸いだ。
また、上司が「サービスごと止めればいいのでは？」と言い出す前に、この記事をそっと差し出して思いとどまらせてほしい。