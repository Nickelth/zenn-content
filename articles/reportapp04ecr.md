---
title: "【#4】GitHub ActionsでDockerをECRに登録する手順"
emoji: "🦄"
type: "tech"
topics: ["githubactions", "docker", "ecr", "github"]
published: false
---

## GitHub Actions登録手順

### 0. はじめに

採用担当に「お、この人ちゃんとCI/CDの流れ分かってるじゃん」と思わせたいなら、**完全自動化パターン**一択です。

---

## 評価高めの順番（完全自動構築）

1. **ECR作成**

   * プライベートリポジトリ
   * タグ=Mutable, 暗号化=AES-256

2. **ECS Fargate構築**

   * クラスター、サービス、タスク定義（最初はダミーイメージでもOK）
   * タスク実行ロール（`ecsTaskExecutionRole`）とアプリ用ロール作成

3. **GitHub Actions設定**（ECR push + ECS更新までワンストップ）

   * OIDCロール作成（信頼ポリシーに`sub`/`aud`）
   * 権限ポリシーにECRとECS更新（`ecs:RegisterTaskDefinition`、`ecs:UpdateService`、`iam:PassRole`）追加
   * `ecr-push.yml`に続いて、`ecs-deploy.yml`も作る
   * ワークフローが動くと：

     1. Docker Build
     2. ECR push
     3. ECSタスク定義更新
     4. サービス反映

4. **テスト & 証拠動画**

   * `workflow_dispatch`で実行して、ECSのタスクが自動で更新される様子を画面収録
   * 「コードpush → 本番更新」の流れを丸ごと見せられる


#### 使用技術の選定理由

### 1. IAMロール作成

#### 1-1. ECSTaskExecutionロールの作成
1. IAM → ロール → ロールを作成

2. 信頼されたエンティティの種類：AWS のサービス
    - 使⽤事例：Elastic Container Service → Elastic Container Service Task を選択
        - （これで信頼ポリシー Principal: ecs-tasks.amazonaws.com が自動セットされる）

3. アクセス権限で AmazonECSTaskExecutionRolePolicy にチェック

3. ロール名：ecsTaskExecutionRole→ 作成

4. 作成後の画面で ARN をコピー → taskdef.json の executionRoleArn に貼る

5. GitHubのOIDCロールの iam:PassRole の Resource にこのARNを必ず入れる（忘れるとまた怒られる）

#### 2.2 IAM ID プロバイダを作成
IAM＞アクセス管理＞ID プロバイダを選択

「プロバイダを追加」をクリック

2. プロバイダータイプ → OpenID Connect

3. プロバイダーURL →
```plaintext
https://token.actions.githubusercontent.com
```

4. 対象者 (Audience) →
```plaintext
sts.amazonaws.com
```

5. サムプリント（Thumbprint）欄は自動で埋まるか、AWSのデフォ値 `6938fd4d98bab03faadb97b34396831e3780aea1` を指定

6. 作成完了すると、「プロバイダーを選択」のプルダウンに**token.actions.githubusercontent.com**が出てくる

#### 2.3 IAM ロールを作成

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

#### 2.4 作成したIAMロールの信頼ポリシーを編集
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

:::default 信頼ポリシーのJSON
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

```json:Condition部分の一例
      "Condition": {
          "StringEquals": {
              "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": [
            "repo:Nickelth/Papyrus:ref:refs/heads/main"
          ]
        }
      }
```

