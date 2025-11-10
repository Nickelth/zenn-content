---
title: "CI: ECS Deploy（TaskDef登録→desire=1）"
---

## ECSタスク数の手動制御

### 目的

* 新イメージへの切替を**無停止**で行い、ロールバック容易性を担保

### 前提（クラスター/サービス）

* `ECS_CLUSTER`/`ECS_SERVICE` の設定、DesiredCount

### ワークフロー設計

#### タスク定義の更新戦略（digest）

* どこでイメージダイジェストを差し込むか

#### force-new-deployment

* 使う理由と挙動（旧タスク終端条件）

### 実装ポイント

#### 失敗時ロールバック

* 前リビジョンへの戻し手順（CLI/コンソール）

#### バージョニング

* タスク定義リビジョン管理の方針

### 証跡の取り方

* `describe-services`／デプロイイベント抜粋を evidence に保存

### 失敗例と対策

* タスク起動失敗（Secrets/Env欠落、SG誤り）

### セキュリティ注意

* Task role／execution role の最小権限

### 付録（YAML抜粋枠）

* `register-task-definition` と `update-service` 部分