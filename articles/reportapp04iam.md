---
title: "【#4】アプリ本番環境公開に向けてAWS IAMを設定する"
emoji: "🛡️"
type: "tech"
topics: ["iam", "aws"]
published: false
---

## AWS基盤を準備する（IAM編）

### 0. はじめに

前回の構築までは失敗談の体で記事を投稿していたが、今回からは本業ということもありまじめに記述する。
Flaskアプリをネット上に公開し、アクセスできることを目標とする。

#### 要件定義

個人で運用するポートフォリオ用のアプリなので、**コスト管理**を最大限重視する。
- リージョンは価格が比較的安い米国オレゴン州(`us-west-2`)を採用する。
- GithubActionsを用いて必要な時にのみECS/Fargateを手動起動・停止できるようにする。(従属課金)
- デプロイ時にCI/CDを自動実行する。
- 課金対策のため、ALBは使用しない。

IAM設定が異様に長いので、この記事ではIAMの設定のみを対象に解説する。

#### 設定するIAMロール2種類について

**ECRPowerUser**
|key|value|
|---|---|
|ポリシー|
|用途|

**ecsTaskExecutionRole**
|key|value|
|---|---|
|ポリシー|`AmazonECSTaskExecutionRolePolicy`|
|用途|

### 1. IAM最小権限ユーザの作成




### 2. ECSTaskExecutionロールの作成

1. IAM → ロール → ロールを作成

2. 信頼されたエンティティの種類：AWS のサービス
    - 使⽤事例：Elastic Container Service → Elastic Container Service Task を選択
        - （これで信頼ポリシー Principal: ecs-tasks.amazonaws.com が自動セットされる）

3. アクセス権限で `AmazonECSTaskExecutionRolePolicy` にチェック

3. ロール名：`ecsTaskExecutionRole`→ 作成

4. 作成後の画面で ARN をコピー → `taskdef.json` の `executionRoleArn` に貼る

5. GitHubのOIDCロールの iam:PassRole の Resource にこのARNを必ず入れる（忘れるとまた怒られる）

### 3. IAM ID プロバイダを作成

IAM＞アクセス管理＞ID プロバイダを選択

「プロバイダを追加」をクリック

2. プロバイダータイプ → OpenID Connect

3. プロバイダーURL には以下を入力
```plaintext
https://token.actions.githubusercontent.com
```

4. 対象者 (Audience) には以下を入力
```plaintext
sts.amazonaws.com
```

5. サムプリント（Thumbprint）欄は自動で埋まるか、AWSのデフォ値 `6938fd4d98bab03faadb97b34396831e3780aea1` を指定

6. 作成完了すると、「プロバイダーを選択」のプルダウンに**token.actions.githubusercontent.com**が出てくる

### 4. IAM ロールを作成

1. IAM＞ロール＞ロールを作成

2. 「信頼されたエンティティを選択」画面で以下の項目を入力

|key|value|
|---|---|
|信頼されたエンティティタイプ|ウェブアイデンティティ|
|ウェブアイデンティティ|`token.actions.githubusercontent.com`|
|Audience|`sts.amazon.com`|
|GitHub Organization|(任意名称)|

3. 「許可を追加」画面
`AmazonEC2ContainerRegistryPowerUser`を付与して「次へ」

4. 「ロールを作成」をクリックし、IAM ロール作成完了

### 5. 作成したIAMロールの信頼ポリシーを編集

IAMのポリシーエディタは **「信頼ポリシー」と「アクセス許可ポリシー」を別々に登録する」** 仕組みなので、両方編集する。

