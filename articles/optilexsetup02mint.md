---
title: "中古PC初回起動後のLinuxセットアップ手順書"
emoji: "🔧"
type: "tech"
topics: ["Linux", "bash", "cli"]
published: false
---

## 

### 0. はじめに
先日中古PCのハードウェア面でのセットアップを実施した。
この記事では、Linuxインストール直後に実施すべきコマンド、ユーザ操作コマンド、Githubクローンコマンドについて順に掲載する。

特にキーボード日本語化についてはかなり詰まりやすいので、画像付きで解説する。

使用OS: Linux Mint Cinnamon

### 1. 初期設定用コマンド（省電力化など）

基本パッケージ操作
```bash
# システムアップデート
sudo apt update && sudo apt list --upgradable && sudo apt upgrade -y

# Flatpak削除
sudo apt remove --purge flatpak -y

# ファイル検索高速化
sudo apt install mlocate -y
sudo updatedb

# CLI充実化
sudo apt install -y curl git neofetch htop tlp tlp-rdw powertop
```

省電力化
```bash
sudo apt install tlp tlp-rdw -y
sudo systemctl enable tlp
sudo systemctl start tlp

# Powertopチューニング
sudo powertop --auto-tune

sudo reboot
```

### 2. キーボードを日本語化する

```bash
sudo apt install fcitx-mozc mozc-utils-gui -y

# Fcitxを使用するように切り替え
im-config -n fcitx
```
切替後、ログアウト→ログイン

Fcitxツールを開く
```bash
fcitx-configtool
```
![](https://storage.googleapis.com/zenn-user-upload/57ee53c15490-20250805.png)


### 3. conkyでCPU・メモリの状態をリアルタイム表示

### 4. その他のコマンド

#### ユーザ追加
```bash
sudo adduser
```

#### ユーザをsudoグループに追加
```bash
sudo usermod -aG sudo ユーザー名
```

#### 実ユーザを表示
```bash
# ID >= 1000 のユーザ（自分で作ったユーザ）を表示
# nobodyは65534なので注意
awk -F: '$3 >= 1000 { print $1 }' /etc/passwd
```

#### rootパスワード変更（sudoユーザ前提）
```bash
sudo passwd root
```

#### Githubクローン
```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```
この後コマンド待機までEnterキー連打

生成された公開鍵を表示
```bash
cat ~/.ssh/id_ed25519.pub
```
この出力をGitHubの SSH keys 設定ページ に追加。

GitHubからSSHでクローン
```bash
git clone git@github.com:ユーザー名/リポジトリ名.git
```

```bash
ssh -T git@github.com
```

成功すると、以下のメッセージが表示される。
```plaintext
Hi ユーザー名! You've successfully authenticated...
```

### 5. 中古PCシリーズ
@[card](https://zenn.dev/nickelth/articles/optilexsetup01)
@[card](https://zenn.dev/nickelth/articles/optilexsetup03rmhdd)