---
title: "CI: Audit Evidence（CloudTrail/Config 24h・別WF）"
---

## CloudTrail/Config の証跡取得用CI

### 目的

* 監査証跡は**必要時のみ**まとめてPR化（スモークと分離）

### 前提（最小権限/IAM）

* `cloudtrail:LookupEvents`/`config:Describe*` 許可
* リージョンとアカウントの一致確認

### 設計の要点

#### スモークと分離する理由

* ノイズ・遅延・機微の観点

#### イベントフィルタ方針

* `EventSource` や `Resources.ResourceName` のマッチ規則

### 実装ポイント

#### CloudTrail/Config 抽出

* 期間・クエリ・フィルタ・出力ファイル名

#### PR 自動化

* ブランチ命名・コミットメッセージ・ラベル

### 証跡の取り方

* `*_cloudtrail_24h_filtered.json`／`_config_*.json` のサンプル

### 失敗例と対策

* AccessDenied／空ファイル時の扱い

### セキュリティ注意（機微削減）

* マスク／フィルタ強化／保持期間

### 付録（YAML抜粋枠）

* ミニマルな`workflow_dispatch`の骨子