---
title: "VPS業者を選定し、アプリを配置して限定公開する話"
emoji: "☁"
type: "tech"
topics: ["Ubuntu", "cli", "bash"]
published: false
---

### VPS業者を選ぶ
- 例：ConoHa、さくらVPS、Amazon Lightsail、Linode、Vultr、etc.
    → 正直、ポートフォリオ用途なら安いやつでOK。ConoHaあたりはUIも日本語でわかりやすい。

- プランを選ぶ
    → 最安のやつで問題ない。512MB RAMとか1GBのやつでOK。
        月数百円～数千円。カップラーメン2〜3個我慢すれば払える。

- OSを選ぶ
    → だいたいUbuntuを選んどけば人類の9割と話が通じる。

- VPSを起動すると、IPアドレスが振られる
    → 「あなたの厨房はここにあります」と住所がもらえるイメージ。

- SSHで接続（遠隔ログイン）
    → ターミナルで ssh user@IPアドレス とか打って、ラーメン厨房に入る。

- アプリを配置する
    → ここが一番ラーメン職人ぽい。Node.jsアプリならNode.jsを入れたり、ポート開けたり、pm2とかで動かしたり。