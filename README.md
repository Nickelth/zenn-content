# 🧩 Master Plan：MLS合格 × Zenn構築 × AI力強化

## 🎯 ゴール

- ✅ 2026年4月上旬：AWS Certified Machine Learning – Specialty 合格
- ✅ GitHub + Zennでの技術アウトプット20本（面接で武器にする）
- ✅ MLOps/AI案件への実務対応力を確保
- ✅ Terraform・CI/CD・セキュリティの強化
- ✅ 転職 or 案件増のためのポートフォリオ確立

---

## 📆 フェーズ分け（7月〜翌4月）

| 月  | テーマ | メインタスク |
|-----|--------|--------------|
| 7月 | Pythonリハビリ & WSL構築 | Flask/PDF出力/WSL環境/Zenn投稿 |
| 8月 | AI集中お盆合宿 & Terraform基礎 | PyTorch, CNN, IaC記事, Github Actions |
| 9月 | AI実案件ブリッジ & 保守の言語化 | 自社モデル調査, バッチ→Python記事化 |
| 10月 | Docker & fine-tuning & MLOps設計 | モデルAPI化, GPU対応Docker, Zenn更新 |
| 11月 | Transfer Learning & IaC応用 | ResNet/EfficientNet, CDK検証 |
| 12月 | 運用/監視設計 & セキュリティ導入 | AutoML, Inspector, GuardDuty, MLOps記事 |
| 1月 | パイプライン強化 & CI/CD構成改善 | 再学習構成, Trivy, GitHub Actions連携 |
| 2月 | コスト最適化 & 最終整理 | Spot試算, CloudWatch, 構成見直し |
| 3月 | 模試対策 & Zenn集大成 | MLS対策まとめ、模試→弱点潰し |
| 4月 | 本番試験 & 転職/発信強化 | MLS受験、受験記Zenn、まとめ記事 |

---

## 🔧 進行中タスク（更新して使う）

### ✅ 7月25日（金）夜： ③ 着手

#### 🐳 GitHub Actions + Dockerfile + docker-compose（③）

* Nginxインストール
* docker-compose：`web`（Flask）と`db`（PostgreSQL）と`nginx` or `traefik`（あるなら）
* Dockerfile：production用に最適化（gunicornあたり）
* GitHub Actions：`main` pushで Docker build → push → SSH or remote deploy
  
所要時間：①は2〜3時間書けば形になる。③は4〜6時間泥臭い。慣れてなければ地味にバグる。

---

### ✅ 7月26日（土）フル稼働：② + Zennドラフト②・③・④下書き

#### 🔐 Auth0連携（②）

