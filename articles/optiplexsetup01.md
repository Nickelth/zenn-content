---
title: "中古PCを舐めプしてたらネジ1本に作業を止められた件"
emoji: "🔩"
type: "tech"
topics: ["Linux"]
published: true
---

※当記事を閲覧して被った損害に関しては一切の責任は負いかねます。

## 中古購入前に気を付けるべきポイント

### 0. はじめに

いつまでもWSLだと今後戦えないので、お盆中にLinuxを攻略し尽くそうと思い切って購入

ノートPCでUSBブートした経験はあるので軽いノリでやったらトラブル頻発

・SSDつけようとしたら**M.2ネジが必要**だと分かり作業中断
・電源プラグが**欧州製**だったため並行輸入品であったことを購入後に知った（日本版を後から購入）
・**Displayport**という謎規格出力しか穴が開いてなかったのでHDMI変換器を別途購入した
・その変換器も**アクティブとパッシブの２種類**あり誤購入（のちに返品）

### 1. スペックと費用

|パーツ|価格|備考|
|---|---|---|
|Optiplex 7070|¥28,600|9世代Intel Corei7, 電源プラグが欧州, SSDなし|
|Crucial 1TB SSD NVMe4|¥6,900|ポイント使用後|
|BENFEI DP(オス)-HDMI(メス)変換器|¥1,200|アクティブじゃないと画面が映らない|
|電源プラグ|¥2,100|日本規格|
|合計|¥38,800|高い|

というわけで3万未満でお得に購入のはずが気づけば4万付近
しかも動作時がなんかうるさい

### 2. 画像付きセットアップ手順

背面に青色のレバーがあるのでスライド

開けた状態
![](https://storage.googleapis.com/zenn-user-upload/263730c2bd46-20250808.jpg)

HDDのロックを解除する
![](https://storage.googleapis.com/zenn-user-upload/72238ced4ce9-20250808.png)

慎重に外す。プラグが戻せるなら外したほうがいい
![](https://storage.googleapis.com/zenn-user-upload/186a7f3a5674-20250808.jpg)

SSD挿込口が見えている状態
![](https://storage.googleapis.com/zenn-user-upload/98e7bd3bb161-20250808.jpg)

SSDを挿す。斜め45度に挿してカチッと音がするまで押し込む

M.2ネジと元々合ったネジでSSDを挟むようにしてM.2ネジを挿す
![](https://storage.googleapis.com/zenn-user-upload/0bd15f3835ea-20250808.jpg)

SSDを挿し終わったのでPCを組み立て戻す

#### 参考にした動画
@[card](https://youtu.be/zcmY4Bcja1Y?si=OSNkqTr3yqYFZV-m)

### 3. 感想：爆音の原因について
ハードウェア知識が全くない状態で中古PCデビュー。
思ったより地雷部分が多くて想定よりセットアップが長引いた。
USBブートできたものの、画面サイズがおかしかったり日本語キーボードがデフォルトで利用できなかったりソフト面でもトラブルが出たので、こちらも記録する予定。

爆音の原因は不明。

### 4. 中古PCシリーズ
@[card](https://zenn.dev/nickelth/articles/optiplexsetup02mint)
@[card](https://zenn.dev/nickelth/articles/optiplexsetup03rmhdd)