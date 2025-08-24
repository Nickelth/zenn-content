---
title: "【#4 4/4】GitHubActionsとECS/Fargateを連携、CI/CD環境を構築する"
emoji: "🔁"
type: "tech"
topics: ["ecs", "aws", "githubactions", "ci/cd" ]
published: false
---

## GitHub Actions編（OIDC → Build/Push → Deploy）

### 0. はじめに

1. **GitHub Actions設定**（ECR push + ECS更新までワンストップ）

   * OIDCロール作成（信頼ポリシーに`sub`/`aud`）
   * 権限ポリシーにECRとECS更新（`ecs:RegisterTaskDefinition`、`ecs:UpdateService`、`iam:PassRole`）追加
   * `ecr-push.yml`に続いて、`ecs-deploy.yml`も作る
   * ワークフローが動くと：

     1. Docker Build
     2. ECR push
     3. ECSタスク定義更新
     4. サービス反映

2. **テスト & 証拠動画**

   * `workflow_dispatch`で実行して、ECSのタスクが自動で更新される様子を画面収録
   * 「コードpush → 本番更新」の流れを丸ごと見せられる


> Q. 開発環境で使用していた.envの環境変数をSSM/Secretsに登録しておけば、GithubのVariablesへの登録は不要になりますか？

A. なりません。
SSM/Secrets は「実行時のアプリ用」、GitHub Variables/Secrets は「CI/CD が回るための設定 と ビルド時に必要なもの」。用途が違うので、どちらか片方だけで済むわけではない。欲張りセットってやつ。

- ビルド時に必要？ → はい → GitHub Secrets
- 実行時だけで良い？ → はい → SSM/Secrets
- CI/CD のメタ情報？（リージョン/リポジトリ名/サービス名） → GitHub Variables

### 1. OIDC設定（IdP/Role/信頼ポリシー）



### 2. GitHub Variables/Secrets整備

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

|env_variable|value|
|---|---|
|`AWS_REGION`|`us-west-2`|
|`CONTAINER_NAME`|`app`|
|`ECR_REPOSITORY`|例: papyrus|
|`ECS_CLUSTER`|ECSクラスター名|
|`ECS_SERVICE`|ECSサービス名|
|`TASK_FAMILY`|タスク名

### 3. 実行用yamlの整備

#### ecr-push.yml

:::details ecr-push.ymlのサンプル
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

#### ecs-deploy.yml

:::details ecs-deploy.ymlのサンプル
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

      # log group is pre-created; streams are created by ecsTaskExecutionRole

      - name: Who am I
        run: aws sts get-caller-identity

      - name: Fetch current task definition from ECS
        run: |
          aws ecs describe-task-definition --task-definition "${{ env.TASK_FAMILY }}" \
            --query taskDefinition > taskdef.json
          jq -r '.containerDefinitions[].name' taskdef.json  # デバッグ：存在確認

      - run: ls -la; test -f taskdef.json && echo "found taskdef.json" || echo "missing"

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

#### ecs-scale.yml

:::details ecs-scale.ymlのサンプル
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

### 4. ECR pushの実行

![pushの様子](https://storage.googleapis.com/zenn-user-upload/390ad72355d2-20250814.png)
*mainブランチにpushすると自動実行する。*

![push成功](https://storage.googleapis.com/zenn-user-upload/092a225b93c8-20250814.png)
*push成功*



### 5. デプロイ & 節約の実行
Actions → Deploy to ECS Fargate

image_tag=latest

desired_count=1

完了したら タスク詳細 → Public IP:5000 でアクセスして録画

Actions → ECS Scale Service

count=0（すぐ止める。請求ブレーキ）

### 6. ロールバック & 運用Tips






### 7. 関連記事

#### AWS 関連

@[card](https://zenn.dev/nickelth/articles/reportapp07rds)
@[card](https://zenn.dev/nickelth/articles/reportapp06ecs)
@[card](https://zenn.dev/nickelth/articles/reportapp05aws)
@[card](https://zenn.dev/nickelth/articles/reportapp04iam)

#### 開発~本番まで

@[card](https://zenn.dev/nickelth/articles/reportapp03docker)
@[card](https://zenn.dev/nickelth/articles/reportapp02auth0)
@[card](https://zenn.dev/nickelth/articles/reportapp01flask)


