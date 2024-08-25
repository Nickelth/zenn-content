---
title:  "自宅サーバの設営"
emoji: "🌸"
type: "tech"
topics: ["Ubuntu"]["ChatGPT"]
published: false
---

自宅サーバの設営にあたり、中古PCにUbuntuをCLIのみでインストールし、セキュリティを高めた設定を行うための手順を以下に示します。

### 1. Ubuntu Serverのインストール準備
- **必要なもの**:
  - Ubuntu ServerのISOイメージ（[公式サイト](https://ubuntu.com/download/server)からダウンロード）
  - ブータブルUSBドライブ（ISOイメージをUSBに書き込むため）
  - 別のPCでISOイメージをUSBに書き込むためのツール（Rufus、UNetbootinなど）

### 2. ブータブルUSBの作成
- ダウンロードしたUbuntu ServerのISOイメージをRufusなどを使ってブータブルUSBに書き込みます。

### 3. 中古PCにUbuntu Serverをインストール
- 中古PCにブータブルUSBを挿入し、PCを起動します。
- BIOS/UEFI設定でUSBからブートするように設定します。
- Ubuntu Serverのインストーラーが起動するので、以下の手順でインストールを進めます:
  1. **言語設定**: 日本語または英語を選択
  2. **ネットワーク設定**: インストール中は有線LANでの接続を確認（DHCPを使用する場合は自動で設定されます）
  3. **インストールタイプ**: 「標準システムユーティリティ」のみを選択し、GUIはインストールしない
  4. **ディスクの設定**: 「ディスク全体を使う」を選択し、パーティションを自動設定
  5. **ユーザー設定**: 管理者アカウントを作成
  6. **SSHのインストール**: 「OpenSSH Server」を選択してインストール

### 4. ファイアウォールの設定
- **ufw（Uncomplicated Firewall）のインストールと有効化**:
  ```bash
  sudo apt update
  sudo apt install ufw
  sudo ufw allow OpenSSH
  sudo ufw enable
  ```
- **外部アクセスをブロック**:
  ```bash
  sudo ufw default deny incoming
  sudo ufw default allow outgoing
  ```
  これにより、SSHを許可する以外の全ての外部からのアクセスを拒否します。

### 5. SSHの設定強化
- **SSH設定ファイルの編集**:
  ```bash
  sudo nano /etc/ssh/sshd_config
  ```
  以下の設定を確認・変更します:
  - `PermitRootLogin no`（rootでのログインを禁止）
  - `PasswordAuthentication no`（パスワード認証を無効化し、公開鍵認証のみ許可）
  - `AllowUsers`にアクセスを許可するユーザー名を追加（例: `AllowUsers yourusername`）

- **SSHサービスの再起動**:
  ```bash
  sudo systemctl restart ssh
  ```

### 6. ローカルネットワーク内のPCからアクセスするための設定
- 無線LANに接続している個人PCにSSHクライアント（例えばPuTTY）をインストールし、サーバーのIPアドレスを使ってアクセスします。
- サーバーのIPアドレスは、有線LANインターフェースのIPを確認して使用します。確認方法は以下の通りです:
  ```bash
  ip a
  ```
  `inet`の後に表示されるのがIPアドレスです。

### 7. 外部アクセスの遮断
- サーバー側で確実に外部アクセスを遮断するため、ルーターの設定でサーバーへの外部からのアクセスを禁止するルールを設定します。
- 必要であれば、ホストベースのファイアウォール（`iptables`や`nftables`など）を使用してさらに厳格なアクセス制御を行います。

### 8. その他のセキュリティ対策
- **定期的なアップデート**:
  ```bash
  sudo apt update && sudo apt upgrade -y
  ```
- **不要なサービスの停止**:
  ```bash
  sudo systemctl disable <サービス名>
  sudo systemctl stop <サービス名>
  ```
- **ログ監視の設定**: `rsyslog`や`logwatch`をインストールして、システムログを定期的に確認します。

### 9. 無線LANからのアクセス確認
- サーバーの設定が完了したら、無線LANに接続しているPCからSSHでアクセスできるか確認します。問題なく接続できれば、インストールと設定は完了です。

これで、自宅サーバの基本的なセキュリティ設定を含むインストール手順は完了です。