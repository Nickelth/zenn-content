---
title: "CI: ECR Push（ビルド→ECR）"
---

## デプロイ時ECRイメージ自動更新

### 目的

- 何を自動化し、何を保証するか：＜再現性・digest固定・脆弱性配慮＞

### 前提（リポ/Secrets/権限）

- 必要なIAM権限：
    - GitHub OIDC 実行ロール
    - 目的: CI/CD（ECR push, ECS deploy, 一時ALB/TG, CloudWatch/CloudTrail読取）
    - 主要権限:
        - "ecr:BatchCheckLayerAvailability",
        - "ecr:InitiateLayerUpload",
        - "ecr:UploadLayerPart",
        - "ecr:CompleteLayerUpload",
        - "ecr:PutImage",
        - "ecr:BatchGetImage",
        - "ecr:DescribeRepositories",
        - "ecr:ListImages"

### ワークフロー設計

#### トリガー

- `on.push.branches: [master]`
  リリース運用を “master = 可動” に固定

- `on.push.tags: [ 'v*.*.*' ]`
  タグ push でも同じ処理。タグ名は CHANGELOG のアンカーにも使用。

#### キャッシュ/タグ設計（digest固定）

- `latest` 不採用の理由
  - ロールバック不可能、**再現性ゼロ**のため廃止。

- 参照は **digest 固定**に寄せる
  - 例: タスク定義や Helm values で `image: <acct>.dkr.ecr.../repo@sha256:xxxxx`
  - 取得元はワークフローの出力（下で説明）か、`aws ecr batch-get-image --image-ids imageTag=<tag>` の `imageId.imageDigest`。

### 実装ポイント

#### Docker build/push ステップ（要点）

- **マルチステージ**でビルドとランタイムを分離。`RUN apt-get ...` はビルド段のみ。
- **.dockerignore** は`docs/`, `.git/`, `node_modules` 等を必ず除外。
- **build args** は非機密のバージョン番号だけ。秘密は **BuildKit secrets** を使う（`RUN --mount=type=secret`）。
- **出力**は `docker/build-push-action` の `outputs.digest` を使う。

#### 成果物・メタデータ（imageDigest を残す）

- 例: ECR に push 後、以下を evidence に書き出す。
  - `repository`, `imageTag`, **`imageDigest`**, `build args`, `base image@digest`

- 取得例（どれか一つやれば十分）
  - `steps.build.outputs.digest`
  - `aws ecr batch-get-image --repository-name ... --image-ids imageTag=$TAG --query images[0].imageId.imageDigest`
  - `docker inspect --format='{{index .RepoDigests 0}}' <image:tag>`

### 失敗例と対策（テンプレ）

- **OIDCロールに Assume 権限なし**

  - 症状: `Not authorized to perform sts:AssumeRoleWithWebIdentity`
  - 対策: 対象ロールの**信頼ポリシー**で

    - `Principal.Federated` に `arn:aws:iam::<acct>:oidc-provider/token.actions.githubusercontent.com`
    - `Condition.StringEquals` に
      `token.actions.githubusercontent.com:aud = sts.amazonaws.com`
    - `Condition.StringLike` に
      `token.actions.githubusercontent.com:sub = repo:<OWNER>/<REPO>:ref:refs/heads/master`（ブランチ/タグに合わせて増やす）

### セキュリティ注意

- **秘密情報は ARG に渡さない**
  - `ARG` はイメージ履歴に残る。BuildKit の `--secret` を使う。`ENV` も禁止。

- **Base image の更新方針**
  - 月次で `base@digest` を更新し、ECR スキャン結果を evidence に残す。
  - Dependabot だけに丸投げしない。Digest 固定にして意図的に上げる。