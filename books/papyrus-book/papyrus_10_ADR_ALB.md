---
title: "ADR: 一時ALBスモーク vs 恒久ALB"
---

## ALB設定の意思決定

### Context

* コスト／証跡／手離れの要件

### Decision

* 一時ALB採用、destroy徹底

### Alternatives

* 恒久ALB／ローカルcurlのみ／VPC Lattice 等

### Consequences（Pros/Cons）

* Pros/Cons を3点ずつ

### Decision Record（日時/担当/適用範囲）

* 形式：YYYY-MM-DD, Owner, 影響範囲

### Follow-ups（移行条件）

* 恒久化へ移る判断基準（トラフィック閾値等）