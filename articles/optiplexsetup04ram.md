---
title: "中古PCのRAMを増設して32GBにする"
emoji: "💾"
type: "tech"
topics: ["Linux", "bash", "cli"]
published: false
---

## 8GB×2本を追加する

### 0. はじめに

中古PCを購入して3週間。
セットアップやブラウジング、Dockerで開発環境移植などを一通り完了。
ここまではWindowsでもできるのでそろそろ中古PC+Linuxでないとできないことを始めようと思う。

まあ機械学習ですね。

昨今はAIの能力が急激にインフレしているのは言うまでもないが、インフラ・AWSをからめた運用の練習がしたいと思いPC性能アップを決断。
本当はGPU買って個人でゴリゴリ学習させたいけど、そこまでの金はないし。

### 1. メーカー/型番を確認

32GBを最大限活用するには、8GB×4本すべてでメーカー/型番をそろえないといけない。
もしメーカー/型番が適当だと、

- 違うメーカー混在 → だいたい動くけど、最悪片方だけ認識とかブルスクのリスクあり
- 速度は結局 遅い方に引っ張られる

というリスクがある。

確認自体は簡単。フタをあけても駆使するだけ。

![RAMの型番確認](https://storage.googleapis.com/zenn-user-upload/0be1f2b0f8ab-20250823.png)

というわけでSamsungでした。

### 2. 価格ドットコムで最安値を検索

ここはお祈りフェーズ。
いい感じに手ごろなものが見つかってほしい。
検索欄に型番をぶっ込む。

![価格ドットコム](https://storage.googleapis.com/zenn-user-upload/f53b5b5f1aca-20250823.png)

¥4,374/本
悪くはない。

![楽天市場にて購入](https://storage.googleapis.com/zenn-user-upload/8fc0f9f64323-20250823.png)

楽天市場の場合はちゃんと0か5のつく日に購入してお得に購入しましょう。
どうせフタあけて差し込むだけなんで平日夜でもそんなに時間は取りません。

### 3. RAMを本体基盤に挿入



### 4. 中古PCシリーズ

@[card](https://zenn.dev/nickelth/articles/optiplexsetup01)
@[card](https://zenn.dev/nickelth/articles/optiplexsetup02mint)
@[card](https://zenn.dev/nickelth/articles/optiplexsetup03rmhdd)