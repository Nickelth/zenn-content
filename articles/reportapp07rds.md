---
title: "【#7】ECSとRDSを接続する"
emoji: "🔁"
type: "tech"
topics: ["rds", "aws", "ecs"]
published: false
---

> RDS高すんぎ

## コスト削減しつつ将来的なMulti-AZへの移行も容易にするRDS構築

### 0. はじめに

※特に断りがない限り、リージョンは`us-west-2`とする。

RDSをAWS構築に組み込み、より実践的なアーキテクチャ構築・運用に近づける。
まずはSingle-AZで作成しコツとカットに努める。
GitHub Actionsで起動/停止をCI/CDに組み込む（次章）。

### 1. 要件定義

- 動かしたい時だけ動かす従属課金運用に合わせる。
- サンドボックス・Single-AZで作成する。
- 将来的なMulti-AZ移行に備え、**同一VPC内で異なるAZのサブネットを2つ以上**入れておく
- Multi-AZ移行は、インスタンスの昇格で対応する。
- その他、課金要素はなるべく使用しない。

### 2. RDS作成と運用

#### 2.1 RDS（PostgreSQL）を作る

「Aurora and RDS」を開き、ダッシュボードで「データベースを作成する」をクリック
![データベースを作成する](https://storage.googleapis.com/zenn-user-upload/3fd582f53ea7-20250824.png)

識別子: papyrus-db-sbx
ユーザー名: papyrus
DB名: papyrus_db

|項目|設定値|
|---|---|
|テンプレート|サンドボックス|
|デプロイ| 1インスタンス (Single-AZ)|
|識別子/ユーザー名|任意名|
|認証情報管理|Secrets Manager（セルフより楽・安全）|
|インスタンスクラス|db.t4g.micro|
|ストレージ|gp3 / 20 GiB（追加IOPS/スループットは無指定）|
|EC2コンピューティングリソース|接続しない|
|パブリックアクセス|なし|
|VPCセキュリティグループ|新規作成（インバウンド5432 = ECSタスクのSGのみを許可）|
|データベース認証|パスワード認証|
|Performance Insights|無効|
|拡張モニタリング|無効|
|ログエクスポート|無効|
|DevOps Guru|無効|
|自動バックアップ|有効 / 保持 1–3日 / ウィンドウ深夜帯|
|スナップショットにタグコピー|任意（有効推奨）|
|暗号化| 有効（aws/rds デフォルト）|
|マイナー自動アップグレード|有効|
|メンテナンスウィンドウ|深夜帯|
|削除保護| 無効（検証なので消しやすく）|


> もし一時的にローカルから直接 psql したいなら、**一瞬だけ Public access=Yes にして**、SG に**自宅IPだけ**開ける→終わったら閉じる。これでもOK（短時間で）。

#### 2.2 RDS作成後に停止→再起動する手順



### 3. Secrets Manager を RDS 値で更新

`papyrus/prd/db` の中身を書き換える

```bash
aws secretsmanager put-secret-value \
  --region us-west-2 \
  --secret-id papyrus/prd/db \
  --secret-string '{
    "host":"papyrus-db-prd.xxxxxx.us-west-2.rds.amazonaws.com",
    "port":5432,
    "username":"papyrus",
    "password":"<RDSで設定したPW>",
    "database":"papyrus"
  }'
```

> 既に `DB_SECRET_ID=papyrus/prd/db` はタスクに入ってるから、**アプリ側のコード変更は不要**。

### 4. `init.sql` を一度だけ実行（安全なやり方を2通り）

#### 方法A-1：ローカルから psql（最短）

* 一時的に RDS の SG で **自分のグローバルIP** から 5432 を許可（5分だけ）。
* 手元で実行：

  ```bash
  psql "host=papyrus-db-prd.xxxxxx.us-west-2.rds.amazonaws.com port=5432 dbname=papyrus user=papyrus sslmode=require" -f init.sql
  ```
* 終わったら **SG を元に戻す**（ECS SG のみ許可）。

#### 方法A-2：ECSの“一回だけ”タスクで psql（閉域のまま）

RDSは非公開のまま、**Fargate 上で `postgres:16-alpine` を1回だけ走らせて流す**。

:::detail CLIでinit.sqlを実行する
```bash
# ① 一時タスク定義を登録（postgresクライアントでSQLを流す）
cat > db-init-task.json <<'JSON'
{
  "family": "papyrus-db-init",
  "requiresCompatibilities": ["FARGATE"],
  "networkMode": "awsvpc",
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::438336773404:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::438336773404:role/papyrusTaskRole",
  "containerDefinitions": [
    {
      "name": "dbinit",
      "image": "public.ecr.aws/docker/library/postgres:16-alpine",
      "essential": true,
      "environment": [
        {"name":"DB_HOST","value":"papyrus-db-prd.xxxxxx.us-west-2.rds.amazonaws.com"},
        {"name":"DB_PORT","value":"5432"},
        {"name":"DB_NAME","value":"papyrus"},
        {"name":"DB_USER","value":"papyrus"},
        {"name":"DB_PASS","value":"<RDSのPW>"}
      ],
      "command": ["sh","-lc",
        "cat <<'SQL' >/tmp/init.sql && \
         PGPASSWORD=\"$DB_PASS\" psql -h \"$DB_HOST\" -p \"$DB_PORT\" -U \"$DB_USER\" -d \"$DB_NAME\" -v ON_ERROR_STOP=1 -f /tmp/init.sql

-- init.sqlのソースコードを記述

--init.sqlここまで
SQL"
      ],
      "logConfiguration": {
        "logDriver":"awslogs",
        "options":{
          "awslogs-region":"us-west-2",
          "awslogs-group":"/ecs/papyrus",
          "awslogs-stream-prefix":"dbinit"
        }
      }
    }
  ],
  "runtimePlatform": { "cpuArchitecture":"ARM64", "operatingSystemFamily":"LINUX" }
}
JSON

aws ecs register-task-definition --cli-input-json file://db-init-task.json

# ② いまのVPCで、ECSタスクSGを付けて起動（RDS SG がこのSGを許可していること）
aws ecs run-task \
  --cluster papyrus-ecs-prd \
  --launch-type FARGATE \
  --task-definition papyrus-db-init \
  --count 1 \
  --network-configuration "awsvpcConfiguration={subnets=[<SUBNET1>,<SUBNET2>],securityGroups=[<ECS_TASK_SG>],assignPublicIp=DISABLED}"
```
:::

> 秘密をCLIに直書きしたくないなら、上のコンテナ `environment` を削って、`secrets` フィールドで **Secrets Manager のキー**から注入でもOK。
> 今回は「速さ優先」で見せてます。終わったらこのタスク定義は**削除**してよし。

### 5. アプリを再デプロイ → 動作確認

* **サービスを Force new deployment**（Desired=1 で）
* CloudWatch Logs に `db.host=<RDSエンドポイント>` みたいなデバッグを一行出しとくと秒で判断できる
* `curl http://<PublicIP>:5000/index`（面接用の“動いてる風”動画を撮るならここ）

### 6. よくあるハマり所

* **RDS SG** が **ECS タスク SG** を許可していない
  → `could not translate host name` じゃなくても結局つながらない。**必ず SG→SG 許可**。
* **Secrets を更新したのにアプリが古いまま**
  → **新しいタスク**を起動しないと反映されない。Force new deployment しなさい。
* **`search_path` 問題**
  クエリで `papyrus_schema.products` と**スキーマ付き**で書いてないなら、
  `ALTER ROLE papyrus SET search_path TO papyrus_schema, public;` を `init.sql` 末尾に足すと幸せ（任意）。

### 7. 片付け（課金＆衛生）

* 本番デモが終わったら **Desired=0**、RDS は **停止できない**ので**スナップショット取って削除**が最安。
* Secrets/SSM は残しても微課金。気になるなら **タグ `project=portfolio`** でまとめて削除候補に。
* さっきの**一時タスク定義**（`papyrus-db-init`）は削除でOK。

---

### 8. おわりに

* **ECS で “db” は勝手に出てこない**。RDS を作る or 何かを立てる必要がある。
* 今日は **RDS 作る → Secret を RDS 値に更新 → `init.sql` を一度だけ流す → 再デプロイ**。

<!--
### 8. 次の記事

@[card](https://zenn.dev/nickelth/articles/reportapp08cicd)
-->