* 公式サンプルをFlask + Jinjaでやってみる（[Auth0 Flask quickstart](https://auth0.com/docs/quickstart/webapp/flask)）
* `/index`や`/generate_pdf`を`@requires_auth`で保護（開発用環境だけスキップ）


### 📘 Zenn投稿②・③の下書きだけ着手

* タイトル案：

  * 「GitHub ActionsでFlask + PostgreSQL + Dockerを自動デプロイした」
  * 「FlaskアプリをAuth0で守るまで（Zenn版）」

所要時間：②は1〜1.5時間で設定・確認完了見込み。
Zenn②③は見出し案とスクショだけでOK。

### ④Nginxインストール見出し追加
リリース済みのUbuntu+Python+Apache記事にNginxインストールの内容を追記

---

### ✅ 7月27日（日）余裕があればZenn②③④清書・投稿

疲れてたら、もうココで諦めて「後日投稿します」で逃げろ。

---

### ✅ Zenn投稿計画

| タイトル | 優先度 | 状況 | 予定日 |
|----------|--------|------|--------|
| Linux上で動画→GIF変換を最適化した話：Adobeからffmpegへ | ★★★★☆ | 完成 | 7/25 |
| psycopg2やめてSQLAlchemy導入 | ★★★★☆ | 未着手 | 7/25 |
| Gunicorn + Nginx で本番環境構築してFlaskアプリを全世界に公開 | ★★★★☆ | 未着手 | 7/26 |
| GitHub Actions + Dockerfile + docker-composeでデプロイ | ★★★★★ | 仕込み中 | 7/26 |
| PythonアプリとAuth0を連携 | ★★★★★ | 実装中 | 8/2 |
| TerraformでAPI構築 | ★★★★☆ | お盆予定 | 8/2 |
| TerraformでWeb構築 | ★★★★☆ | お盆予定 | 8/9 |
| Nest.js地雷話 | ★★★★★ | ドラフト中 | 8/10 |
| “Pythonできる風”のZenn記事を書いていたら、本当にできるようになった件 | ★☆☆☆☆ | 着手前 | - |

---

#### 中古PC候補 3~6万
- 中古 Ryzen5 16GB SSD 50000円以下 Tiny SFF Ubuntu対応
- 中古 HP EliteDesk 800 G4 core i5 Docker WSL
- 中古 ThinkPad X280 メモリ16GB Docker対応

Linux Mint 21.3 Xfce
Pop!_OS 22.04

---

### 🧠 AI学習進捗トラッカー

| タスク | 状況 | メモ |
|--------|------|------|
| PyTorch初学（MNIST） | 🔲 未着手 | 8月中 |
| CIFAR-10実験 | 🔲 未着手 | お盆予定 |
| ゼロから作るDL（読書） | 🔲 途中 | 第3章まで |
| Docker構成（GPU対応） | 🔲 未着手 | 10月予定 |
| Transfer Learning | 🔲 未着手 | ResNetかEfficientNet予定 |
| ML API化（Flask/FastAPI） | 🔲 未着手 | Docker-compose連携予定 |

---

### 🏗️ MLOps・インフラ・IaC計画

| タスク | 状況 | ツール/備考 |
|--------|------|-------------|
| GitHub Actions構成（buildspec + Docker） | 🔲 着手前 | 8月中旬記事化 |
| CodePipeline構成図 | 🔲 着手前 | draw.io使う予定 |
| SageMaker Pipelines構築 | 🔲 着手前 | 11月目標 |
| 脆弱性チェック（Trivy） | 🔲 未着手 | GitHub Actionsに統合 |
| Spot Instances活用 | 🔲 検証予定 | 2月コスト最適化 |

---

### 🧪 MLS試験対策進捗

| フェーズ | 内容 | 進捗 |
|---------|------|------|
| 基礎習得 | G資格の教本を読む | 🔲 8月予定 |
| 基礎習得 | AWS AIサービス、SageMaker初触り | 🔲 8月予定 |
| 動画視聴 | MLS動画教材 | 🔲 9月予定 |
| 動画視聴 | 出題範囲のAWSサービスをBlackBeltで勉強 | 🔲 9月予定 |
| 模試① | TutorialsDojo, Udemy | 🔲 10月予定 |
| 弱点克服 | ドメインごと整理 | 🔲 11月予定 |
| 模試② | 本番前リハ | 🔲 3月中 |
| 受験本番 | AWS MLS試験 | 🔲 4月予定 |

---

## 💡 メモ：失速したときの対処法

- やる気ない日は：
  - タイトルだけ書いたZenn記事を下書き保存
  - READMEのテンプレだけ用意する
  - GitHubに草を1本植える

- “無理”な日は：
  - **なにもしない日**をカレンダーに「休み」と明示しろ（自己肯定感が死なない）

---

## 🌟 やる意味を忘れたら読むやつ

> これはただの資格勉強ではなく、  
> 「見える実績 × 書けるアウトプット × 転職に効く構成」  
> を同時に積み上げる、**大人のポートフォリオRPG**である。

---

1. MLOps設計の「実績」作り
- MLパイプライン（データ前処理〜学習〜評価〜デプロイ）の自作
- JenkinsやGitHub Actionsと連携した再学習フロー
- SageMaker PipelinesやSageMaker Model Monitorで、運用感出す
  
2. パフォーマンス・コスト最適化経験
- AI×GPUインスタンスのコスト試算
- Spot Instance使ったコスト最適化
- Storage（S3＋Glacier）、ログ分析（CloudWatch＋Athena）で“削減実績”

3. 個人プロジェクト or OSS参加
- 個人で画像認識アプリ（Flask or FastAPIベース）をSageMakerデプロイ
- TrivyやGrypeで脆弱性チェック自動化、GitHubに公開
- OSSでIssue整理、あるいは日本語ドキュメントの改善に貢献（地味だけど見られる）


💡 今から育てるべきスキル（ガチャ素材）
- モデル設計やパラメータチューニング ✖学習済みモデル使うだけ
- PyTorch or TensorFlow：フレームワーク使ってオリジナルモデルをちょっとでも触る。
- MLOps系：SageMaker使わされそうだけど、せめてDockerでローカル推論環境作ると強い。
- IaC（Terraform or CDK）：CodeBuild触れるなら、このへんでDevOps度を盛っておく。
- セキュリティ：AI×クラウドの脆弱性対応は、もはやスキルポイント大量発生ゾーン。
