---
title: "CI: ECS Scale（手動 DesiredCount 制御）"
---

## ECSタスク数の手動スケーリング（停止専用）

### 目的

- 検証用 ECS サービスを **「使うときだけ起動する」ための停止ワークフロー**
- 起動は別ワークフロー（`ecs-deploy.yml`）で `desired-count=1` にして新イメージをデプロイ  
  停止は本ワークフロー（`ecs-scale.yml`）で `desired-count=0` にしてタスクを全停止
- スケジューラではなく、**運用担当が明示的に「起動」「停止」を実行する運用**とし、  
  使っていない時間帯の Fargate 実行コストを削減する


### 前提（停止/再開のポリシー）

- 対象はあくまで **ECS サービス単位**（`ECS_CLUSTER` / `ECS_SERVICE` は事前に定義済み）
- どの環境を止めてよいか、どの時間帯に誰が実行してよいかは運用ルールで決定
  - 例：本番は対象外、検証クラスタのみ / 平日夜間と休日だけ停止など

- CI そのものは **ポリシーを強制しない**（誤操作は IAM / 運用ルールで防ぐ前提）

### ワークフロー設計

#### トリガーと入力

- `workflow_dispatch` で手動起動
  - `count`: `--desired-count` に渡す値
    - デフォルト `'0'`（= 停止）
    - YAML 上は任意の値を受け取れるが、運用ポリシーとしては **停止専用（= 0 のみ使用）**
- 起動側ワークフロー（`ecs-deploy.yml`）では、常に `desired-count=1` としてタスク定義更新 + デプロイを行う

#### コスト最適化と業務時間

- 例として以下のような運用を想定：
  - 日中：`count=1` で最小構成起動（開発者が触れる）
  - 夜間・休日：`count=0` で停止し、Fargate 実行コストを削減

- 将来的に EventBridge Scheduler での自動化も可能だが、  
  **まずは「明示的に誰かが止めた／動かした」がログに残る形（workflow_dispatch）から始める**。

### 実装ポイント

#### DesiredCount の変更

- 実際に行っている処理はシンプルで、`aws ecs update-service` 一発：

```bash
aws ecs update-service \
--cluster "$ECS_CLUSTER" \
--service "$ECS_SERVICE" \
--desired-count "${{ inputs.count }}"
```

- `update-service` の結果は AWS 側で非同期に反映されるため、
  このワークフローでは **安定化待ちや状態確認は行っていない**。
- 実運用では、必要に応じてコンソール or 別途 `describe-services` を叩いて状態を確認する。

#### 影響範囲（接続断）

- `count=0` にすると対象サービスのタスクはすべて停止するため、
  - ALB のターゲットも 0 になり、新規リクエストは失敗する

- 再起動時（`count>0`）は、アプリの初期化処理（DB 接続・マイグレーションなど）が
  正しく idempotent に動いている前提で設計しておく必要がある。

### 証跡の取り方

- このワークフロー自体は状態ダンプを保存していないため、証跡は以下に依存：
  - GitHub Actions の実行履歴（誰がいつ、どの `count` で実行したか）
  - AWS CloudTrail（`UpdateService` API 呼び出し）
  - ECS サービスイベント（スケールイン / アウトのイベントログ）

- 追加で厳密に残したい場合は、別途 `describe-services` の結果を
  `docs/evidence` に保存するステップを足す余地がある。

### 失敗例と対策

- **クラスター名 / サービス名のミス**
  - `ECS_CLUSTER` / `ECS_SERVICE` が存在しないと `update-service` がエラーで終了
  - Vars / Secrets 側で typo しないようレビュー必須

- **権限不足 / スロットリング**
  - CI 用 IAM ロールに `ecs:UpdateService` が無い場合は即エラー
  - 頻繁にスケール操作する場合は、`ThrottlingException` を避けるため運用頻度も調整

### セキュリティ注意

- CI 用 IAM ロールには、少なくとも以下の権限が必要：
  - `ecs:UpdateService`（対象クラスタ / サービスにスコープを限定）

- 本来は「デプロイ用ロール」「スケール操作用ロール」を分けることで、
  - 誰がサービス構成を変えられるか
  - 誰がタスク数だけ触れるか
    を分離できる設計が望ましい。