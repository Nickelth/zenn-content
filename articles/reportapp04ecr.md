---
title: "【#4】GitHub ActionsでDockerをECRに登録する手順"
emoji: "🦄"
type: "tech"
topics: ["githubactions", "docker", "ecr", "github"]
published: false
---

## ECR作成→ECS Fargate構築→GitHub Actions設定

### 0. はじめに

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

#### 1-2. ロググループ作成

##### コンソールで作成
1. CloudWatch → ログ → ロググループ → Create log group

2. Name: /ecs/papyrus、Region: ECSと同じ

3. 保持期間: 1 day（デモ用途ならこれで十分）

##### CLIで作成
```bash
LOG_GROUP=/ecs/papyrus
REGION=us-west-2   # か ap-northeast-1

aws logs create-log-group --log-group-name "$LOG_GROUP" --region "$REGION" 2>/dev/null || true
aws logs put-retention-policy --log-group-name "$LOG_GROUP" --retention-in-days 1 --region "$REGION"
aws logs tag-log-group --log-group-name "$LOG_GROUP" --tags project=portfolio --region "$REGION"
```

4. `ecs-deploy.yml`に記述
後述の`ecs-deploy.yml`にて該当する箇所はこちら。
```yaml
- name: Ensure CloudWatch Log Group exists
  run: |
    aws logs create-log-group --log-group-name "/ecs/papyrus" --region "${{ env.AWS_REGION }}" 2>/dev/null || true
    aws logs put-retention-policy --log-group-name "/ecs/papyrus" --retention-in-days 1 --region "${{ env.AWS_REGION }}"
```

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
            "repo:<ORG>/<REPO_NAME>:ref:refs/heads/main"
          ]
        }
      }
    }
  ]
}
```
:::

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
      "environment": [
        { "name": "AUTH0_DOMAIN",       "value": "arn:aws:ssm:us-west-2:438336773404:parameter/papyrus/prd/AUTH0_DOMAIN" },
        { "name": "AUTH0_CLIENT_ID",    "value": "arn:aws:ssm:us-west-2:438336773404:parameter/papyrus/prd/AUTH0_CLIENT_ID" },
        { "name": "AUTH0_CALLBACK_URL", "value": "arn:aws:ssm:us-west-2:438336773404:parameter/papyrus/prd/AUTH0_CALLBACK_URL" },
        { "name": "GUNICORN_WORKERS",        "value": "arn:aws:ssm:us-west-2:438336773404:parameter/papyrus/prd/GUNICORN_WORKERS" },
        { "name": "GUNICORN_THREADS",        "value": "arn:aws:ssm:us-west-2:438336773404:parameter/papyrus/prd/GUNICORN_THREADS" },
        { "name": "GUNICORN_TIMEOUT",        "value": "arn:aws:ssm:us-west-2:438336773404:parameter/papyrus/prd/GUNICORN_TIMEOUT" },
        { "name": "GUNICORN_MAX_REQUESTS",        "value": "arn:aws:ssm:us-west-2:438336773404:parameter/papyrus/prd/GUNICORN_MAX_REQUESTS" },
        { "name": "GUNICORN_MAX_REQUESTS_JITTER",        "value": "arn:aws:ssm:us-west-2:438336773404:parameter/papyrus/prd/GUNICORN_MAX_REQUESTS_JITTER" }
      ],
      "secrets": [
        { "name": "AUTH0_CLIENT_SECRET", "valueFrom": "arn:aws:ssm:us-west-2:438336773404:parameter/papyrus/prd/AUTH0_CLIENT_SECRET" },
        { "name": "DATABASE_URL",        "valueFrom": "arn:aws:ssm:us-west-2:438336773404:parameter/papyrus/prd/DATABASE_URL" },
        { "name": "DB_NAME",        "valueFrom": "arn:aws:ssm:us-west-2:438336773404:parameter/papyrus/prd/DB_NAME" },
        { "name": "DB_USER",        "valueFrom": "arn:aws:ssm:us-west-2:438336773404:parameter/papyrus/prd/DB_USER" },
        { "name": "DB_PASSWORD",        "valueFrom": "arn:aws:ssm:us-west-2:438336773404:parameter/papyrus/prd/DB_PASSWORD" },
        { "name": "DB_HOST",        "valueFrom": "arn:aws:ssm:us-west-2:438336773404:parameter/papyrus/prd/DB_HOST" },
        { "name": "POSTGRES_USER",        "valueFrom": "arn:aws:ssm:us-west-2:438336773404:parameter/papyrus/prd/POSTGRES_USER" },
        { "name": "POSTGRES_PASSWORD",        "valueFrom": "arn:aws:ssm:us-west-2:438336773404:parameter/papyrus/prd/POSTGRES_PASSWORD" },
        { "name": "POSTGRES_DB",        "valueFrom": "arn:aws:ssm:us-west-2:438336773404:parameter/papyrus/prd/POSTGRES_DB" },
        { "name": "FLASK_SECRET_KEY",        "valueFrom": "arn:aws:ssm:us-west-2:438336773404:parameter/papyrus/prd/FLASK_SECRET_KEY" }
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

※awslogs の設定はそのまま（※ロググループは先に作っておくこと）

#### 3.3 ECSクラスター内にサービスを作成する。
1. コンソール派：クラスター→「サービスの作成」→

- 起動タイプ: Fargate
- タスク定義: さきほど決めたタスク定義名
- 望ましいタスク数: 0（これで課金止まる）
- ネットワーク: Publicサブネット＋Public IP有効、SGは自分のIPからだけ許可

Public IPは以下の方法で調べる。
Linux/Mac: `curl -s https://checkip.amazonaws.com`コマンド
Windows Powershwll: `(Invoke-WebRequest ifconfig.me/ip).Content`
ブラウザ: `https://whatismyipaddress.com/`


