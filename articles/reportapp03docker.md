---
title: "【#3】Docker+GunicornでFlaskアプリ本番環境構築"
emoji: "🦄"
type: "tech"
topics: ["flask", "docker", "gunicorn"]
published: false
---

## 本番環境の構築

### 0. はじめに

Flaskアプリを公開するために本番環境が必要になった。
当初はVPSで実現する予定だったが、AWSで実施することにした。
価格相応のトラブルが出たことと、実務経験を活かしたいことが理由。

### 1. 選定理由
- **Docker**: 開発環境と本番環境をコンテナで統一し、依存関係や環境差異による不具合を防ぐために採用。
- **Gunicorn**: PythonのWSGIサーバーとして、Flaskアプリを効率よく動作させるために選定。

#### WSGIサーバー（Gunicorn）について

Flaskは開発用サーバーを内蔵しているが、本番運用には適していない。  
そこで登場するのが **WSGI（Web Server Gateway Interface）** というPythonの標準インターフェース。  
WSGIは「Webサーバー（Nginxなど）」と「Pythonアプリケーション（Flaskなど）」の**橋渡し役**を担う。

GunicornはそのWSGI仕様に則った、**高性能かつシンプルなWSGIサーバー**である。  
複数のワーカー（プロセス）を立ち上げ、リクエストを並列処理できるため、スケーラビリティと安定性が向上する。

---

ラーメンに例えると：
- **Docker**：ラーメン屋ごとキッチンカーにして、どこに行っても同じ味を再現できる仕組み
- **Gunicorn**：厨房の責任者。Flaskアプリに仕事を渡して、結果をNginxに返す。
- **Flaskアプリ**：実際にラーメンを作る人

つまり「キッチンカー型ラーメン屋」で、責任者が直接注文を受け、調理担当に指示する構成。
この構成により、Flaskアプリを本番環境で**安全かつ効率的に**動かすことが可能となる。

---

### 2. 動作用ファイルのサンプル
`Flask` + `Docker` + `Gunicorn` + `docker-compose`の構成で動かすためのファイルを公開する。

#### 2-1. Dockerfile

`ENV DEBIAN_FRONTEND=noninteractive`
- 「`apt`が途中で`[Y/n]`を聞いてくる地獄」を回避できる。

`PYTHONDONTWRITEBYTECODE=1`
- Dockerコンテナ内ではほぼ不要な`.pyc`(`__pycache__` ディレクトリにできる)を作らなくなる

`PYTHONUNBUFFERED=1`
- `Python`の標準出力・標準エラーを即時出力する⇒遅延なしでログが見れてデバッグしやすい
```Dockerfile:Dockerfile
# ベースイメージ
FROM python:3.11-slim

# 環境変数で非対話インストール指定
ENV DEBIAN_FRONTEND=noninteractive \
    # ゴミファイル生成防止
    PYTHONDONTWRITEBYTECODE=1 \
    # 遅延なしデバッグ
    PYTHONUNBUFFERED=1\
    LANG=ja_JP.UTF-8 \
    LC_ALL=ja_JP.UTF-8

WORKDIR /app

# WeasyPrint/Pillow/cffi が必要とするネイティブ依存を入れる
# ※ libgdk-pixbuf のパッケージ名は `libgdk-pixbuf-2.0-0`（ハイフンあり）
RUN apt-get update && apt-get install -y --no-install-recommends \
    # コンパイル系
    build-essential \
    python3-dev \
    pkg-config \
    # WeasyPrint ランタイム
    libcairo2 \
    libpango-1.0-0 \
    libpangocairo-1.0-0 \
    libgdk-pixbuf-2.0-0 \
    libffi-dev \
    # Pillow画像系（多くはwheelで入るけど保険）
    libjpeg62-turbo \
    libopenjp2-7 \
    zlib1g \
    # 日本語フォント
    fonts-noto-cjk \
    fonts-dejavu-core \
    # ロケール周り（必要なら）
    locales \
 && sed -i '/ja_JP.UTF-8/s/^# //g' /etc/locale.gen \
 && locale-gen \
 && apt-get clean \
 && rm -rf /var/lib/apt/lists/*

# 依存を先に入れてレイヤキャッシュを効かせる
COPY requirements.txt .
RUN pip install --upgrade pip setuptools wheel \
 && pip install --no-cache-dir -r requirements.txt

# アプリ本体
COPY . .

EXPOSE 5000

CMD ["gunicorn", "-b", "0.0.0.0:5000", "run:app"]
```

