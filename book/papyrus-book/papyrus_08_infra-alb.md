---
title: "IaC: 一時ALB/TG（infra/20-alb）"
---

## 疎通確認用一時ALB/TG

### 目的

* スモーク専用ALB/TGを**毎回作って必ず破棄**する仕組み

### 前提（Public Subnet /2a,2b,2c）

* AZカバレッジの前提（unused対策）

### 設計の要点

#### 健康チェック `/healthz` (200–399)

* 値の根拠／リトライ思想

#### 変数とtfvars生成（runtime）

* CI側での一時ファイル生成

### 実装ポイント（Terraform）

#### lb / tg / listener

* `target_type="ip"`／`deregistration_delay` 値

### 証跡の取り方

#### plan/apply/destroy ログ

* 出力先とサンプル

### 失敗例と対策（DuplicateListener等）

* 競合・残骸・名前衝突の回避

### セキュリティ注意（必ずdestroy）

* 破棄し損ねの検出とアラート

### 付録（tf抜粋枠）

* `health_check {}` の完全例