:::default CLIでサービス作成する場合のサンプル
```bash:sample.bash
aws ecs create-service \
  --cluster <CLUSTER_NAME> \
  --service-name <SERVICE_NAME> \
  --task-definition <TASK_FAMILY> \
  --desired-count 0 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-xxx,subnet-yyy],securityGroups=[sg-zzz],assignPublicIp=ENABLED}"
```
:::

### 4. Auth0の対応

Auth0の設定、及び`.env`ファイルの`AUTH0_CALLBACK_URL`を変更しておく。

|項目名|値|
|---|---|
|許可するCallbackURL|`http://<PublicIP>:5000/callback`|
|許可するURL|`http://<PublicIP>:5000/logout`|
|許可するWebオリジン|`http://<PublicIP>:5000`
|`AUTH0_CALLBACK_URL`|`http://<PublicIP>:5000/callback`|

### 5. SSM/Secrets で本番環境変数を登録する

AWS Secret Managerの

```bash
# 非機密
aws ssm put-parameter --region <REGION_ID> --name /papyrus/prd/AUTH0_DOMAIN \
  --type String --value your-tenant.us.auth0.com --overwrite

# 変動するコールバックURL（とりあえず仮。起動後に更新して再デプロイ）
aws ssm put-parameter --region <REGION_ID> --name /papyrus/prd/AUTH0_CALLBACK_URL \
  --type String --value http://0.0.0.0:5000/callback --overwrite

# 機密
aws ssm put-parameter --region <REGION_ID> --name /papyrus/prd/AUTH0_CLIENT_SECRET \
  --type SecureString --value 'xxxx' --overwrite
```


### 6. GitHubでの操作

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

#### ecr-push.ymlの作成

:::default ecr-push.ymlのサンプル
```yml:ecr-push.yml
# .github/workflows/ecr-push.yml
# Build Docker image and push to Amazon ECR on push to main and tags
# AWS secrets are stored in GitHub Secrets, non-sensitive constants in Repository Variables.

name: Build & Push to ECR

on:
  push:
    branches: [ main ]
    tags: [ 'v*.*.*' ]
  workflow_dispatch:

permissions:
  id-token: write   # for OIDC auth to AWS
  contents: read    # to checkout code
  packages: read

concurrency:
  group: ecr-push-${{ github.ref }}
  cancel-in-progress: false

env:
  AWS_REGION: ${{ vars.AWS_REGION }}            # Variables: safe to expose
  ECR_REPOSITORY: ${{ vars.ECR_REPOSITORY }}    # Variables: safe to expose
  IMAGE_NAME: ${{ vars.ECR_REPOSITORY }}

jobs:
  build-and-push:
    name: Docker build & push
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      # --- Auth to AWS via OIDC (no static keys) ---
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_IAM_ROLE_ARN }} # Secrets: sensitive
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Extract metadata (tags, labels)
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}
          tags: |
            type=raw,value=latest,enable={{is_default_branch}}
            type=ref,event=branch
            type=ref,event=tag
            type=sha

      - name: Build and push
        uses: docker/build-push-action@v6
        with:
          context: .
          file: ./Dockerfile
          push: true
          platforms: linux/amd64,linux/arm64
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          build-args: |
            ENV=prod

      - name: Echo image digest
        run: echo "Image digest ${{ steps.meta.outputs.tags }} -> ${{ steps.buildx.outputs.digest }}"

      - name: Export image URI (for downstream jobs)
        id: out
        run: |
          IMAGE_URI="${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ github.sha }}"
          echo "image_uri=$IMAGE_URI" >> $GITHUB_OUTPUT

# --- Optional: Static keys fallback (NOT recommended) ---
#      - name: Configure AWS credentials (static keys)
#        uses: aws-actions/configure-aws-credentials@v4
#        with:
#          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
#          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
#          aws-region: ${{ env.AWS_REGION }}
```
:::