3. アクセスポリシーを編集する。
先ほど作成したロールから、「許可」タブ→「許可を追加」トグル→「インラインポリシーを作成」を選択
![インラインポリシーを作成](https://storage.googleapis.com/zenn-user-upload/4b3a8c9f065d-20250812.png)
`<>`の部分(`<REGION>`, `<ACCOUNT_ID>`, `<ECR_REPOSITORY>`, `<CLUSTER>`, `<SERVICE>`, `<TASK_FAMILY>`)は置き換え
|変数名|コピペ元|
|---|---|
|<CLUSTER>|ECS → クラスター → 対象クラスターの名前|
|<SERVICE>|ECS → クラスター → サービスタブ → 対象サービスの名前|
|<TASK_FAMILY>|ECS → タスク定義 → 対象のファミリー名（:1みたいなリビジョン番号は不要）|
|<TASK_EXECUTION_ROLE_ARN> / <TASK_ROLE_ARN>|ECS → タスク定義 → 対象リビジョンを開き、JSONタブの`executionRoleArn`（実行ロール）と `taskRoleArn`（アプリ用ロール）をそのままコピペ。|

:::default アクセス権限のJSON
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
      "Sid": "ECSDescribe",
      "Effect": "Allow",
      "Action": [
        "ecs:DescribeClusters",
        "ecs:DescribeServices",
        "ecs:DescribeTaskDefinition"
      ],
      "Resource": [
        "arn:aws:ecs:<REGION>:<ACCOUNT_ID>:cluster/<CLUSTER>",
        "arn:aws:ecs:<REGION>:<ACCOUNT_ID>:service/<CLUSTER>/<SERVICE>",
        "arn:aws:ecs:<REGION>:<ACCOUNT_ID>:task-definition/<TASK_FAMILY>*"
      ]
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



### 2. ECRの作成

#### 2.1 
イメージタグはミュータブル、暗号化設定はAESを選択する。
![ECRのミュータブル・暗号化設定](https://storage.googleapis.com/zenn-user-upload/ee057d171e1f-20250811.png)

ECR作成完了
![ECR](https://storage.googleapis.com/zenn-user-upload/0db90c13e04f-20250811.png)





### 3. ECSの設定をする。
#### 3.1 ECSクラスターをコンソールで作成する。
1. ECSのトップ画面で「今すぐ始める」＞「クラスターを作成」をクリック

2. クラスターを作成する。
コスト節約を重視するので、
- Fargateを選択（インスタンス管理なし＆秒課金。デモ向き）
- Container Insightsはオフ（CloudWatchメトリクスの追加課金を回避）
- 暗号化: KMSキー未指定 （Fargateの一時ストレージはデフォでAWS管理キーで暗号化。KMSカスタムは不要・コスト増）
![クラスターを作成する](https://storage.googleapis.com/zenn-user-upload/acde573a7c37-20250813.png)

:::default クラスター作成に失敗する場合
** リソース ECSClusterは CREATE_FAILED状態です **
![リソース ECSClusterは CREATE_FAILED状態です](https://storage.googleapis.com/zenn-user-upload/9fbe920f901b-20250813.png)

「CloudFormation」＞スタックから、ステータスが「失敗」のスタックを選択肢、右上の「削除」を押してスタックを削除する。
![スタック削除](https://storage.googleapis.com/zenn-user-upload/4f08ffcc54fe-20250813.png)

削除後、もう一度クラスターを作成する。
:::

#### 3.2 ECSタスク定義を作成する。
1. ECS＞「タスク定義」から、「新しいタスク定義の作成」＞「JSONを使用した新しいタスク定義の作成」をクリック
![JSONを使用した新しいタスク定義の作成](https://storage.googleapis.com/zenn-user-upload/5b8140bb6c54-20250813.png)

2. 以下のJSONをタスク定義のエディターに入力。
`<>`の変数は各自置き換える。
|変数名|値|
|---|---|
|`<TASK_FAMILY>`|好きにつけてOK(英数字,ハイフン,アンダースコア)|
|`<TASK_EXECUTION_ROLE_ARN>` |arn:aws:iam::<ACCOUNT_ID>:role/ecsTaskExecutionRole
|`<CONTAINER_NAME>`|`app`|

:::default taskdef.json
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

※awslogs の設定はそのまま（※ロググループは先に作っておくこと）

#### 3.3 ECSクラスター内にサービスを作成する。





### 5. GitHubでの操作

「該当リポジトリ」＞「Setting」＞「Actions」をクリック

![Actionsを選択](https://storage.googleapis.com/zenn-user-upload/0cafadbbe168-20250810.png)

![Settings](https://storage.googleapis.com/zenn-user-upload/68051b626775-20250810.png)

![SecurityからActionsへ](https://storage.googleapis.com/zenn-user-upload/5174213ae1e2-20250810.png)

![Variablesを選択](https://storage.googleapis.com/zenn-user-upload/db71600d3fcd-20250810.png)

「Maneage environment variables」＞「New environment」の順にクリックして以下の変数を記入する。

![Repository Variablesをクリック](https://storage.googleapis.com/zenn-user-upload/e543e79be625-20250811.png)

![New Variables](https://storage.googleapis.com/zenn-user-upload/21ee424ba83f-20250811.png)

機密情報を入力する場合は「Secrets」タブに切り替える。
「Actions secrets/New secret」の画面が出るので、同様に入力する。

![Add Secrets](https://storage.googleapis.com/zenn-user-upload/6874b3b685d6-20250811.png)

```plaintext
AWS_ACCOUNT_ID

AWS_REGION（例: ap-northeast-1）

ECR_REPOSITORY（例: papyrus）

AWS_IAM_ROLE_ARN（GitHub OIDCを信頼するIAMロールのARN）


```
token.actions.githubusercontent.com
#### アプリ構成図