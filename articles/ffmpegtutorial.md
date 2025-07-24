---
title: "Linux上で動画→GIF変換を最適化した話：Adobeからffmpegへ"
emoji: "🎬"
type: "tech"
topics: ["linux", "wsl", "ubuntu", "debian"]
published: false
---

## ffmpegコマンドについて解説

### 0.はじめに

先日Flaskアプリについての記事を執筆した際、Adobeのフリーツールでmp4をgifに変換する作業を行った。
@[card](https://zenn.dev/nickelth/articles/outputreportpy)

満足いく変換ができず複数回試したところで制限が来てしまい、使用できなくなった。
履歴やCookie削除を行い、もう１度試したが利用できなかった。

その後、`ffmpeg`コマンドの存在を知り利用。うまくgif変換ができた。

### 1. ffmpegのインストール

以下記事で作成したLinuxベースの開発環境でffmpegを活用した。
@[card](https://zenn.dev/nickelth/articles/ubuntuenvsetup)


- クラウド上での処理も考慮して軽量なGIFを生成

- CI/CDパイプラインに組み込む際の注意点（って書くとプロっぽい）

Zennに今すぐ載せる用(変換したい動画の名前を`input.mp4`にする)
```bash:高画質版
ffmpeg -i input.mp4 -vf "fps=15,scale=800:-1:flags=lanczos,palettegen" -y palette.png
ffmpeg -i input.mp4 -i palette.png -filter_complex "fps=15,scale=800:-1:flags=lanczos[x];[x][1:v]paletteuse" -y output.gif
```
`mp4→png→gif`というように2段階工程(パレット生成＆適用方式)にすることで高画質を実現している。
`flags=lanczos,palettegen`のフラグを立てて高画質を担保している(らしい)


```bash:軽量版
ffmpeg -i input.mp4 -vf "fps=15,scale=800:-1:flags=lanczos" output.gif
```
`scale`で画面縦幅(px)を調整できる。縦横比維持で横幅は自動調整される。

### 2. おわりに
オフライン環境で変換ツールが使用できる`Linux`はまだまだ奥が深いと感じた。Adobeにアップロードしないのでセキュリティにも優しい。他の変換ツール(pdf→pngなど)があれば自宅サーバ建ててオフライン変換というアイデアも実現できそう

### 3.参考文献
@[card](https://namileriblog.com/mac/ffmpeg)
@[card](https://qiita.com/saka212/items/fae883a4030857f41c1c)