#### 2-2. docker-compose.yml

```yaml:docker-compose.yml
services:
  web:
    build: .
    ports:
      - "5000:5000"
    environment:
      # アプリ側で使う追加ENV（必要なら）
      AUTH0_CLIENT_ID: ${AUTH0_CLIENT_ID}
      AUTH0_CLIENT_SECRET: ${AUTH0_CLIENT_SECRET}
      AUTH0_DOMAIN: ${AUTH0_DOMAIN}
      AUTH0_CALLBACK_URL: ${AUTH0_CALLBACK_URL}
      FLASK_SECRET_KEY: ${FLASK_SECRET_KEY}
      DB_HOST: ${DB_HOST:-db}
      DB_PORT: ${DB_PORT:-5432}
      DB_NAME: ${POSTGRES_DB}
      DB_USER: ${POSTGRES_USER}
      DB_PASSWORD: ${POSTGRES_PASSWORD}
      FLASK_ENV: ${FLASK_ENV:-development}
      GUNICORN_WORKERS: ${GUNICORN_WORKERS:-2}
      GUNICORN_THREADS: ${GUNICORN_THREADS:-4}
    depends_on:
      db:
        condition: service_healthy
    volumes:
      - .:/app
    command: >
      gunicorn -b 0.0.0.0:5000 run:app
      -w ${GUNICORN_WORKERS:-2}
      --threads ${GUNICORN_THREADS:-2}
      --timeout ${GUNICORN_TIMEOUT:-60}
      --max-requests ${GUNICORN_MAX_REQUESTS:-200}
      --max-requests-jitter ${GUNICORN_MAX_REQUESTS_JITTER:-50}

  db:
    image: postgres:16
    # ホストでは15432でアクセス、コンテナ内では5432
    ports:
      - "15432:5432"
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 5s
      timeout: 3s
      retries: 10

volumes:
  postgres_data:
```

`Error: address already in use` が出た場合、ホストで既に同じポートが使われている。別ポートに割り当てるか、使用中のプロセスを停止する。

#### 2-3. その他
:::message
`requirements.txt`は初回のみ`venv`環境内で`pip freeze > requirements.txt`で作成する。それ以降は固定にし、Docker内で使いまわす。
ホストのグローバル`Python`から`freeze`するとゴミが混ざるので注意。
:::
```bash
pip freeze > requirements.txt
```
仮想環境`venv`を閉じたあと
```bash 
sudo apt update
sudo apt install -y docker.io docker-compose-plugin
```

`Ubuntu 24.04 Noble`など、標準APTで docker-compose-plugin が見つからない場合は、以下の公式手順へ
```bash:docker-compose-pluginの公式インストール手順
# 2. 公式Dockerリポジトリを追加してインストール
sudo apt remove docker docker-engine docker.io containerd runc -y
sudo apt install -y ca-certificates curl gnupg

# GPGキー登録
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# リポジトリ追加（自動でUbuntuバージョンを取得して設定）
. /etc/os-release
echo \
  "deb [arch=$(dpkg --print-architecture) \
  signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  ${UBUNTU_CODENAME} stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

インストール完了後
```bash:動作確認/権限グループ追加
docker --version
docker compose version
sudo usermod -aG docker $USER
```
`usermod -aG`でグループに追加後、
- SSH環境：exit → 再接続が一番安全
- 開発PC＋VSCode：VSCodeを再起動（Docker拡張も含めて権限が反映される）
- すぐ試したいとき：`newgrp docker`コマンドで一時的に反映

> `gunicorn` や `flask` をホストにインストールする必要はない（コンテナ内で動く）。

```bash:起動コマンド
# 開発
docker compose --env-file .env.dev build --no-cache --progress=plain
# 本番環境
docker compose --env-file .env.prd build --no-cache --progress=plain
```

:::message
`docker compose`コマンドは、ソース変更を反映させたい場合は`--build`オプションをつける。そうでない場合や`.env`の値だけ変えた場合、本番デプロイ時(CI/CD内)はつける必要はない。
:::
アクセス → `http://localhost:5000`

