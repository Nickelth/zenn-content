---
title: "CI: ALB Smoke（疎通検証・一時ALB/TG）"
---

## ALB/TG/SGの手動ヘルスチェック

### 目的

* `/healthz` `/dbcheck` の**外形監視**を一時ALBで再現可能に

### 前提（VPC/サブネット/SG）

* 使用VPC／公開サブネット（2a/2b/2c）／ALB SG／ECSタスクSG

### 設計の要点

#### 一時ALB/TG & destroy徹底

* ランタイムtfvarsと破棄の流れ

#### Listener冪等更新

* DuplicateListener耐性と分岐

#### AZミスマッチ検知

* `unused` 回避のロジック（ALB有効AZ vs タスクAZ）

#### 自前 wait healthy

* `describe-target-health` ループ条件・タイムアウト

#### CloudWatch 1行JSON収集

* フィルタ式・リトライ・出力ファイル

### 実行手順（固定化ステップ）

* Preflight → Drift検知 → Terraform apply → SG許可 → Register → Wait → Curl → CloudWatch収集 → Destroy

### 証跡の取り方（docs/evidence）

* 期待するファイル名一覧と中身の例

### 失敗例と対策

* `unused`／SG重複／Listener未解決／権限不足

### セキュリティ注意（RDSエンドポイントのマスク）

* マスク方法とCI自動化の案内

### 付録（YAML抜粋枠）

* 主要`run:`ブロックの貼付スペース
