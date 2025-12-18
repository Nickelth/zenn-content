---
title: "【負け】ConoHaVPSで本番環境立てようとしたら失敗して日曜日が溶けた話"
emoji: "🦄"
type: "tech"
topics: ["cli", "linux", "bash"]
published: false
---

## 「WINGとVPSを間違え、/etc/resolv.confに祟られ、rp_filterに殴られた話」

### 目的の再定義

ベンダ叩きは素人臭い。評価軸と実測で「採用しない判断」を証跡付きで示す。

### 集める証拠

損耗の実測（ping loss/latency、再現条件、時刻）

iptables/nftablesやufwの設定差分とログ抜粋

代替案の費用比較（運用工数も含めた概算）

### 見出しの骨格

背景と制約（コスト上限、必要SLA、マネージド要件）

評価軸（可用性、運用負荷、拡張性、サポート窓口）

実験と結果（時系列ログ）

決定：採用しない理由と代替案

移行時の教訓（ヘルスチェック、二重化の最低限）

### タイトル案

「採用しなかった理由を書く：VPS評価ログと撤退判断」

「可用性は“安さ”で買えない：VPS検証の実測と結論」

「本番要件に届かなかったのでやめた、という成果」

### 捨てるもの

長尺の設定スクショ。必要な差分だけ。

### 0. はじめに

転職用のFlaskアプリを「必要なときだけ起動」したくてConoHaの**VPS**を採用。ところが初手で**WING契約**、次に**SSHタイムアウト地獄**。最終的にDNS・ルーティング・セキュリティグループ・ufw・iptables/nft などの**典型的ハマりポイント**を総当たりで解決していった記録。

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

### 1

