---
title: "IaC: RDS 薄切り（infra/10-rds）"
---

## RDS構築用 Terraform

### 目的

* 既存RDSの**最小構成だけ**コード化し、差分管理と再現性を担保

### 前提（既存RDS/ParamGroup）

* インポート範囲／未Terraform化項目の扱い

### 設計の要点

#### rds.force_ssl=1

* 動的／適用確認の方法

#### SGポリシー（ECS SG のみ許可）

* ルール定義とCIDR非使用

#### init.sql：DRYRUN→本適用

* ECS Exec＋`BEGIN/ROLLBACK` と `COMMIT` 切替

### 実装ポイント（Terraform）

#### variables / main / outputs

* 重要変数／出力の使い道（例：Endpointは**記事ではマスク**）

### 証跡の取り方

#### ParamGroup差分

* `terraform plan`／`apply`抜粋

#### 再起動タイムスタンプ

* RDSイベント／メトリクスのどちらを見るか

#### psqlログ

* 成功/失敗の最小例

### 失敗例と対策

* 反映待ち・依存順序・タイムアウト

### セキュリティ注意（Secrets分離）

* Secrets Managerのキー名・権限分離

### 付録（tf抜粋枠）

* `resource` と `parameter` 周り