#### ecs-deploy.ymlの作成

:::default ecs-deploy.ymlのサンプル
```yaml:ecs-deploy.yml
name: Deploy to ECS Fargate

on:
  workflow_dispatch:
    inputs:
      image_tag:
        description: ECR image tag to deploy (e.g., latest or a SHA)
        required: true
        default: latest
      desired_count:
        description: Desired task count for the service
        required: true
        default: '1'

permissions:
  id-token: write
  contents: read

env:
  AWS_REGION: ${{ vars.AWS_REGION }}
  ECR_REPOSITORY: ${{ vars.ECR_REPOSITORY }}
  ECS_CLUSTER: ${{ vars.ECS_CLUSTER }}
  ECS_SERVICE: ${{ vars.ECS_SERVICE }}
  CONTAINER_NAME: ${{ vars.CONTAINER_NAME }} # e.g., app
  TASK_FAMILY: ${{ vars.TASK_FAMILY }}

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_IAM_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Ensure CloudWatch Log Group exists
        run: |
          aws logs create-log-group --log-group-name "/ecs/papyrus" --region "${{ env.AWS_REGION }}" 2>/dev/null || true
          aws logs put-retention-policy --log-group-name "/ecs/papyrus" --retention-in-days 1 --region "${{ env.AWS_REGION }}"

      - name: Login to Amazon ECR (for registry URI)
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Render task definition (inject image tag)
        id: render
        uses: aws-actions/amazon-ecs-render-task-definition@v1
        with:
          task-definition: taskdef.json
          container-name: ${{ env.CONTAINER_NAME }}
          image: ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ inputs.image_tag }}

      - name: Deploy to Amazon ECS
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: ${{ steps.render.outputs.task-definition }}
          service: ${{ env.ECS_SERVICE }}
          cluster: ${{ env.ECS_CLUSTER }}
          wait-for-service-stability: true
          desired-count: ${{ inputs.desired_count }}
          force-new-deployment: true
```
:::

#### ecs-scale.ymlの作成

:::default ecs-scale.ymlのサンプル
```yaml:ecs-scale.yaml
name: ECS Scale Service

on:
  workflow_dispatch:
    inputs:
      count:
        description: Desired count (0 to stop)
        required: true
        default: '0'

permissions:
  id-token: write
  contents: read

env:
  AWS_REGION: ${{ vars.AWS_REGION }}
  ECS_CLUSTER: ${{ vars.ECS_CLUSTER }}
  ECS_SERVICE: ${{ vars.ECS_SERVICE }}

jobs:
  scale:
    runs-on: ubuntu-latest
    steps:
      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_IAM_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Update desired count
        run: |
          aws ecs update-service \
            --cluster "$ECS_CLUSTER" \
            --service "$ECS_SERVICE" \
            --desired-count "${{ inputs.count }}"
```
:::

#### ECR pushの実行
![デプロイの様子](https://storage.googleapis.com/zenn-user-upload/390ad72355d2-20250814.png)
GitHubのActionsタブ → ワークフロー “Build & Push to ECR” を開く

Run workflow（workflow_dispatch）で実行

ブランチ：main

入力不要（そのまま走らせる）

成功の目印（ログで確認）

Login Succeeded（ECRログイン）

pushed: <registry>/<repo>:latest

pushed: ...:<branch> or ...:<sha>

最後に Image digest -> sha256:...

ECRコンソール → 対象リポジトリ

Tags に latest と sha が並ぶのを確認

マニフェストが multi-arch（linux/amd64 と linux/arm64）になってると最高

注意: もし denied: requested access to the resource is denied が出たらリポ名ミス。vars.ECR_REPOSITORY と ECR の実名が一致してるか見直し。no basic auth credentials は OIDCロール/信頼ポリシーが怪しい。

#### デプロイ & 節約
Actions → Deploy to ECS Fargate

image_tag=latest

desired_count=1

完了したら タスク詳細 → Public IP:5000 でアクセスして録画

Actions → ECS Scale Service

count=0（すぐ止める。請求ブレーキ）

![デプロイ成功](https://storage.googleapis.com/zenn-user-upload/092a225b93c8-20250814.png)

#### 最終チェック（いま詰まりやすい3点）
タスクは ARM64：pushはmulti-archでやる（もう対応済みって言ったよね、えらい）。

ポート統一：SG 5000/tcp ↔ タスク定義 containerPort: 5000 ↔ アプリが 5000でLISTEN。三点一致。

iam:PassRole：executionRoleArn/taskRoleArn を フルARNでインラインポリシーに入ってること。