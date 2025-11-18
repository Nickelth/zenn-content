---
title: "CI: ECR Push（ビルド→ECR）"
---

## デプロイ時ECRイメージ自動更新

### 目的

- デプロイ時に ECR イメージを自動ビルド & プッシュする

### 前提

- CI 用 IAM ロールは GitHub OIDC 専用に分離。
- ECR push / ECS deploy / 監視読み取りでロールを分割。
- 必要な API のみ許可したカスタムポリシーを付与し、最小権限になるよう設計。

### ワークフロー設計

#### トリガー

- `main` ブランチへの push
- `v*.*.*` タグの push で本番向けイメージ作成

#### キャッシュ/タグ設計（digest固定）

- `latest` 不採用の理由
  - ロールバック不可能、**再現性ゼロ**のため廃止。

- 参照は **digest 固定**に寄せる
  - 例: タスク定義や Helm values で `image: <acct>.dkr.ecr.../repo@sha256:xxxxx`
  - 取得元はワークフローの出力（下で説明）か、`aws ecr batch-get-image --image-ids imageTag=<tag>` の `imageId.imageDigest`。

### 実装ポイント

#### Docker build/push ステップ

- **マルチステージ**でビルドとランタイムを分離。`RUN apt-get ...` はビルド段のみ。
- **.dockerignore** は`docs/`, `.git/`, `node_modules` 等を必ず除外。
- **build args** は非機密のバージョン番号だけ。秘密は **BuildKit secrets** を使う（`RUN --mount=type=secret`）。
- **出力**は `docker/build-push-action` の `outputs.digest` を使う。

### 失敗例と対策

- **OIDCロールに Assume 権限なし**

  - 症状: `Not authorized to perform sts:AssumeRoleWithWebIdentity`
  - 対策: 対象ロールの**信頼ポリシー**で

    - `Principal.Federated` に `arn:aws:iam::<acct>:oidc-provider/token.actions.githubusercontent.com`
    - `Condition.StringEquals` に
      `token.actions.githubusercontent.com:aud = sts.amazonaws.com`
    - `Condition.StringLike` に
      `token.actions.githubusercontent.com:sub = repo:<OWNER>/<REPO>:ref:refs/heads/master`（ブランチ/タグに合わせて増やす）
    - CIジョブ内でOIDCの`sub`/`aud`をログ確認

### セキュリティ注意

- **秘密情報は ARG に渡さない**
  - `ARG` はイメージ履歴に残る。BuildKit の `--secret` を使う。`ENV` も禁止。

- **Base image の更新方針**
  - 月次で `base@digest` を更新し、ECR スキャン結果を evidence に残す。
  - Dependabot だけに丸投げしない。Digest 固定にして意図的に上げる。