---
title: "FlaskアプリをGunicorn + Nginxで本番公開するまでの全手順"
emoji: "🦄"
type: "tech"
topics: ["Ubuntu", "linux", "nginx", "github", "Gunicorn"]
published: false
---

## 本番環境の作成（Gunicorn + Nginx）

### 0. はじめに

先日、Flaskアプリを作成した。  
@[card](https://zenn.dev/nickelth/articles/outputreportpy)

業務上、デプロイ手段をGitHub Actionsへ変更する案が持ち上がった。  
その前段階として、本アプリを用いて構成を確認する必要があった。  
その過程で、**Gunicorn + Nginx** による本番環境の構築が必要となり、検討・実装を行った。

#### 使用技術の選定理由

- **Nginx**: 静的ファイルの配信やリバースプロキシとしての役割を担う。負荷分散やセキュリティ強化にも貢献する。
- **Gunicorn**: PythonのWSGIサーバーとして、Flaskアプリを効率よく動作させるために選定。

#### 前提
- Debian系Linux環境(WSL可)
- Linux内でVSCodeが開く
- Apacheをインストールしていない
- 有線LANにつながっている
- VPSを1台用意（例：ConoHa / さくらVPS / Lightsail / etc.）
    - Ubuntu 20.04 or 22.04がインストールされている前提
    - SSH接続が可能であること
- FlaskアプリがGitHubなどで管理されていること

#### アプリ構成図

```plaintext
Client (ブラウザ)
    ↓
Nginx (Reverse Proxy)
    ↓
Gunicorn (WSGIサーバー)
    ↓
Flask App
    ↓
SQLAlchemy
    ↓
PostgreSQL（などのDB）
```

#### WSGIサーバー（Gunicorn）について

Flaskは開発用サーバーを内蔵しているが、本番運用には適していない。  
そこで登場するのが **WSGI（Web Server Gateway Interface）** というPythonの標準インターフェース。  
WSGIは「Webサーバー（Nginxなど）」と「Pythonアプリケーション（Flaskなど）」の**橋渡し役**を担う。

GunicornはそのWSGI仕様に則った、**高性能かつシンプルなWSGIサーバー**である。  
複数のワーカー（プロセス）を立ち上げ、リクエストを並列処理できるため、スケーラビリティと安定性が向上する。

簡単に言えば：

- Nginx：入り口の受付。リクエストを受けてGunicornに渡す。
- Gunicorn：中の処理係。Flaskアプリに仕事を渡して、結果をNginxに返す。
- Flaskアプリ：実際に中身の処理をする人

この構成により、Flaskアプリを本番環境で**安全かつ効率的に**動かすことが可能となる。

---

### 1.	wsl.conf&resolv.confの設定
`generateResolvConf = false`でUbuntu再起動時のDNS再生成を防止
``` conf
[network]
generateResolvConf = false

[boot]
systemd = true
```
Ctrl+Oで保存
Enter
Ctrl+Xで離脱

PowerShell側からWSLを再起動
``` powershell
wsl --shutdown
```
※シャットダウン後、Ubuntuに接続すると自動起動する。

`ps aux`（実行中の全プロセスの一覧表示）を実行し、`/sbin/init` が含まれているかを確認。
これは systemd が起動プロセスとして動作しているかの目安になる。
![](https://storage.googleapis.com/zenn-user-upload/ce56f0db93cb-20250717.png)

コマンドプロンプトで `ipconfig /all` を実行し、使用中のネットワークアダプターの「DNS サーバー」欄にある IP アドレスを確認。
``` plaintext
DNS サーバー . . . . . . . . . . . : 192.168.1.1
```

Ubuntuを立ち上げ、以下のコマンドで疎通確認(ping先は実際のDNSサーバーのIPアドレスに合わせる)
``` bash
ping 192.168.1.1
sudo apt update
```

Ubuntu再起動時のresolv.confの初期化を防ぐため、以下のコマンドを入力
``` bash
sudo chattr +i /etc/resolv.conf
sudo systemctl disable systemd-resolved.service
sudo systemctl stop systemd-resolved.service
```

### 2. Flaskアプリ起動(Gunicorn前提)

#### 2.1 必要パッケージのインストールと起動


```bash:Flaskアプリ起動
# 例：app.py に create_app() がある場合
gunicorn -w 6 -b 127.0.0.1:8000 app:app
```
:::message
`w 6` → ワーカー数（CPUコア数に合わせる）
`b` → バインド先（Nginxからアクセスできるよう127.0.0.1推奨）
:::


```bash:Nginxの起動・ステータス確認
sudo systemctl start nginx
sudo systemctl enable nginx
sudo systemctl status nginx
```

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

``` bash:Nginxインストール
sudo apt update
sudo apt install -y nginx
```

`ufw`（Uncomplicated Firewall）は、Ubuntuに標準で用意されているシンプルなファイアウォール管理ツール。
以下のコマンドで、Nginxの通信を許可する：
```bash:ファイアウォール設定
sudo ufw allow 'Nginx Full'
sudo ufw status
```

```bash:ポート番号指定で開ける場合
sudo ufw allow 80/tcp
sudo ufw status
```

```bash:VPSのグローバルIPアドレス確認
hostname -I
```

```plaintext
http://YOUR.VPS.IP.ADDRESS/
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


以上で、Gunicorn による Flask アプリの起動および動作確認が完了。
次回はGithub Actions + DockerでCI/CDを構築する。
systemd を使ってサービス化してもよい。