![](https://storage.googleapis.com/zenn-user-upload/ee2d5819bc12-20250818.png)

![](https://storage.googleapis.com/zenn-user-upload/93b8c26916ac-20250818.png)

![](https://storage.googleapis.com/zenn-user-upload/ade100c010b4-20250818.png)

:::details /var/log/auth.log

```plaintext
$ tail -f /var/log/auth.log
2025-07-27T19:17:01.160148+09:00 Ryot CRON[84257]: pam_unix(cron:session): session opened for user root(uid=0) by root(uid=0)
2025-07-27T19:17:01.163098+09:00 Ryot CRON[84257]: pam_unix(cron:session): session closed for user root
2025-07-27T19:25:02.051274+09:00 Ryot CRON[85775]: pam_unix(cron:session): session opened for user root(uid=0) by root(uid=0)
2025-07-27T19:25:02.054427+09:00 Ryot CRON[85775]: pam_unix(cron:session): session closed for user root
2025-07-27T19:35:01.690864+09:00 Ryot CRON[87678]: pam_unix(cron:session): session opened for user root(uid=0) by root(uid=0)
2025-07-27T19:35:01.693114+09:00 Ryot CRON[87678]: pam_unix(cron:session): session closed for user root
2025-07-27T19:45:01.300438+09:00 Ryot CRON[89584]: pam_unix(cron:session): session opened for user root(uid=0) by root(uid=0)
2025-07-27T19:45:01.302297+09:00 Ryot CRON[89584]: pam_unix(cron:session): session closed for user root
2025-07-27T19:55:01.049070+09:00 Ryot CRON[91506]: pam_unix(cron:session): session opened for user root(uid=0) by root(uid=0)
2025-07-27T19:55:01.051815+09:00 Ryot CRON[91506]: pam_unix(cron:session): session closed for user root
```

:::

![](https://storage.googleapis.com/zenn-user-upload/fdfc7c464cb9-20250818.png)

![](https://storage.googleapis.com/zenn-user-upload/c553e2eeb87a-20250818.png)

![](https://storage.googleapis.com/zenn-user-upload/d0fd329fec7a-20250818.png)

![](https://storage.googleapis.com/zenn-user-upload/e1542f1227a2-20250818.png)

![](https://storage.googleapis.com/zenn-user-upload/3f6c2859561e-20250818.png)
![](https://storage.googleapis.com/zenn-user-upload/8a46ebfe6872-20250818.png)

![](https://storage.googleapis.com/zenn-user-upload/e0869363689f-20250818.png)
![](https://storage.googleapis.com/zenn-user-upload/bae9c57422a0-20250818.png)

![](https://storage.googleapis.com/zenn-user-upload/c0a47c01827a-20250818.png)
![](https://storage.googleapis.com/zenn-user-upload/300939be312a-20250818.png)

![](https://storage.googleapis.com/zenn-user-upload/a8aa2f93d031-20250818.png)
![](https://storage.googleapis.com/zenn-user-upload/5fa1febabb10-20250818.png)
![](https://storage.googleapis.com/zenn-user-upload/8e665f77d4c4-20250818.png)

## 0. まず結論（TL;DR）

- 何が原因で、どう直したかを最短で列挙。
- 例：**WING≠VPS**、**resolv.confがsymlink**、**デフォルトルート無し**、**セキュリティグループ未適用 or 期待と違う**、**rp_filter=2**、**ufw/iptables/nftの整合**など。

## 1. ゴールと前提

- 目的：_「必要時のみ起動してIP直アクセスで確認。一般公開はしない/最小コスト」_
- 環境：ConoHa VPS（Ubuntu 22.04/24.04 想定）、ローカルはWindows + WSL
- 鍵運用方針：`.pem` を使う、**root直は初期のみ**→後で一般ユーザ + sudo へ

## 2. 開幕ミス：WINGとVPSは別物

- **WING**＝共用サーバ（起動/停止の概念が弱い、WordPress向け）
- **VPS**＝仮想サーバ（root権限・起動/停止/爆破が自由）
- 画像：WING削除→VPS作成の画面（※君のスクショをここに）

## 3. 鍵で入れ：`.pem` の配置と権限

- 置き場所：`~/.ssh/your-key.pem`、権限：`chmod 600`
- 接続：`ssh -i ~/.ssh/your-key.pem root@<IP>`
- rootで`sudo`を打つ無意味さネタ（WSLの癖）も一言だけ添える

## 4. DNS地獄：`/etc/resolv.conf` は**symlink**だった

- 症状：`ping github.com` / `apt update` が死ぬ、`nameserver 127.0.0.53` など謎設定
- 対処の流れ：
  1. `systemctl stop/disable systemd-resolved`
  2. `/etc/resolv.conf` の**リンクを削除** → **実体ファイルで再作成**
  3. `nameserver 8.8.8.8`（+ `1.1.1.1`）だけにする
  4. `chattr +i` で固定（必要時は `-i` で解除）

- 補足：`options edns0 trust-ad`/`search .` は用途が分からなければ削除でOK
- 画像：symlink状態の`ls -l /etc/resolv.conf`（→リンク先表示のやつ）

```bash
sudo systemctl stop systemd-resolved
sudo systemctl disable systemd-resolved
sudo rm /etc/resolv.conf
printf "nameserver 8.8.8.8\nnameserver 1.1.1.1\n" | sudo tee /etc/resolv.conf
sudo chattr +i /etc/resolv.conf
```

## 5. SSHタイムアウトの**定石デバッグ手順**

- サーバ起動確認、**正しいグローバルIP**確認（管理画面のやつ）
- `systemctl status ssh` / `ss -tlnp | grep :22` / `.pem`権限 / `tail -f /var/log/auth.log`
- `ssh -vvv` の使い所
- 画像：`auth.log`に何も出ない＝「そもそも届いてない」パターン

## 6. **セキュリティグループ**と**ufw**は別物（外壁 vs 内壁）

- ConoHa管理画面の**セキュリティグループ**＝外部インバウンド制御
- OSの`ufw`/`iptables`/`nft`＝内部のパケットフィルタ
- ありがちな落とし穴：
  - **default**グループが「All許可」に見えても**インスタンスに未アタッチ**
  - **IPv6だけ許可**で**IPv4閉鎖**

- 最低限の開放：22/80/443（IPv4側で）
- 画像：`ssh-allow`を作って22/TCP許可→VPSへアタッチの画面

## 7. 外へ出られない：**デフォルトルート無し**

- 症状：`ip route` に `default via <GW>` が無い
- 一時対応：`ip route add default via <gateway> dev eth0`
- 永続化：`/etc/netplan/*.yaml` に `routes:`（or `gateway4:`）設定して `netplan apply`
- 画像：`ip route` のビフォー/アフター

## 8. それでも返事が来ない：**rp_filter** の壁

- `net.ipv4.conf.all.rp_filter=2`（デフォ）で**戻りパケットが落ちる**ことがある
- 試すなら一時的に `0`（無効）へ → `sysctl -p`
- 効いたら永続化（`/etc/sysctl.conf`）
- 画像：`sysctl` 出力のスクショ

```bash
echo "net.ipv4.conf.all.rp_filter=0" | sudo tee -a /etc/sysctl.conf
echo "net.ipv4.conf.default.rp_filter=0" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

## 9. まだ臭う：**ufw / iptables / nftables** の整合

- どれが有効か分からなくなったら**一度リセット**して最小構成に戻す
- 例（要注意・自己責任）：

```bash
sudo ufw reset && sudo ufw disable
sudo iptables -F && sudo iptables -X
sudo iptables -t nat -F && sudo iptables -t nat -X
sudo iptables -t mangle -F && sudo iptables -t mangle -X
sudo nft flush ruleset
sudo ufw allow ssh && sudo ufw enable
```

- 画像：`iptables -L -v -n | grep -E 'DROP|REJECT'` の結果

## 10. パケットで見る：`tcpdump` と `auth.log` の合わせ技

- `tcpdump -i eth0 port 22`：**届いてるか**を見る
- `tail -f /var/log/auth.log`：**sshdが反応してるか**を見る
- 「届いてるのに応答無し」→**戻りが落ちてる/ルート不正**
- 画像：tcpdumpでSYN/SYN-ACKが見える例

## 11. 小ネタ：WSLの癖（rootでsudo叩く等）

- **root中の`sudo`はただの丁寧語**。直してもいいし、直さなくても人生は続く。

## 12. 学び（チェックリスト）

- \*\*クラウド外壁（セキュリティグループ）**と**OS内壁（ufw等）\*\*を混同しない
- `/etc/resolv.conf` が**symlink**ならまず破壊→実体化→固定
- `ip route` に `default via` が無ければ**外に返事できない**
- `rp_filter` は一時的に疑ってもいい
- 迷ったら**tcpdump + auth.log**で「届く/返す」を分解
- そもそも**正しいグローバルIP**に向かっているか毎回確認

## 13. 付録A：使ったコマンド・チートシート

- ネットワーク：`ip a`, `ip route`, `ping`, `curl ifconfig.me`
- SSH：`ssh -vvv`, `systemctl status ssh`, `ss -tlnp | grep :22`, `tail -f /var/log/auth.log`
- DNS：`ls -l /etc/resolv.conf`, `systemctl stop/disable systemd-resolved`, `chattr +/-i`
- FW：`ufw status verbose`, `iptables -L -v -n`, `iptables -S`, `nft list ruleset`
- パケット：`tcpdump -i eth0 port 22`
- カーネル：`sysctl net.ipv4.conf.all.rp_filter`

## 14. 付録B：初動トラブル対応テンプレ（コピペ用）

```bash
# 1) サービス/ポート
systemctl status ssh || systemctl start ssh
ss -tlnp | grep :22

# 2) 外壁・内壁
# （※ConoHaのセキュリティグループで 22/TCP(IPv4) を許可・アタッチ）
ufw status verbose

# 3) DNS（symlinkなら破壊→固定）
ls -l /etc/resolv.conf
# …必要なら stop/disable → rm → 再作成 → chattr +i

# 4) ルート
ip route
# default なければ:
ip route add default via <gateway> dev eth0

# 5) 監視
tail -f /var/log/auth.log &
tcpdump -i eth0 port 22
```

## 15. おわりに

- 今回潰した罠と得た知見を一言で。
- ※この先はAWSで運用を組み直したが、本稿では割愛（ここまでがConoHa/VPSの教訓）。

---

# メタ（執筆メモ）

- スクショ差し込み位置：各セクション末に「※ここに〇〇のスクショ」コメントを入れておくと編集が楽。
- コードブロックは**すべてコピペで動く形**に統一。
- トーンは「失敗談→原因→対処→再発防止」に揃えると読みやすい。
