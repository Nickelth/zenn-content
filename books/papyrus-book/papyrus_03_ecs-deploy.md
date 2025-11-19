---
title: "CI: ECS Deploy（TaskDef更新 → DesiredCount制御）"
---

## ECSタスク数とイメージ切り替えを CI から制御する

### 目的

- 手動トリガー（workflow_dispatch）で、指定した **ECR イメージタグ** を ECS サービスに反映する
- `desired_count` を入力として受け取り、**タスク数を CI から制御**できるようにする
- `force-new-deployment` を使い、**ロールバックしやすい単純なローリングデプロイ**に統一する  
  （※Strict な「完全無停止」ではなく、DesiredCount/デプロイ設定に依存）

### 前提（クラスター / サービス）

- 事前に以下が作成済みであること
  - Fargate 用 ECS クラスター：`ECS_CLUSTER`
  - サービス：`ECS_SERVICE`（`TASK_FAMILY` を元にタスク定義を参照）
  - コンテナ名：`CONTAINER_NAME`
- ECS サービスのデプロイ設定（min/max percent）はクラスター側で定義済み  
  → CI からは「どのイメージタグを使うか」「DesiredCount をいくつにするか」だけを操作

### ワークフロー設計

#### トリガーと入力

- `workflow_dispatch` で手動起動
  - `image_tag`: デプロイ対象の ECR イメージタグ（例: `latest` や コミット SHA タグ）
  - `desired_count`: サービスの Desired タスク数（デフォルト `'1'`）

#### タスク定義の更新戦略（タグ差し込み）

- 実際のフロー：
  1. `aws ecs describe-task-definition` で **現行タスク定義をそのまま JSON 取得**（`taskdef.json`）
  2. `amazon-ecs-render-task-definition` で、`container-name` に対して  
     `registry/repo:${{ inputs.image_tag }}` を差し込み
  3. 生成されたタスク定義 JSON を `amazon-ecs-deploy-task-definition` に渡す

- ポイント：
  - **タスク定義テンプレートをリポジトリ側で持たず、ECS から逆輸入する形**にしている  
    → 環境差分や細かいパラメータを IaC / コンソールで調整しても CI が壊れにくい
  - CI 側では「どのタグを使うか」だけを責務にする

#### Preflight チェック（ARN解決）

- `describe-clusters` / `describe-services` で
  - `ECS_CLUSTER` / `ECS_SERVICE` が **本当に存在するかを事前チェック**
  - `clusterArn` / `serviceArn` を出力に保存し、後続ステップで使用
- クラスタ名やサービス名の typoエラーを防ぎ、対象が存在しない」系のミスを冒頭で明示的に止める。

#### force-new-deployment の扱い

- `amazon-ecs-deploy-task-definition@v1` で以下を指定：

  - `desired-count: ${{ inputs.desired_count }}`
  - `force-new-deployment: true`
  - `wait-for-service-stability: true`

- 挙動：
  - 新しいタスク定義リビジョンを登録 → サービスの `taskDefinition` を差し替え
  - ECS のデプロイ設定（min/max percent）に従って、**ローリングでタスクを入れ替え**
  - 安定状態になるまで CI が待機する

### 実装ポイント

#### 失敗時ロールバック

- ロールバック方法はシンプルに「**前のイメージタグ or 前のタスク定義リビジョンに戻す**」

  - パターン1：CI から再実行  
    - `image_tag` に以前のタグ（例: 直前の SHA タグ）を指定して再デプロイ
  - パターン2：AWS コンソール / CLI から  
    - `UpdateService` で過去の taskDefinition リビジョンを再指定

- 「どのタグ／どのリビジョンがいつ使われたか」は別途証跡（後述）で残す。

#### バージョニング（タスク定義リビジョン）

- すべてのデプロイは ECS の `taskDefinition: <family>:<revision>` として記録される
- CI は常に「最新リビジョンをベースに、コンテナイメージだけ差し替えて新リビジョンを登録」する形
- これにより、**設定を少し変えたデプロイも含めて履歴を追いやすく**している

### 証跡の取り方

- CI 実行時のログとは別に、最低限以下を `docs/evidence` に保存：

  - `aws ecs describe-services` の結果（対象サービスの `deployments` 情報）
  - ECS サービスのイベントログ（新旧デプロイの状態遷移が分かる部分の抜粋）
  - `aws ecs describe-task-definition` の出力（どのリビジョンがどのイメージを参照しているか）

- 例：  
  - `20260101_010203_ecs_describe-services.json`  
  - `20260101_010203_ecs_taskdef_papyrus-task-42.json`

### 失敗例と対策

- **タスク起動失敗（STOPPED連発）**

  - 原因例：
    - Secrets / 環境変数のキー名変更に対してアプリが未対応
    - SG 誤設定により RDS に到達できない
    - `image_tag` が存在しない（ECR にそのタグがない）

  - 対策：
    - デプロイ後は CloudWatch Logs でコンテナの起動ログを確認
    - `describe-tasks` → `stoppedReason` で明示的に原因を特定
    - `image_tag` は ECR push CI と揃える（SHA タグ運用など）

### セキュリティ注意

- **Execution Role**
  - ECR からのイメージ取得 / CloudWatch Logs 書き込み / Secrets Manager 読み取りに必要な権限のみ
- **Task Role**
  - アプリから参照する SSM パラメータなど、実行時に必要なものだけを付与
- CI 用 IAM ロールとは分離し、**「デプロイする権限」と「アプリが実行時に持つ権限」を明確に分ける**。

### 付録（CI YAML 抜粋の位置づけ）

- このワークフローでは、GitHub Actions の公式アクションを使い、  
  内部的に以下の ECS API をラップしている：

  - `DescribeTaskDefinition`
  - `RegisterTaskDefinition`
  - `UpdateService`

- 主要ステップ：
  - `amazon-ecs-render-task-definition@v1`  
    → 既存タスク定義 JSON に新しいイメージタグを差し込む
  - `amazon-ecs-deploy-task-definition@v1`  
    → タスク定義リビジョンを登録し、サービスを更新してデプロイ