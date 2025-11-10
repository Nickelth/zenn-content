---
title: "CI: ECR Push（ビルド→ECR）"
---

## デプロイ時ECRイメージ自動更新

### 目的

* 何を自動化し、何を保証するか：＜再現性・digest固定・脆弱性配慮＞

### 前提（リポ/Secrets/権限）

* 必要なGitHub Secrets/Vars：＜AWS_IAM_ROLE_ARN, AWS_REGION, ECR_REPO＞
* IAM権限：＜ecr:BatchCheckLayerAvailability, ecr:PutImage…＞

### ワークフロー設計

#### トリガー

* どのブランチ・タグで走るか

#### キャッシュ/タグ設計（digest固定）

* `latest` 非採用の理由
* `image@sha256:…` をどこで参照するか

### 実装ポイント

#### Docker build/push ステップ

* build args／マルチステージ／不要ファイル除外

#### 成果物・メタデータ

* `imageDigest` を出力に残す方法

### 証跡の取り方

* `docs/evidence/<ts>_ecr_push.log` の想定内容
* 失敗時のエラー例と原因切り分け

### 失敗例と対策

* 認証失敗／レイヤーキャッシュ不整合／プライベート依存

### セキュリティ注意

* ビルド時の秘密情報扱い
* Base imageの更新方針

### 付録（YAML抜粋枠）

* 必要最小の`jobs:`抜粋を貼るスペース