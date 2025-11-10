---
title: "ADR: TaskDef は digest 固定"
---

## ECS タスク定義の意思決定

### Context

* 再現性・ロールバック・SBOM/脆弱性

### Decision

* `image@sha256:…` で固定

### Alternatives（`latest` 等の不採用理由）

* latest/tag移動の不確実性／キャッシュ地雷

### Consequences（再現性/ロールバック）

* 成功パターン／失敗防止

### Decision Record

* YYYY-MM-DD, Owner, 適用範囲

### Follow-ups（タグ運用）

* 安全なタグ戦略（`release-*` など）