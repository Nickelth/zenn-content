---
title: "中古PC初回起動後のLinuxセットアップ手順書"
emoji: "🔧"
type: "tech"
topics: ["Linux", "bash", "cli"]
published: true
---
## Linuxセットアップコマンド

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

### 2. 画面比を1920x1980(FullHD)にする
再起動時も引き継ぐ設定にするために、`.xprofile`に書き込む。

```bash
nano ~/.xprofile
```
```plaintext
xrandr --newmode "1920x1080_60.00" 173.00 1920 2048 2248 2576 1080 1083 1088 1120 -hsync +vsync
xrandr --addmode DP-1 "1920x1080_60.00"
xrandr --output DP-1 --mode "1920x1080_60.00"
```
`Ctrl+O` → `Enter` → `Ctrl+X` で保存・終了
その後再起動

### 3. キーボードを日本語化する

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
日本語を選択
![日本語を選択](https://storage.googleapis.com/zenn-user-upload/f8e8a21518ca-20250805.png)

「入力メソッドのオン/オフ」が「ZenkakuHankaku」になっていることを確認
![入力メソッドのオン/オフ](https://storage.googleapis.com/zenn-user-upload/57ee53c15490-20250805.png)


### 4. ユーザー系の操作コマンド

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

### 5. おわり
最後の方は初回セットアップと関係なさそうなコマンドが続いたが、環境移行時用についでに書いたほうが調べる手間が減って楽だと感じたので追記した。
ノートPCでDocker設定を終えたら中古PCに環境移行してみたい。

### 5. 中古PCシリーズ
@[card](https://zenn.dev/nickelth/articles/optiplexsetup01)
@[card](https://zenn.dev/nickelth/articles/optiplexsetup03rmhdd)