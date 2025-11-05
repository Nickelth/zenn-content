---
title: "Papyrus Invoice 全体概要"
emoji: "📄"
type: "tech"
topics: ["rds", "postgresql", "ecs", "aws", "terraform", "githubactions", "cloudwatch", "devops", "portfolio"]
published: false
---

@[card](https://github.com/Nickelth/papyrus-invoice)

## はじめに（Purpose）

**PJの目的**: RDSのマスターデータから納品書をPDF出力する最小B2Bワークフローの構築
**評価点**: IaC/観測/証跡/CI
**スコープ内・外**: 含む：ECS/Fargate, RDS, 一時ALBスモーク｜含まない：恒久ALB, 本番SLA

## システム構成（Architecture）

### コンポーネント一覧

* ECS（Fargate）コンテナ：＜役割／イメージ出自／ポート＞
* RDS（PostgreSQL）：＜バージョン／SSL必須／SG方針＞
* 一時ALB/TG（スモーク時のみ）：＜作成・破棄の流れ＞
* CloudWatch Logs/Alarms：＜1行JSON／閾値＞
* Secrets Manager/SSM：＜保存項目／参照箇所＞

### データフロー

* Web→ALB→TG→ECS→RDS の順で、主要リクエスト：`/healthz` `/dbcheck`
* 出力（PDF/ログ/証跡）の流れ：＜どこに保存？＞

### セキュリティ境界

* VPCセグメント／プライベートサブネット
* SGルール（ECS→RDSのみ許可、CIDR解放なし）
* `rds.force_ssl=1`/`PGSSLMODE=require`

## 開発・運用ポリシー

### 完成定義（Definition of Done）

* DoD項目の箇条書き：疎通・最小スキーマ・観測入口・薄切りIaC・証跡・ParamGroup反映・CLI履歴

- ECS→RDS の疎通 OK

- RDSに最小スキーマを適用済み

- CloudWatch Logs に構造化ログを出力（JSON1行）

- IaC 薄切り (RDS/SG/ParameterGroup だけTerraform化。完全Importは後回し)

- 証跡：`psql` 接続ログ、SG設定SS、アプリログに接続成功 

- Parameter Group反映

- CLI履歴の証跡化

### 証跡運用（docs/evidence）

* 何が、いつ、どのワークフローで落ちるか：＜ファイル命名規則＞
* 公開可否（マスク指針）：＜RDSエンドポイントは伏せる等＞

## 依存関係と前提

* AWSリージョン／必要IAMロール／GitHub Actions Secrets/Vars
* 開発PC要件：＜Python, awscli, jq＞

## リポジトリ構成

* 主要ディレクトリの役割：`infra/10-rds`, `infra/20-alb`, `infra/30-monitor`, `docs/evidence`, `papyrus/…`

## 今後の拡張方針

* 恒久ALB化の判断基準
* sslrootcert厳格化
* 本番SLAとスケール戦略

## 付録：用語集

* ECS/Fargate、TaskDef、TargetGroup、Digest固定、観測の入口…を一行定義