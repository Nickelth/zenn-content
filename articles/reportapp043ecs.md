---
title: "【#4 3/4】AWS基盤にECS/Fargateを構築する"
emoji: "🚢"
type: "tech"
topics: ["ecs", "aws"]
published: false
---

## ECS/Fargate構築編（クラスタ/タスク/サービス/ALB）

### 0. はじめに

**ECS Fargate構築**

* クラスター、サービス、タスク定義（最初はダミーイメージでもOK）
* タスク実行ロール（`ecsTaskExecutionRole`）とアプリ用ロール作成

### 1. ECSクラスターをコンソールで作成する。

1. ECSのトップ画面で「今すぐ始める」＞「クラスターを作成」をクリック

2. クラスターを作成する。
コスト節約を重視するので、
- Fargateを選択（インスタンス管理なし＆秒課金。デモ向き）
- Container Insightsはオフ（CloudWatchメトリクスの追加課金を回避）
- 暗号化: KMSキー未指定 （Fargateの一時ストレージはデフォでAWS管理キーで暗号化。KMSカスタムは不要・コスト増）
![クラスターを作成する](https://storage.googleapis.com/zenn-user-upload/acde573a7c37-20250813.png)

:::details クラスター作成に失敗する場合
** リソース ECSClusterは CREATE_FAILED状態です **
![リソース ECSClusterは CREATE_FAILED状態です](https://storage.googleapis.com/zenn-user-upload/9fbe920f901b-20250813.png)

「CloudFormation」＞スタックから、ステータスが「失敗」のスタックを選択肢、右上の「削除」を押してスタックを削除する。
![スタック削除](https://storage.googleapis.com/zenn-user-upload/4f08ffcc54fe-20250813.png)

削除後、もう一度クラスターを作成する。
:::

### 2. ECSタスク定義を作成する。

1. ECS＞「タスク定義」から、「新しいタスク定義の作成」＞「JSONを使用した新しいタスク定義の作成」をクリック
![JSONを使用した新しいタスク定義の作成](https://storage.googleapis.com/zenn-user-upload/5b8140bb6c54-20250813.png)

2. 以下のJSONをタスク定義のエディターに入力。
`<>`の変数は各自置き換える。

|変数名|値|
|---|---|
|`<TASK_FAMILY>`|好きにつけてOK(英数字,ハイフン,アンダースコア)|
|`<TASK_EXECUTION_ROLE_ARN>` |`arn:aws:iam::<ACCOUNT_ID>:role/ecsTaskExecutionRole`
|`<CONTAINER_NAME>`|`app`|

`environment`と`secrets`に`.env`ファイルで使用した環境変数を定義する。


:::details taskdef.json
```json
{
  "family": "<TASK_FAMILY>",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "<TASK_EXECUTION_ROLE_ARN>",
  "containerDefinitions": [
    {
      "name": "<CONTAINER_NAME>",
      "image": "public.ecr.aws/docker/library/nginx:alpine",
      "essential": true,
      "portMappings": [{ "containerPort": 5000, "protocol": "tcp" }],
      "environment": [
        { "name": "AUTH0_DOMAIN",       "value": "arn:aws:ssm:<REGION>:<ACCOUNT_ID>:parameter/papyrus/prd/AUTH0_DOMAIN" },
        { "name": "AUTH0_CLIENT_ID",    "value": "arn:aws:ssm:<REGION>:<ACCOUNT_ID>:parameter/papyrus/prd/AUTH0_CLIENT_ID" },
        { "name": "AUTH0_CALLBACK_URL", "value": "arn:aws:ssm:<REGION>:<ACCOUNT_ID>:parameter/papyrus/prd/AUTH0_CALLBACK_URL" }
      ],
      "secrets": [
        { "name": "AUTH0_CLIENT_SECRET", "valueFrom": "arn:aws:ssm:<REGION>:<ACCOUNT_ID>:parameter/papyrus/prd/AUTH0_CLIENT_SECRET" },
        { "name": "POSTGRES_USER",        "valueFrom": "arn:aws:ssm:<REGION>:<ACCOUNT_ID>:parameter/papyrus/prd/POSTGRES_USER" },
        { "name": "POSTGRES_PASSWORD",        "valueFrom": "arn:aws:ssm:<REGION>:<ACCOUNT_ID>:parameter/papyrus/prd/POSTGRES_PASSWORD" },
        { "name": "POSTGRES_DB",        "valueFrom": "arn:aws:ssm:<REGION>:<ACCOUNT_ID>:parameter/papyrus/prd/POSTGRES_DB" },
        { "name": "FLASK_SECRET_KEY",        "valueFrom": "arn:aws:ssm:<REGION>:<ACCOUNT_ID>:parameter/papyrus/prd/FLASK_SECRET_KEY" }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-region": "<REGION>",
          "awslogs-group": "<LOG_GROUP_NAME>",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ],
  "runtimePlatform": {"cpuArchitecture": "arm64", "operatingSystemFamily": "LINUX"},
  "tags": [{"key":"project","value":"portfolio"}]
}
```
:::

※awslogs の設定はそのまま（※ロググループは先に作っておくこと）

### 3. ECSクラスター内にサービスを作成する。

**サービスの詳細**
|項目|値|
|---|---|
|タスク定義ファミリー|先ほど定義したタスク定義名|
|サービス名|自分で決める|

**デプロイ設定**
|項目|値|
|---|---|
|起動するタスク|`0`|

*この設定により、Desired=0となる*

**ネットワーキング**
|項目|値|
|---|---|
|サブネット|aとbの2つを選択|
|セキュリティグループ|先ほど作成したやつ|

これ以外はすべてデフォルトでOK


### 4. 動作確認 & トラブルシュート