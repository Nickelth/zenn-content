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

### ✅ 7月23日（水）までにやっとけ（軽作業パート）

* [x] GitHubリポジトリ整備（README、.env.sample、必要なSQL初期化スクリプト）
* [ ] Zenn用アカウント整備・下書き枠の準備
* [x] docker-compose環境にする前に、現ローカル構成の依存確認しておく
* [x] Dockerfileのベース案準備（requirements.txt・.env連携どうするか考えとく）

所要時間：1〜2hくらい（最悪、平日の夜に頑張れ）

---

### ✅ 7月25日（金）夜：① + ② 着手

#### 🍶 Zenn投稿①（Java案件の供養＋Flaskアプリの構成メモ）

* タイトル：*「古の納品書アプリをFlaskで再構築した話」*
* 見出し：Javaあるある → Flaskで作り直した構成 → 自動補完・PDF出力の実装 → デモGIFかスクショ
* ぶっちゃけここ、**気合い入れすぎず適度に供養感**出すと読まれる

#### 🔐 Auth0連携（②）

* 公式サンプルをFlask + Jinjaでやってみる（[Auth0 Flask quickstart](https://auth0.com/docs/quickstart/webapp/flask)）
* `/index`や`/generate_pdf`を`@requires_auth`で保護（開発用環境だけスキップ）

所要時間：①は2〜3時間書けば形になる。②は1〜1.5時間で設定・確認完了見込み。

---

### ✅ 7月26日（土）フル稼働：③ + Zennドラフト②・③下書き

#### 🐳 GitHub Actions + Dockerfile + docker-compose（③）

* docker-compose：`web`（Flask）と`db`（PostgreSQL）と`nginx` or `traefik`（あるなら）
* Dockerfile：production用に最適化（gunicornあたり）
* GitHub Actions：`main` pushで Docker build → push → SSH or remote deploy

### 📘 Zenn投稿②・③の下書きだけ着手

* タイトル案：

  * 「FlaskアプリをAuth0で守るまで（Zenn版）」
  * 「GitHub ActionsでFlask + PostgreSQL + Dockerを自動デプロイした」

所要時間：③は4〜6時間泥臭い。慣れてなければ地味にバグる。
Zenn②③は見出し案とスクショだけでOK。

---

### ✅ 7月27日（日）余裕があればZenn②③清書・投稿

疲れてたら、もうココで諦めて「後日投稿します」で逃げろ。

---

### ✅ Zenn投稿計画

| タイトル | 優先度 | 状況 | 予定日 |
|----------|--------|------|--------|
| Javaレガシー帳票出力アプリをPythonにリメイクしてみた | ★★★★☆ | 完了 | 7/21 |
| PythonアプリとAuth0を連携 | ★★★★★ | 実装中 | 7/26 |
| GitHub Actions + Dockerfile + docker-composeでデプロイ | ★★★★★ | 仕込み中 | 7/27 |
| TerraformでAPI構築 | ★★★★☆ | お盆予定 | 8/2 |
| TerraformでWeb構築 | ★★★★☆ | お盆予定 | 8/9 |
| Nest.js地雷話 | ★★★★★ | ドラフト中 | 8/10 |
| “Pythonできる風”のZenn記事を書いていたら、本当にできるようになった件 | ★☆☆☆☆ | 着手前 | - |

---

#### Python帳票出力アプリ
- このツールは帳票出力に特化していて、実運用では社内PC使用前提です。
- モバイル対応は必要性を感じた段階で着手できるよう、Bootstrapベースにしています。
-  「目的：帳票出力精度＋自動補完＋DB連携」
-  バックエンド・インフラ・AI職志望
-  "簡易的な社内ツール"というポジションづけ

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
