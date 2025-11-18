---
title: "ADR: 一時ALBスモーク vs 恒久ALB"
---

## ALB設定の意思決定

### 背景（Context）

* コスト／証跡／手離れの要件

### 決定（Decision）

* 一時ALB採用、destroy徹底

### 代替案（Alternatives）

* 恒久ALB／ローカルcurlのみ／VPC Lattice 等

### 影響（Consequences）

* Pros/Cons を3点ずつ

### 記録（Decision Record）

* 形式：YYYY-MM-DD, Owner, 影響範囲

### フォローアップ（Follow-ups）

* 恒久化へ移る判断基準（トラフィック閾値等）