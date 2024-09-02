---
title: "【WireGuard】「デバイスの使用準備ができていません。」(Code 0x000010DF)の解決方法"
emoji: "🔠"
type: "tech"
topics: ["WireGuard"]
published: false
---
### エラー内容
Windows10のWireGuardアプリで「有効化」ボタンを押すとエラー
``` bash
 [TUN] [client2] Failed to setup adapter (problem code: 0x38, ntstatus: 0x0): デバイスの使用準備ができていません。 (Code 0x000010DF)
```

### 原因
「デバイスマネージャー」内の「デバイスアダプター」に「TAP-Windows Adapter」や「WireGuard Adapter」などのVPNアダプターがインストールされていない

### 解決方法
A. WireGuardを再インストールする。このとき一緒に「WireGuard Adapter」もインストールされる。
    ⇒筆者の環境ではインストールされなかった。

B. [OpenVPN](https://openvpn.net/community-downloads/)をインストールする。このとき一緒に「TAP-Windows Adapter」もインストールされる。
    ⇒インストール成功

### 結果
エラーが解決され、WireGuardが有効化できるようになった。