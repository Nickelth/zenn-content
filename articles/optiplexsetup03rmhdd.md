---
title: "爆音HDDを取り外して中古PCを精神的蘇生"
emoji: "😌"
type: "tech"
topics: ["Linux", "bash", "cli"]
published: false
---

## HDD取り外し手順

### 0. はじめに
紆余曲折あって中古PCにSSD装填、LinuxMintの起動に成功した。
日本語入力、Githubのクローンが完了し、作業環境の移行も完了。
しかし、HDD爆音問題が日に日に深刻さを増しているので、この度切除することに決めた。
すでにSSDブートできていることだし。

#### 患部のスペック
Seagate「Barracuda」シリーズの3.5インチ
容量：1TB

### 1. 起動ドライブがSSDかどうかを確認する

#### 方法1. lsblkで見る

```bash
lsblk -o NAME,ROTA,SIZE,MOUNTPOINT,MODEL
```

```plaintext
NAME   ROTA SIZE MOUNTPOINT MODEL
sda       1 1.8T            ST2000DM008 ← SeagateのHDD（回転してる）
nvme0n1   0 500G /          CT500P3SSD8 ← CrucialのSSD（ROTA=0、ルートにマウント）
```
`lsblk`でルート（`/`）が`ROTA=0`なら、もうHDDはガヤです。

#### 方法2. df -h + ls -l /dev/disk/by-uuid/

まず現在のルートディレクトリのデバイスを確認：
```bash
df -h /
```
```plaintext
Filesystem      Size  Used Avail Use% Mounted on
/dev/nvme0n1p2  500G   10G  460G   3% /
```

```bash
ls -l /dev/disk/by-uuid/
```

#### 方法3. dmesgで見る
起動時のストレージ関連ログを確認できる。コアな自分に酔いたいとき用。
```bash
dmesg | grep -i 'nvme\|ssd\|sda'
```

### 2. 取り外し方法（画像付き）

### 3. 取り外した爆音HDDの再雇用用途（感想）
Linux監視カメラのアイデアがあるので、動画の保存先として活用しようと考えていた。
しかしHDDの中身を改造して騒音を改善する方法はないらしい。
ゴムやワッシャーがうんぬん（めんどい）

### 4. 中古PCシリーズ
@[card](https://zenn.dev/nickelth/articles/optiplexsetup01)
@[card](https://zenn.dev/nickelth/articles/optiplexsetup02mint)