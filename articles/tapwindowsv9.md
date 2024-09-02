---
title: "【WireGuard】「デバイスの使用準備ができていません。」(Code 0x000010DF)の解決方法"
emoji: "🔌"
type: "tech"
topics: ["WireGuard"]
published: true
---
### エラー内容
Windows10のWireGuardアプリで「有効化」ボタンを押すとエラー
``` bash
 [TUN] [client2] Failed to setup adapter (problem code: 0x38, ntstatus: 0x0): デバイスの使用準備ができていません。 (Code 0x000010DF)
```

### 原因
「デバイスマネージャー」内の「デバイスアダプター」に「TAP-Windows Adapter」や「WireGuard Adapter」などのVPNアダプターがインストールされていない

コントロールパネルを開き、「ネットワークと共有センター」をクリック
![](https://storage.googleapis.com/zenn-user-upload/d10bf3d07f78-20240902.png)

左側に「アダプターの設定の変更」があるのでクリック
![](https://storage.googleapis.com/zenn-user-upload/32b9001c913a-20240902.png)

「ネットワーク接続」の中に「TAP-Windows Adapter」か「WireGuard Adapter」があればインストールされている
![](https://storage.googleapis.com/zenn-user-upload/f92b2efb5cea-20240902.png)

「デバイスマネージャー」を開いて「ネットワークアダプター」から確認する方法もある
![](https://storage.googleapis.com/zenn-user-upload/e5b3243fa3e8-20240902.png)

### 解決方法
A. WireGuardを再インストールする。このとき一緒に「WireGuard Adapter」もインストールされる
    ⇒筆者の環境ではインストールされなかった。

B. [OpenVPN](https://openvpn.net/community-downloads/)をインストールする。このとき一緒に「TAP-Windows Adapter」もインストールされる
    ⇒インストール成功

### 結果
エラーが解決され、WireGuardが有効化できるようになった

### 補足
WireGuardで何度も有効化繰り返すと、不要なネットワーク設定が溜まり「ネットワークと共有センター」を開く処理が重くなる場合がある
コマンドプロンプトで以下のコマンドを入力する
``` bash
netsh winsock reset
netsh int ip reset
ipconfig /release
ipconfig /renew
ipconfig /flushdns
```
ネットワークがリセットされ、「ネットワークと共有センター」にアクセスできるようになる