#### 2-4. 環境変数（.env）について

`Flask`アプリや`Docker Compose`では、接続情報や起動時の設定値を.env`ファイルにまとめることで、ソースコードや構成ファイルにパスワード等を直書きせずに管理できる。

`.env`記載内容のサンプル
```.env.sample:.env.sample
# DB接続情報
DB_NAME=db_name
DB_USER=db_user
DB_PASSWORD=yourpassword
# ↓Docker ComposeにDB_HOSTをDNS解決させる
DB_HOST=db
POSTGRES_USER=ps_user
POSTGRES_PASSWORD=ps_password
POSTGRES_DB=postgres_db

# Auth0設定
AUTH0_CLIENT_ID=auth0_client_id
AUTH0_CLIENT_SECRET=auth0_client_secret
AUTH0_DOMAIN=auth0_domain.com
# ローカル用
AUTH0_CALLBACK_URL=http://callback_url
# 本番デプロイ先のドメイン（またはALBのDNS名）
AUTH0_CALLBACK_URL=https://invoice.example.com/callback
FLASK_SECRET_KEY=flask_secret_key

# Gunicorn設定（詳細は 3.6 参照）
GUNICORN_WORKERS=2
GUNICORN_THREADS=2
GUNICORN_TIMEOUT=60
GUNICORN_MAX_REQUESTS=200
GUNICORN_MAX_REQUESTS_JITTER=50
```
:::message alert
`.env` は `.gitignore`と`.dockerignore`に追加してリポジトリには含めないようにし、代わりにサンプル値を入れた `.env.sample`をGithubに公開する。
:::

```.gitignore:.gitignore(抜粋。.dockerignoreも同じ)
.env
.env.*
!.env.sample
```

:::message
**DB_HOST=db**に変更する
ローカルで動かすとき、これまでは`DB_HOST=localhost`でよかったが、

**localhostは「そのコンテナ自身」**を指す
→ webコンテナから見た`localhost:5432`は、webコンテナの中の5432番ポート
→ 当然`Postgres`なんて動いてないから接続エラー

`db`というサービス名を指定すると、`Compose`が内部DNSで解決してくれる
→ `db` → `Postgres`コンテナのIP
:::

### 3. Gunicornで起動

ここでは、Flaskアプリケーションを Gunicorn 経由で起動し、  
アプリが正しく起動・応答するかどうかを確認する。

#### 3.1 Gunicorn 起動コマンド

```bash
# 例：開発環境
docker compose --env-file .env.dev up --build
# 以降はコードやDockerfileを変えてなければ --build は省略可
```

Gunicornの起動パラメータは `docker-compose.yml` の `command:` もしくは `.env` で制御（詳しくは 3.3 メモリ/スレッド数の調整）。

---

#### 3.2 メモリ / スレッド数の調整

|構成|カスタム|
|---|---|
|小規模構成（Fargate 0.25〜0.5 vCPU / 512MB〜1GB）|`workers=1〜2`、`threads=2〜4`|
|PDF生成が軽い（`weasyprint`で変換 / 既存ファイル返す / S3配信）|`workers=1〜2`、`threads=2〜4`|
|PDF生成が重い（`wkhtmltopdf`等で変換）|CPU型、`workers=2〜3`・`threads=1〜2`[^1]|

[^1]:長時間処理ならジョブキューやLambda化も検討


**.envファイルの例（Gunicorn部分）**
```env
GUNICORN_WORKERS=2
GUNICORN_THREADS=2
GUNICORN_TIMEOUT=60
GUNICORN_MAX_REQUESTS=200
GUNICORN_MAX_REQUESTS_JITTER=50
```
> 2-2 の `docker-compose.yml` の `command:` 部分で `.env` から読み込んで適用

### 4. おわりに
これで、FlaskアプリをDocker Compose + Gunicornで起動できるようになった。  
次は、より高性能な環境への移植や、本番環境へのデプロイを試してみる予定。
:::message alert
Auth0をGoogle連携している場合は、次回のECS Fargateへのデプロイまでにメールアドレス連携に変更しておく。
:::

Flaskアプリシリーズ
@[card](https://zenn.dev/nickelth/articles/reportapp02auth0)
@[card](https://zenn.dev/nickelth/articles/reportapp01flask)