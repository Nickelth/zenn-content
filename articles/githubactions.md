---
title: "FlaskアプリをGithub Actionsでデプロイして全世界に公開"
emoji: "☯"
type: "tech"
topics: ["Ubuntu", "linux", "nginx", "github", "Gunicorn"]
published: false
---

## Github Actions + Gunicorn + Nginx

### 0. はじめに
先日、Flaskアプリを作成した。
@[card](https://zenn.dev/nickelth/articles/outputreportpy)


```plaintext
Client (ブラウザ)
    ↓
Nginx (Reverse Proxy)
    ↓
Gunicorn (WSGIサーバー)
    ↓
Flask App
    ↓
PostgreSQL（などのDB）
```

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

`ps aux` を打って以下のように`/sbin/init`の表示を確認
![](https://storage.googleapis.com/zenn-user-upload/ce56f0db93cb-20250717.png)

コマンドプロンプトで`ipconfig /all`を入力してDNSサーバーのIPアドレスを確認
``` c
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

``` bash:Nginxインストール
sudo apt update
sudo apt install -y nginx
```

```bash:Flaskアプリ起動
# 例：app.py に create_app() がある場合
gunicorn -w 4 -b 127.0.0.1:8000 app:app
```
:::message
`w 4` → ワーカー数（CPUコア数に合わせる）
`b` → バインド先（Nginxからアクセスできるよう127.0.0.1推奨）
:::


```bash:Nginxの起動・ステータス確認
sudo systemctl start nginx
sudo systemctl enable nginx
sudo systemctl status nginx
```

```bash:ファイアウォール設定(任意)
sudo ufw allow 'Nginx Full'
sudo ufw status
```
#### 2.2 Nginx設定ファイル

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

全て終了後、動作確認
```bash
curl http://localhost
```
Flaskのトップページが表示されればリバースプロキシ成功

### 3. Gunicornで起動


