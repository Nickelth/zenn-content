---
title: "製造現場向け業務効率化SaaSプラットフォーム 運用保守PJ"
---

## 製造現場向け業務効率化SaaS SPA

### 概要

#### ドメイン

製造現場向け業務効率化SaaSプラットフォーム（設備管理・図面管理・不良記録・日報管理・作業報告書 等）

#### 目的

生産管理・各種帳票作成のワークフローのデジタル化

#### 構成

TypeScript ＋ Vue.js ＋ Node.js ＋ Nest.js ＋ Rest API ＋ Prisma ＋ Aurora(PostgreSQL) ＋ ECS Fargate ＋ ECR ＋ Code Pipeline / Code Commit / Code Build ＋ S3 ＋ Docker Desktop

### 案件情報

期間： 2024/08 〜 現在

体制： PM 1名、PL2名、SE 3~4名で構成された4〜7名の受託開発チーム。(時期により変動)

自分の役割： 詳細設計 / 実装 / 運用保守 など

### 担当業務

- フロントエンド／バックエンドの改修（`Vue.js` / `Nest.js` / `TypeScript`）
- Nest.jsの脆弱性対応（v10 → v11）に伴う依存パッケージの更新、および破壊的変更点への対応（DI構成、Decoratorなど）
- CI/CDパイプライン（CodeBuild / CodeDeploy / CodePipeline）の保守・改修
  - `buildspec.yaml`の修正、Dockerビルド最適化、Node.jsの依存関係管理
- 開発/本番DB方向へのリモートポートフォワーディング用の`PowerShell`の改修
- AWS環境での運用・保守対応
  - S3, CloudFrontによるログ監視、静的コンテンツ管理
  - CloudWatch Logsを用いたFargate上のDockerログ監視
  - Amazon Inspectorによるセキュリティチェックおよび修正対応
  - ECRのイメージ管理、CodePipeline連携による自動デプロイの運用
  - 障害時のDB復旧対応（Aurora + CloudFront + EventBridge）
  - タスク定義の更新(`taskdef.json.tpl`)

- その他業務
  - 顧客データの復元・物理削除対応（S3 / PostgreSQL）
  - Auth0を利用した認証処理に関するパフォーマンス改善対応
  - Prisma バージョンアップ（v6 → v7）に伴う不整合エラー解消対応
  - フロントエンドのレスポンシブデザイン含むUI修正（CSS / Vue.js）

### 技術スタック

**言語・FW:** TypeScript, PostgreSQL, Vue.js, PowerShell, CSS

**インフラ:** AWS ECS Fargate, ECR, Code Pipeline / Code Commit / Code Build, Aurora, S3

**ツール類:** VSCode, Docker Desktop, A5:SQL Mk-2, Git, WinMerge, Redmine




### Key ADR（重要な意思決定）2〜3個

### 抄録

SaaS SPAの保守で、**機能追加・脆弱性対応・顧客データ物理削除・RDS激甚時のリストア演習**を、運用目線の意思決定で整理。

### TL;DR

* 機能追加方針、Inspector例外設計、物理削除の安全策、DR演習の要点
* 成果は定性中心（SLO維持、運用負荷低減、誤検知抑制）

### スコープ/非スコープ

* スコープ: フロント実装ポリシー、認証・API連携、セキュリティ対応、DR演習
* 非スコープ: 新規大規模開発

### 使用技術

* TypeScript, Vue, Auth関連, RDS, Inspector

### 構成図（1枚）

* SPA→API→DB、認証トークンの流れ、DR経路

### Decision Log

| テーマ   | 代替案                      | 採用            | 理由             |
| ----- | ------------------------ | ------------- | -------------- |
| 認証連携  | 都度取得/キャッシュ+refresh/SDK委譲 | キャッシュ+refresh | 体感速度と整合性の両立    |
| 脆弱性対応 | 全件対応/例外設計/ツール乗り換え        | 例外設計          | 誤検知疲れ回避とMTTR短縮 |
| 物理削除  | 論理削除/物理削除/保留             | 物理削除          | 法務/監査要件        |
| DR演習  | 机上/定期演習/本番相当             | 定期演習          | 復旧手順の陳腐化防止     |

### 代表タスクと貢献

* 新機能追加のレビュー方針（型安全・E2E範囲）
* Inspector対応の基準票（例外条件）
* 物理削除の二重確認フロー
* RDS復旧演習のRunbook整備

### リスクと緩和

* 認証リトライ嵐 → バックオフと集中管理
* 物理削除の誤操作 → 二重承認とドライラン

### 今後の改善

* E2Eの@critical最小集合、リリース後の外形監視強化

---

## 仕上げミニチェックリスト（全記事共通）

* [ ] 抄録200字内、TL;DR 4行以内
* [ ] 図1枚、表1枚、Decision Log表あり
* [ ] スコープ/非スコープ明記
* [ ] 「採用しなかった理由」を必ず1つ以上
* [ ] 個人情報・機密ゼロ、実務名は匿名化

この型で、再現実験ゼロでも“判断と構成”で押し切れる。時間が足りないなら、図は四角と矢印だけにして出す。文章は短く、決定と理由は濃く。