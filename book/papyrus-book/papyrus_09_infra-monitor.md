---
title: "IaC: CloudWatch Alarm（infra/30-monitor）"
---

## しきい値監査用 CloudWatch Alarm

### 目的

* しきい値を**意図つき**でコード化し、観測の入口を担保

### 前提（SNS/通知）

* どのTopicを使い、誰が購読するか

### 設計の要点（閾値の根拠）

#### ECS Memory >80%（Avg, 5m×2）

* 80%の根拠／誤検知の扱い

#### ALB 5xx >1（Sum, 5m×2）

* 5xxの意味（ELB vs Target）

#### TargetResponseTime p90 >1.5s

* p90を採用した理由

### 実装ポイント（Terraform）

* `aws_cloudwatch_metric_alarm` の最小例

### 証跡の取り方

* `apply`ログ／メトリクス画面の説明キャプション

### 失敗例と対策

* 権限不足／命名衝突／Namespaceミス

### セキュリティ注意

* SNSトピックの公開範囲

### 付録（tf抜粋枠）

* 3種アラームのひな形
