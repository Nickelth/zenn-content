---
title: "【Ubuntu】Python実行環境構築例【Apache2】"
emoji: "🐧"
type: "tech"
topics: ["ubuntu", "wsl", "apache", "postgres", "python"]
published: true
---

<!--## .env付きのWSL2構成例（Python/DB/Apache）-->
## WSL2構成例（Python/DB/Apache）
※有線LAN接続が前提
※実際の業務とは無関係な個人検証をもとにした内容です。

### 1.	仮想環境(WSL2)の有効化

#### WSLを有効化する

Windows+R⇒「control」を入力して「Enter」⇒コントロールパネルが開く
![](https://storage.googleapis.com/zenn-user-upload/5e222502c944-20250717.png)

コントロールパネル＞プログラム＞プログラムと機能
「Windowsの機能の有効化または無効化
![](https://storage.googleapis.com/zenn-user-upload/0e6f32fe287a-20250717.png)

「Linux用Windowsサブシステム」にチェックを入れる
![](https://storage.googleapis.com/zenn-user-upload/d8b7f702052b-20250717.png)

スクロールして、「仮想マシンプラットフォーム」にもチェックを入れる
![](https://storage.googleapis.com/zenn-user-upload/0ea8758c0eea-20250717.png)

変更が完了したら、PCを再起動する。

#### WSLのバージョンが2であることを確認する

「スタート」からPowershellを検索して以下のコマンドを入力
``` powershell
wsl --version
```

#### WSL本体をアップデートする
``` powershell
wsl --update
```

完了後、念のためバージョン確認
``` powershell
wsl --version
```

バージョンが2出ない場合は以下を入力
``` powershell
wsl --set-default-version 2
```

### 2.	Windows標準のLinux仮想環境上にUbuntu 24.04.1をインストールする。
「スタート」→「Microsoft Store」で「Ubuntu 24.04」を検索
「入手」⇒「開く」
![](https://storage.googleapis.com/zenn-user-upload/471e6c3fdf7a-20250717.png)

黒いターミナルの初期設定画面が出るので、任意のユーザー名・パスワードを設定する
![](https://storage.googleapis.com/zenn-user-upload/c1881cd108ac-20250717.png)

rootパスワードを設定する
``` bash
sudo passwd root
```
![](https://storage.googleapis.com/zenn-user-upload/e75b7943cf43-20250717.png)



### 3.	wsl.conf&resolv.confの設定
```bash
sudo nano /etc/wsl.conf
```
`wsl.conf`の`[boot]`を`systemd=true`にする
    `systemctl`の利用、サービス自動起動などのため
```conf:/etc/wsl.conf
[boot]
systemd=true

[user]
default=username
```
Ctrl+Oで保存
Enter
Ctrl+Xで離脱

```bash
sudo nano /etc/resolv.conf
```
`generateResolvConf = false`でUbuntu再起動時のDNS再生成を防止
``` conf:/etc/resolv.conf
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



### 4.	Apacheインストール
``` bash
sudo apt upgrade
sudo apt -y install apache2
```

`dpkg-query -l`で確認
![](https://storage.googleapis.com/zenn-user-upload/7d999c129c63-20250717.png)

`sudo systemctl status`で●緑丸やrunningが出ることを確認
![](https://storage.googleapis.com/zenn-user-upload/84e0c356d729-20250717.png)
`q`を押して離脱

Apacheを起動
``` bash
sudo systemctl start apache2
```

#### SSLの有効化
``` bash
sudo a2enmod ssl	
sudo a2ensite default-ssl	
sudo systemctl restart apache2	
```
![](https://storage.googleapis.com/zenn-user-upload/e8e04d13efaa-20250717.png)

#### Apache起動確認
ブラウザで、下記にそれぞれアクセスし起動を確認する。														
IPアドレスについては、起動ごとに変わるため`hostname -I`で確認する。
>    http://<WSLのIPアドレス>/													
>    https://<WSLのIPアドレス>/
※ HTTPSアクセス時には証明書の警告が表示されるが、開発環境のため許可して問題なし			
![](https://storage.googleapis.com/zenn-user-upload/8b7f922dcd23-20250717.png)
「It works!」が出ればOK


#### Apache2アンインストール方法
後続PJとの相性を鑑み、Apache2を削除(7/27)
```bash
sudo systemctl stop apache2
sudo apt purge -y apache2 apache2-utils apache2-bin apache2.2-common
sudo apt autoremove -y
sudo rm -rf /etc/apache2
```


### 5.	PostgreSQLのインストール
``` bash
sudo apt -y install postgresql
sudo psql --version
```

サービス起動
``` bash
sudo systemctl start postgresql
```

パスワード設定
``` bash
sudo passwd postgres
```

ユーザpostgresに遷移
``` bash
su - postgres
```

テスト用DB作成
``` sql
createdb testdb -O postgres
```

`\q`でDBから抜ける

`exit` rootに戻る

### 6.	Python のインストール
``` bash
sudo apt install -y python3 python3-pip python3-venv python3-dev build-essential libpq-dev
# 仮想環境env作成（プロジェクトごとに）
python3 -m venv env
source env/bin/activate
```

### 7. おわりに
Python + Postgres の開発環境をUbuntu(WSL)上に作成できた。

~~この環境をベースにアプリ開発を実施する予定。~~
アプリ開発記事を公開。
@[card](https://zenn.dev/nickelth/articles/outputreportpy)