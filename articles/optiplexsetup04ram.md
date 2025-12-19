---
title: "中古PCのRAMを増設して32GBにする"
emoji: "💾"
type: "idea"
topics: ["pc"]
published: false
---

## 8GB×2本を追加する

### 1. メーカー/型番を確認

32GBを最大限活用するには、8GB×4本すべてでメーカー/型番をそろえないといけない。
もしメーカー/型番が適当だと、

- 違うメーカー混在 → だいたい動くけど、最悪片方だけ認識とかブルースクリーンのリスクあり
- 速度は結局 遅い方に引っ張られる

というリスクがある。

確認自体は簡単。

![RAMの型番確認](https://storage.googleapis.com/zenn-user-upload/0be1f2b0f8ab-20250823.png)

というわけでSamsungでした。

### 2. 価格ドットコムで最安値を検索

ここはお祈りフェーズ。
いい感じに手ごろなものが見つかってほしい。
検索欄に型番をぶっ込む。

![価格ドットコム](https://storage.googleapis.com/zenn-user-upload/f53b5b5f1aca-20250823.png)

¥4,374/本
悪くはない。

※2025年8月に購入

![楽天市場にて購入](https://storage.googleapis.com/zenn-user-upload/8fc0f9f64323-20250823.png)

楽天市場の場合はちゃんと0か5のつく日に購入してお得に購入しましょう。
どうせフタあけて差し込むだけなんで平日夜でもそんなに時間は取りません。

### 3. RAMを本体基盤に挿入

### 4. ここまでの全体費用

| パーツ                           | 価格    | 備考                                         |
| -------------------------------- | ------- | -------------------------------------------- |
| Optiplex 7070                    | ¥28,600 | 9世代Intel Corei7, 電源プラグが欧州, SSDなし |
| Crucial 1TB SSD NVMe4            | ¥6,900  | ポイント使用後                               |
| BENFEI DP(オス)-HDMI(メス)変換器 | ¥1,200  | アクティブじゃないと画面が映らない           |
| 電源プラグ                       | ¥2,100  | 日本規格                                     |
| RAM 8G x2                         | ¥8,748 | マザボが古いので差し込めるRAMの種類は限られる |
| 合計                             | ¥47,548 | 高い                                         |

### 5. 今後の展望

当初機械学習の教材として活用しようとしたが、途中で頓挫してしまった。
Windows11に移行できないノートPCが家庭内に存在するので、
OptiplexにProxmoxサーバーを立ててWin11VMをホスティングし、
リソースの有効活用・構築・運用の経験を積もうと考えている。

### 6. 中古PCシリーズ

@[card](https://zenn.dev/nickelth/articles/optiplexsetup01)
@[card](https://zenn.dev/nickelth/articles/optiplexsetup02mint)
@[card](https://zenn.dev/nickelth/articles/optiplexsetup03rmhdd)
