---
title: "Adobeで動画→GIF変換したら画像が小さすぎたのでLinuxでやる"
emoji: "🎬"
type: "tech"
topics: ["linux", "cli", "bash"]
published: true
---

## ffmpegコマンドについて解説

### 0.はじめに

先日Flaskアプリについての記事を執筆した際、Adobeのフリーツールで`mp4`を`gif`に変換する作業を行った。
@[card](https://zenn.dev/nickelth/articles/reportapp01flask)

満足いく変換ができず複数回試したところで制限が来てしまい、使用できなくなった。
履歴やCookie削除を行い、もう１度試したが利用できなかった。

その後、`ffmpeg`コマンドの存在を知り利用。うまくgif変換ができた。

### 1. ffmpegコマンド使用方法
```bash
sudo apt install ffmpeg -y
```

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

### 2.補足:PDF関係の変換コマンド

```bash
# pdfをpngに変換する
pdfppm -png input.pdf output

# pdfの3~5ページ目をJPEGに変換する
pdftoppm -jpeg -f 3 -l 5 input.pdf output

# PNGをPDFに変換する
convert input.png output.pdf

# すべてのjpgを1枚のpdfに変換する
convert *.jpg output.pdf
```

`convert`を`PDF→PNG`に使用することもできるが、処理速度やページ指定などの便利機能の面で`pdfppm`に軍配が上がる。

### 3. おわりに
オフライン環境で変換ツールが使用できる`Linux`はまだまだ奥が深いと感じた。オンラインにアップロードしないのでセキュリティにも優しい。PDFが絡むものは使用頻度が高いので家族にもお勧めしたい。

### 4.参考文献
@[card](https://namileriblog.com/mac/ffmpeg)
@[card](https://qiita.com/saka212/items/fae883a4030857f41c1c)