1. IAM → ロール → 該当ロール → 信頼関係（Trust relationships）タブ → 信頼ポリシーの編集
![信頼関係をクリック](https://storage.googleapis.com/zenn-user-upload/fa659ad32d38-20250811.png)

2. 信頼ポリシーを編集する。
- `<>`の部分(`<ACCOUNT_ID>`, `<ORG>`, `<REPO_NAME>`)は置き換え
- `<REPO_NAME>`はGitHubリポジトリのリポジトリ名
- `aud` は固定で `sts.amazonaws.com`
- `sub` はGitHubリポジトリとブランチに合わせる
- メインブランチだけ許可するなら `ref:refs/heads/main`
- タグも許可するなら `ref:refs/tags/*` を追加

:::details 信頼ポリシーのJSON
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": [
            "repo:<ORG>/<REPO_NAME>:ref:refs/heads/main",
            "repo:<ORG>/<REPO_NAME>:ref:refs/tags/v*.*.*"
          ]
        }
      }
    }
  ]
}
```
:::

### 6. 作成したIAMロールのアクセスポリシーを編集する。

先ほど作成したロールから、「許可」タブ→「許可を追加」トグル→「インラインポリシーを作成」を選択
![インラインポリシーを作成](https://storage.googleapis.com/zenn-user-upload/4b3a8c9f065d-20250812.png)
`<>`の部分(`<REGION>`, `<ACCOUNT_ID>`, `<ECR_REPOSITORY>`, `<CLUSTER>`, `<SERVICE>`, `<TASK_FAMILY>`)は置き換え

|変数名|コピペ元|
|---|---|
|`<CLUSTER>`|ECS → クラスター → 対象クラスターの名前|
|`<SERVICE>`|ECS → クラスター → サービスタブ → 対象サービスの名前|
|`<TASK_FAMILY>`|ECS → タスク定義 → 対象のファミリー名（:1みたいなリビジョン番号は不要）|
|`<TASK_EXECUTION_ROLE_ARN>`|ECS → タスク定義 → 対象リビジョンを開き、JSONタブの`executionRoleArn`（実行ロール）と `taskRoleArn`（アプリ用ロール）をそのままコピペ。|

:::details アクセス権限のJSON
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ECRAuth",
      "Effect": "Allow",
      "Action": ["ecr:GetAuthorizationToken"],
      "Resource": "*"
    },
    {
      "Sid": "ECRPushPullRepoScoped",
      "Effect": "Allow",
      "Action": [
        "ecr:BatchCheckLayerAvailability",
        "ecr:CompleteLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:InitiateLayerUpload",
        "ecr:PutImage",
        "ecr:BatchGetImage",
        "ecr:DescribeRepositories",
        "ecr:ListImages"
      ],
      "Resource": "arn:aws:ecr:<REGION>:<ACCOUNT_ID>:repository/<ECR_REPOSITORY>"
    },
    {
      "Sid": "ECRCreateRepoOptional",
      "Effect": "Allow",
      "Action": ["ecr:CreateRepository"],
      "Resource": "*",
      "Condition": {
        "StringEquals": {"aws:RequestedRegion": "<REGION>"}
      }
    },
    {
      "Sid": "ECSDescribeClusterService",
      "Effect": "Allow",
      "Action": [
        "ecs:DescribeClusters",
        "ecs:DescribeServices"
      ],
      "Resource": [
        "arn:aws:ecs:<REGION>:<ACCOUNT_ID>:cluster/<CLUSTER>",
        "arn:aws:ecs:<REGION>:<ACCOUNT_ID>:service/<CLUSTER>/<SERVICE>",
      ]
    },
    {
      "Sid": "ECSDescribeTaskDefinitionAny",
      "Effect": "Allow",
      "Action": ["ecs:DescribeTaskDefinition"],
      "Resource": "*",
      "Condition": { "StringEquals": { "aws:RequestedRegion": "<REGION>" } }
    },
    {
      "Sid": "ECSRegisterTaskDef",
      "Effect": "Allow",
      "Action": ["ecs:RegisterTaskDefinition"],
      "Resource": "*"
    },
    {
      "Sid": "ECSUpdateService",
      "Effect": "Allow",
      "Action": ["ecs:UpdateService"],
      "Resource": "arn:aws:ecs:<REGION>:<ACCOUNT_ID>:service/<CLUSTER>/<SERVICE>"
    },
    {
      "Sid": "PassTaskRoles",
      "Effect": "Allow",
      "Action": ["iam:PassRole"],
      "Resource": [
        "<TASK_EXECUTION_ROLE_ARN>",
        "<TASK_ROLE_ARN>"
      ],
      "Condition": {
        "StringEquals": { "iam:PassedToService": "ecs-tasks.amazonaws.com" }
      }
    }
  ]
}
```
:::


4. 作成したポリシーをロールにアタッチする。
先ほどと同じように「許可」タブ→「許可を追加」トグル→「インラインポリシーを作成」を選択

![ポリシーをアタッチ](https://storage.googleapis.com/zenn-user-upload/819c4a6676f1-20250812.png)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    { "Sid": "LogsCreateGroup",
      "Effect": "Allow",
      "Action": ["logs:CreateLogGroup","logs:DescribeLogGroups"],
      "Resource": "*",
      "Condition": { "StringEquals": { "aws:RequestedRegion": "us-west-2" } }
    },
    { "Sid": "LogsSetRetention",
      "Effect": "Allow",
      "Action": ["logs:PutRetentionPolicy","logs:TagLogGroup"],
      "Resource": "arn:aws:logs:us-west-2:438336773404:log-group:/ecs/papyrus"
    }
  ]
}
```

### 7. チェックリスト（ここまでの出来上がり）

<!--
### 8. 次の記事

@[card](https://zenn.dev/nickelth/articles/reportapp05aws)
-->