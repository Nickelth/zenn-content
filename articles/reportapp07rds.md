---
title: "【#7】ECSとRDSを接続する"
emoji: "🔁"
type: "tech"
topics: ["rds", "aws", "ecs", "ci/cd" ]
published: false
---

##


### 0. はじめに


### 1. RDS（PostgreSQL）を作る

コンソール派でOK（CLIでも可）。設定は“デモ最小構成”で十分。

* **Engine**: PostgreSQL 16
* **Template**: Free tier / Dev
* **DB instance class**: `db.t4g.micro`
* **Storage**: gp3 20GB（自動拡張OFFでも可）
* **DB instance identifier**: `papyrus-db-prd`
* **Master username**: `papyrus`
* **Master password**: 適当に強いやつ
* **VPC**: いまの ECS と同じ
* **Public access**: **No**（非公開推奨）
* **Security group**: 新規 or 既存。**インバウンド5432**に**ECS タスクの SG (`sec-papyrus-prd-ecs-web`) をソース指定**
  → “SG から SG を許可”が鉄板。0.0.0.0/0 は論外（君の面接も論外）。
* **DB name**: `papyrus`（Optional欄にあるやつ。作っとくと楽）
* できたら **エンドポイント** を控える（例: `papyrus-db-prd.xxxxxx.us-west-2.rds.amazonaws.com`）

> もし一時的にローカルから直接 psql したいなら、**一瞬だけ Public access=Yes にして**、SG に**自宅IPだけ**開ける→終わったら閉じる。これでもOK（短時間で）。

### 2. Secrets Manager を RDS 値で更新

君のアプリは Secrets から読む設計なので、**`papyrus/prd/db` の中身を書き換えるだけ**。

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

### 3. `init.sql` を一度だけ実行（安全なやり方を2通り）

#### 方法A-1：ローカルから psql（最短）

* 一時的に RDS の SG で **自分のグローバルIP** から 5432 を許可（5分だけ）。
* 手元で実行：

  ```bash
  psql "host=papyrus-db-prd.xxxxxx.us-west-2.rds.amazonaws.com port=5432 dbname=papyrus user=papyrus sslmode=require" -f init.sql
  ```
* 終わったら **SG を元に戻す**（ECS SG のみ許可）。

#### 方法A-2：ECSの“一回だけ”タスクで psql（閉域のまま）

RDSは非公開のまま、**Fargate 上で `postgres:16-alpine` を1回だけ走らせて流す**。

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

> 秘密をCLIに直書きしたくないなら、上のコンテナ `environment` を削って、`secrets` フィールドで **Secrets Manager のキー**から注入でもOK。
> 今回は「速さ優先」で見せてます。終わったらこのタスク定義は**削除**してよし。

### 4. アプリを再デプロイ → 動作確認

* **サービスを Force new deployment**（Desired=1 で）
* CloudWatch Logs に `db.host=<RDSエンドポイント>` みたいなデバッグを一行出しとくと秒で判断できる
* `curl http://<PublicIP>:5000/index`（面接用の“動いてる風”動画を撮るならここ）

### 5. よくあるハマり所

* **RDS SG** が **ECS タスク SG** を許可していない
  → `could not translate host name` じゃなくても結局つながらない。**必ず SG→SG 許可**。
* **Secrets を更新したのにアプリが古いまま**
  → **新しいタスク**を起動しないと反映されない。Force new deployment しなさい。
* **`search_path` 問題**
  クエリで `papyrus_schema.products` と**スキーマ付き**で書いてないなら、
  `ALTER ROLE papyrus SET search_path TO papyrus_schema, public;` を `init.sql` 末尾に足すと幸せ（任意）。

### 6. 片付け（課金＆衛生）

* 本番デモが終わったら **Desired=0**、RDS は **停止できない**ので**スナップショット取って削除**が最安。
* Secrets/SSM は残しても微課金。気になるなら **タグ `project=portfolio`** でまとめて削除候補に。
* さっきの**一時タスク定義**（`papyrus-db-init`）は削除でOK。

---

### 7. まとめ（君向け翻訳）

* **ECS で “db” は勝手に出てこない**。RDS を作る or 何かを立てる必要がある。
* 今日は **RDS 作る → Secret を RDS 値に更新 → `init.sql` を一度だけ流す → 再デプロイ**。

<!--
### 8. 次の記事

@[card](https://zenn.dev/nickelth/articles/reportapp08cicd)
-->