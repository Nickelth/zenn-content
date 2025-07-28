---
title: "ConoHa VPSを契約したらそもそもSSH接続できなくて泣いた"
emoji: "☁"
type: "tech"
topics: ["Ubuntu", "cli", "bash", "ssh"]
published: false
---

## VPSはちゃんと課金しましょう

### 0. はじめに

Flaskアプリで本番環境を構築中にVPSの設置が必要になった。
記事の構成要素が増加したため、その部分だけ独立して記事を執筆する。



![](https://storage.googleapis.com/zenn-user-upload/e1a701a3b638-20250727.png)
![](https://storage.googleapis.com/zenn-user-upload/9d45585892e0-20250727.png)
「毎時1.9円」「シャットダウン中は課金されない」「UIが日本語」という魅力的な内容だったため契約して本番環境にしようとした結果

- 「ConoHa」で検索した結果、VPSではなく「WING」という全く別のものを契約してしまった
- VPSを契約後SSH接続しようとしたがなぜかPingすら通らない
- VPS側のターミナルはブラウザ版なので文字入力直接コピペできなくて使いづらすぎる
- そのターミナルで調べた結果Pingは受け取っているが謎の設定で破棄されていることが判明

おそらく低価格を維持するためセキュリティを強固にして人件費を削減しているのかと勘ぐってみたり
いずれにしろ日曜日が無駄にされたので他社に乗り換え


### 1. VPSの申し込み



### 2.プラン選択
![](https://storage.googleapis.com/zenn-user-upload/034ca6447ab7-20250727.png)
*最安プランでも性能が破格*
今回の用途はポートフォリオ用に一時公開する程度なので、従属課金制のものが望ましい。
イメージタイプ：Ubuntu24.04
1.9円毎時のものを選択

rootパスワードとネームタグを設定

「追加する」を押してしばらくするとステータスが「構築中」→「起動中」に変わる
![](https://storage.googleapis.com/zenn-user-upload/aadc38c409b3-20250727.png)

このまま放置すると課金されるので、シャットダウン操作をする。
チェックボックスにチェックを入れて、「一括操作」と書かれたドロップダウンリストをクリックし、シャットダウンを選択。
ポップアップが出るので「はい」を押すとシャットダウンが開始される。


### 3. VPS操作

現状のrootパスワードのみ設定されている状態だとブルートフォース攻撃で突破されるので、SSH鍵を設定する。

```bash
ssh root@YOUR_IP_ADDRESS
```
*IPアドレスはログインしたいVPSの設定値を参照*


```bash:作業用一般ユーザ追加。以降、rootログインを禁止にする。
adduser deploy
```


```bash:ssh鍵を作成
ssh-keygen -t ed25519 -C "your@email.com"
```

```bash:公開鍵を表示してコピー
cat ~/.ssh/id_ed25519.pub
```

コピーした`.pub`を`~/.ssh/authorized_keys`に貼り付ける。
```bash
mkdir -p ~/.ssh
nano ~/.ssh/authorized_keys
# → ここに公開鍵ペーストして保存

chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

これで次回からはSSH認証で入れるようになる
```bash
ssh -i ~/.ssh/id_ed25519 deploy@YOUR_IP_ADDRESS
```

次はrootユーザのログイン禁止設定を行う。
一般ユーザ(deploy)でログインできることはここまで操作で確認済み。
nanoエディタで`sshd_config`を書き換え
```bash
sudo nano /etc/ssh/sshd_config
```
36行目付近に以下の記述があるので、noに変更する
```conf
PermitRootLogin no
```

`ssh`プロセスを再起動して設定を反映させる
```bash
systemctl restart sshd
```

rootユーザでのログインを試してみる
```bash
ssh -i ~/.ssh/id_ed25519 root@YOUR_IP_ADDRESS
```

`Permission denied, please try again`のように表示されると成功

VPSなので作業終了時のさっとダウンを行う
```bash
ssh -i ~/.ssh/id_ed25519 deploy@YOUR_IP_ADDRESS
sudo poweroff
```
:::message
VPSの再起動時はGUIで操作する必要がある。
:::

- VPSの登録
- SSH認証の設定
- `root`ログインの禁止
- VPSのシャットダウン

以上の操作が完了した。
おつかれさまでした。


### 4. おわりに
引き続き`Github Actions + Docker`でのCI/CD環境を構築を実施する予定。

Flaskアプリシリーズ
@[card](https://zenn.dev/nickelth/articles/auth0application)
@[card](https://zenn.dev/nickelth/articles/outputreportpy)