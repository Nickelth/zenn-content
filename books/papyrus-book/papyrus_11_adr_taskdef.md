---
title: "ADR: TaskDef は digest 固定"
---

## ECS タスク定義の意思決定

### 背景（Context）

* 再現性・ロールバック・SBOM/脆弱性

### 決定（Decision）

* `image@sha256:…` で固定

### 代替案（Alternatives）

* latest/tag移動の不確実性／キャッシュ地雷

### 影響（Consequences）

* 成功パターン／失敗防止

### 記録（Decision Record）

* YYYY-MM-DD, Owner, 適用範囲

### フォローアップ（Follow-ups）

* 安全なタグ戦略（`release-*` など）