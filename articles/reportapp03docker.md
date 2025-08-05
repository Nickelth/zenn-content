---
title: "【#3】Flask+Docker+Gunicornで本番環境準備"
emoji: "🦄"
type: "tech"
topics: ["flask", "docker", "gunicorn"]
published: false
---

## 

### 0. はじめに

Flaskアプリを公開するために本番環境が必要になった。
当初はVPSで実現する予定だったが、AWSで実施することにした。
価格相応のトラブルが出たことと、実務経験を活かしたいことが理由。

Flaskアプリで本番環境を構築中にVPSの設置が必要になった。
記事の構成要素が増加したため、その部分だけ独立して記事を執筆する。

#### VPSでの失敗談

![](https://storage.googleapis.com/zenn-user-upload/e1a701a3b638-20250727.png)
![](https://storage.googleapis.com/zenn-user-upload/9d45585892e0-20250727.png)
「毎時1.9円」「シャットダウン中は課金されない」「UIが日本語」という魅力的な内容だったため契約して本番環境にしようとした結果

- 「ConoHa」で検索した結果、VPSではなく「WING」という全く別のものを契約してしまった
- VPSを契約後SSH接続しようとしたがなぜかPingすら通らない
- VPS側のターミナルはブラウザ版なので文字入力直接コピペできなくて使いづらすぎる
- そのターミナルで調べた結果Pingは受け取っているが謎の設定で破棄されていることが判明

おそらく低価格を維持するためセキュリティを強固にして人件費を削減しているのかと勘ぐってみたり
いずれにしろ日曜日が無駄にされたのでAWSに乗り換え


### 1. 選定理由
- **Docker**::
- **Gunicorn**: PythonのWSGIサーバーとして、Flaskアプリを効率よく動作させるために選定。

#### WSGIサーバー（Gunicorn）について

Flaskは開発用サーバーを内蔵しているが、本番運用には適していない。  
そこで登場するのが **WSGI（Web Server Gateway Interface）** というPythonの標準インターフェース。  
WSGIは「Webサーバー（Nginxなど）」と「Pythonアプリケーション（Flaskなど）」の**橋渡し役**を担う。

GunicornはそのWSGI仕様に則った、**高性能かつシンプルなWSGIサーバー**である。  
複数のワーカー（プロセス）を立ち上げ、リクエストを並列処理できるため、スケーラビリティと安定性が向上する。

簡単に言えば：

- Nginx：ラーメン店の受付。リクエストを受けてGunicornに渡す。
- Gunicorn：厨房の責任者。Flaskアプリに仕事を渡して、結果をNginxに返す。
- Flaskアプリ：実際にラーメンを作る人

この構成により、Flaskアプリを本番環境で**安全かつ効率的に**動かすことが可能となる。

---

### 2. Flaskアプリ起動(Gunicorn前提)

#### 2.1 必要パッケージのインストールと起動
```bash
# パッケージリスト更新
sudo apt update

# Nginxのインストール
sudo apt install -y nginx

# Python仮想環境を有効にした状態でGunicornをインストール
pip install gunicorn
```

Flaskアプリが `app.py` にあり、グローバルに `app = Flask(__name__)` が定義されている場合の起動方法：

```bash
gunicorn -w 4 -b 127.0.0.1:8000 app:app
```

Flaskアプリが create_app() という関数で返されるファクトリーパターンを採用している場合は、
wsgi.py など別ファイルでアプリを生成して Gunicorn から読み込ませる必要がある：

```python:wsgi.py
from app import create_app
app = create_app()
```
```bash:Flaskアプリ起動
gunicorn -w 4 -b 127.0.0.1:8000 app:app
```
:::message
`-w 4` → ワーカー数。基本の目安は `(2 × CPUコア数) + 1` だが、軽量なアプリなら 4〜6 でも十分。  
過剰に増やすとメモリを圧迫するので、実行環境に応じて調整する。
`b` → バインド先（Nginxからアクセスできるよう127.0.0.1推奨）
:::


#### 2.2 VPS構築して動作確認可能にする

ここでは、VPS（仮想専用サーバー）上に Flask アプリを配置し、  
Gunicorn + Nginx を使って IP アドレスで直接アクセスできる状態を構築する。

##### ■ アプリ配置と依存パッケージのインストール

```bash
# Flaskアプリのクローン（またはSCPなどで転送）
git clone https://github.com/Nickelth/Papyrus.git
cd your-flask-app

# 仮想環境を作成して依存関係をインストール
sudo apt install python3-venv
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

```bash:Gunicornでアプリを起動
# Gunicornを仮想環境内にインストール（未インストールの場合）
pip install gunicorn

# FlaskアプリをGunicornで起動
gunicorn -w 4 -b 127.0.0.1:8000 app:app
```




:::message alert
- アプリを常時公開する必要はない。必要な期間だけポートを開ける。
- 公開期間が終わったら `sudo ufw deny 80` や `sudo systemctl stop nginx` などで閉じる。
- 公開用に限定パスワードを設けたり、Basic認証をつけるのも手。
:::

#### 2.3 Nginx設定ファイル

```nginx
# /etc/nginx/sites-available/papyrus（ファイル名は任意）
server {
    listen 80;
    server_name example.com;  # IPでもOK、localhostならそれで

    location / {
        proxy_pass http://127.0.0.1:8000;  # Gunicornのバインド先
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /static/ {
        alias /path/to/your/project/static/;
    }
}
```
```bash:シンボリックリンクで有効化
sudo ln -s /etc/nginx/sites-available/papyrus /etc/nginx/sites-enabled/
sudo nginx -t  # 構文チェック
sudo systemctl restart nginx
```
`ln -s` は「シンボリックリンク（ショートカット的なもの）」を作成するコマンド。

Nginxでは `/etc/nginx/sites-available/` に構成ファイルを置き、  
そのファイルを `/etc/nginx/sites-enabled/` にリンクすることで有効化する仕組みになっている。

このコマンドで、作成した設定ファイル `papyrus` を Nginx に読み込ませるようにする。

全て終了後、動作確認
```bash
curl http://localhost
```
Flaskのトップページが表示されればリバースプロキシ成功

### 3. Gunicornで起動

ここでは、Flaskアプリケーションを Gunicorn 経由で起動し、  
アプリが正しく起動・応答するかどうかを確認する。

#### 3.1 起動前の準備

以下の前提条件を確認しておく：

- Flaskアプリが `app.py` にあり、`create_app()` または `app` インスタンスが定義されている
- 必要なパッケージがインストールされていること（`gunicorn`, `flask`, etc.）

#### 3.2 Gunicorn 起動コマンド

例：アプリインスタンスが `app.py` に定義されている場合

```bash
gunicorn -w 4 -b 127.0.0.1:8000 app:app
```

オプションの意味：

* `-w 4`：4つのワーカープロセスで処理（CPUコア数に応じて調整）
* `-b 127.0.0.1:8000`：ローカルホスト上で8000番ポートをバインド
    → Nginxからアクセス可能になる

※ `app:app` の左側はファイル名（拡張子なし）、右側はFlaskアプリのインスタンス名。

---

#### 3.3 デーモンとしてバックグラウンド起動（任意）

本番運用では、Gunicornをバックグラウンドで起動することが多い。

```bash
gunicorn -w 4 -b 127.0.0.1:8000 app:app --daemon
```

ログをファイルに出力したい場合：

```bash
gunicorn -w 4 -b 127.0.0.1:8000 app:app \
  --daemon \
  --access-logfile /var/log/gunicorn/access.log \
  --error-logfile /var/log/gunicorn/error.log
```

#### 3.4 動作確認

Gunicornが起動している状態で、以下のコマンドでアプリが応答しているかを確認：

```bash
curl http://127.0.0.1:8000
```

* 正常であれば、FlaskアプリのトップページのHTMLが返ってくる
* 返ってこない場合は、ログやバインド設定を再確認すること

#### 3.5 プロセスの停止

Gunicorn を手動で停止したい場合は、`ps` コマンドで PID を確認して `kill`：

```bash
ps aux | grep gunicorn
kill <PID>
```

または `pkill` を使ってまとめて終了：

```bash
pkill gunicorn
```






### 4. おわりに


Flaskアプリシリーズ
@[card](https://zenn.dev/nickelth/articles/reportapp02auth0)
@[card](https://zenn.dev/nickelth/articles/reportapp01flask)