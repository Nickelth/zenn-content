---
title: "【WireGuard】VPNサーバの構築"
emoji: "⚛️"
type: "tech"
topics: ["WireGuard"]
published: false
---
WireGuardを使用して自宅サーバにVPNサーバを構築し、クライアントから接続する手順を以下に示します。セキュリティを重視した設定を行いながら、シンプルかつ効果的なVPNを構築します。

### 1. WireGuardのインストール

まず、自宅サーバにWireGuardをインストールします。

#### **Ubuntuサーバへのインストール**
```bash
sudo apt update
sudo apt install wireguard
```

### 2. キーペアの生成

WireGuardは、公開鍵と秘密鍵を使用して通信を暗号化します。サーバとクライアントそれぞれにキーペアを生成します。

#### **サーバのキー生成**
```bash
cd /etc/wireguard/
umask 077
wg genkey | tee server_private.key | wg pubkey > server_public.key
```

#### **クライアントのキー生成（個人PC用）**
個人PCでも同様にキーを生成する必要がありますが、ここではサーバで仮に生成し、後でクライアントに転送する方法を示します。
```bash
wg genkey | tee client_private.key | wg pubkey > client_public.key
```

### 3. WireGuardの設定ファイルを作成

次に、WireGuardの設定ファイルを作成します。

#### **サーバの設定ファイル**
`/etc/wireguard/wg0.conf`という名前で設定ファイルを作成します。

```bash
sudo nano /etc/wireguard/wg0.conf
```

以下を設定します。`<YOUR_SERVER_PUBLIC_IP>`はサーバの公開IPアドレスに置き換えてください。

```ini
[Interface]
PrivateKey = <server_private.keyの内容>
Address = 10.0.0.1/24
ListenPort = 51820

# NAT設定（インターネットアクセスをVPN経由で行う場合）
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
PublicKey = <client_public.keyの内容>
AllowedIPs = 10.0.0.2/32
```

- **PrivateKey**: サーバの秘密鍵 (`server_private.key`) を設定します。
- **Address**: サーバのVPN内でのIPアドレスを指定します（例: 10.0.0.1）。
- **ListenPort**: WireGuardがリッスンするポートです。デフォルトでは51820を使用します。
- **[Peer]**: クライアントの情報を設定します。`PublicKey`にはクライアントの公開鍵を設定し、`AllowedIPs`にはクライアントに割り当てるIPアドレスを指定します。

#### **クライアント（個人PC）の設定ファイル**
`client.conf`という名前で設定ファイルを作成します。

```ini
[Interface]
PrivateKey = <client_private.keyの内容>
Address = 10.0.0.2/24
DNS = 8.8.8.8

[Peer]
PublicKey = <server_public.keyの内容>
Endpoint = <YOUR_SERVER_PUBLIC_IP>:51820
AllowedIPs = 0.0.0.0/0, ::/0
PersistentKeepalive = 25
```

- **PrivateKey**: クライアントの秘密鍵 (`client_private.key`) を設定します。
- **Address**: クライアントに割り当てるVPN内のIPアドレス（例: 10.0.0.2）。
- **DNS**: 使用するDNSサーバのアドレスを指定します（例: 8.8.8.8はGoogle DNS）。
- **[Peer]**: サーバの情報を設定します。`PublicKey`にはサーバの公開鍵を設定し、`Endpoint`にはサーバの公開IPアドレスとポートを指定します。

### 4. サーバの起動と有効化

設定が完了したら、WireGuardを起動し、自動的に起動するように設定します。

```bash
sudo wg-quick up wg0
sudo systemctl enable wg-quick@wg0
```

### 5. ルータのポートフォワーディング設定

ルータの設定画面にアクセスし、WireGuardが使用するポート（51820/UDP）を自宅サーバのIPアドレスにフォワーディングします。

### 6. クライアントからの接続

クライアントPCで、WireGuardクライアントを使用してサーバに接続します。`client.conf`をWireGuardクライアントに読み込ませて、接続を確立します。

#### **Windowsでの接続手順**:

1. [WireGuardの公式サイト](https://www.wireguard.com/install/)からWindows用のクライアントをダウンロードしてインストールします。
2. `client.conf`をWireGuardクライアントにインポートします。
3. 接続を開始します。

### 7. 接続の確認

接続が成功すると、クライアントPCのIPアドレスがVPN内のIPアドレス（例: 10.0.0.2）に変わり、すべてのトラフィックがWireGuard経由でルーティングされるようになります。`whatismyip.com`などのサイトでIPアドレスがサーバの公開IPになっているかを確認してください。

### 8. セキュリティ強化のための推奨設定

- **SSHアクセス制限**: サーバのSSHアクセスをVPN接続のみ許可するように制限します。
- **ログ監視**: サーバで`journalctl`を使用してWireGuardの接続ログを監視します。
- **ファイアウォール設定**: 不要なポートやサービスをブロックし、VPN接続のみを許可するようにファイアウォールを設定します。

これで、自宅サーバ上にセキュリティが強化されたWireGuard VPNを構築し、クライアントから安全に接続できるようになります。