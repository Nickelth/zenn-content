---
title: "MLOps Sci-kit Learn 全体概要"
---
## MLOps scikit-learn リポジトリ

@[card](https://github.com/Nickelth/mlops-sklearn-portfolio)

### はじめに（Purpose）

AWS の小さな構成で、表形式の機械学習を「学習 → モデル → API → ECS/ALB」まで一気通貫で動かし、すぐに再現できることを示す。  
CI で学習・評価・配布（S3/ECR）を自動化し、ダイジェスト固定と証跡で、いつでも同じ状態に戻せるようにする。  
1 行ログと最小限のアラーム、手動デプロイとロールバックを備え、少人数でもリモート運用しやすい形に整える。

### システム構成（Architecture）

#### コンポーネント一覧
- **学習**: scikit-learn + Pipeline（成果物は S3 に保存）
- **API**: FastAPI（/health, /schema, /predict, /predict_batch, /reload）
- **実行基盤**: ECS Fargate（ALB 経由で公開、ターゲットは :8000）
- **レジストリ**: ECR（画像はダイジェスト参照を基本とする）
- **監視/ログ**: CloudWatch Logs（1 行 JSON）、CloudWatch Alarms（最小 1 種）

#### データフロー
1. 学習ジョブがモデルと付随メタ情報を **S3** に保存する。  
2. CI がコンテナを **ECR** に発行し、ECS サービスを更新する。  
3. **ALB** がリクエストを受け、**Target Group** 経由で **ECS タスク** に転送する。  
4. API は推論と同時に **CloudWatch Logs** に構造化ログを出力する。

#### セキュリティ境界
- **ALB SG** は 80/443 を受け付け、**Tasks SG** に対しアプリ用ポートのみを許可する。  
- **S3** はバージョニングとサーバーサイド暗号化（SSE-S3）を有効化し、HTTPS 強制を行う。  
- CI の権限は OIDC + AssumeRole を前提に最小化する。

### 開発・運用ポリシー

#### 完成定義（Definition of Done）
- ALB 経由 `/healthz` が常時 200（ターゲットグループ Healthy）。  
- 手動デプロイを再現でき、直前リビジョンへロールバックできる。  
- CloudWatch Logs に 1 行 JSON の構造化ログが出る。  
- Terraform は **薄切り**で最小一式（VPC/ALB/TG/ECS/ECR）を再現し、Import の完全一致は追わない。  
- `docs/evidence/` に curl 結果、ALB/TG ヘルス SS、ECS イベント、`terraform plan` 抜粋を残す。  
- 失敗からのロールバック実演を記録する。  
- アラームは最低 1 件を具体化する（例: 5xx 率、TargetResponseTime、タスク異常終了）。  
- CLI 履歴は `script` か `bash -x` とし、CloudTrail/Config の照合を添える。

#### 証跡運用（docs/evidence）
- 取得物は日付入りファイル名で管理し、README からリンクする。  
- 代表例: `*_healthz_200.txt`, `*_tg_healthy.png`, `*_ecs_rollback_demo.txt`, `*_tf_plan.txt`。

### 依存関係と前提
- 必要ツール: Python 3.x, Docker, Make, AWS CLI, Terraform。  
- AWS 上は既定の VPC/サブネットを利用し、最小構成で開始する。

### リポジトリ構成
