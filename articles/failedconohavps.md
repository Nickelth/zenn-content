---
title: "【負け】ConoHaVPSで本番環境立てようとしたら失敗して日曜日が溶けた話"
emoji: "🦄"
type: "tech"
topics: ["cli", "linux"]
published: false
---

## 

### 0. はじめに

Flaskアプリを公開するために本番環境が必要になった。
当初はVPSで実現する予定だったが、AWSで実施することにした。
価格相応のトラブルが出たことと、実務経験を活かしたいことが理由。

Flaskアプリで本番環境を構築中にVPSの設置が必要になった。
記事の構成要素が増加したため、その部分だけ独立して記事を執筆する。

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

### 1. 

![](https://storage.googleapis.com/zenn-user-upload/ee2d5819bc12-20250818.png)


![](https://storage.googleapis.com/zenn-user-upload/93b8c26916ac-20250818.png)



![](https://storage.googleapis.com/zenn-user-upload/ade100c010b4-20250818.png)


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





### 4. おわりに


Flaskアプリシリーズ
@[card](https://zenn.dev/nickelth/articles/reportapp02auth0)
@[card](https://zenn.dev/nickelth/articles/reportapp01flask)