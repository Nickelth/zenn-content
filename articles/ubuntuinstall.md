---
title: "【Ubuntu】中古PCにUbuntuをインストールする"
emoji: "⚛️"
type: "tech"
topics: ["Ubuntu"]
published: false
---

## 動機


## 使用機材
東芝 Dynabook 第4世代 Intel Core i3
KIOXIA USBメモリ 32GB(推奨:4GB以上)

## 実装手順

### 1.Ubuntu Serverをインストール
①[ubuntuダウンロードページ](https://www.ubuntulinux.jp/home)にアクセスし、「Ubuntuのダウンロード」を選択する
![](https://storage.googleapis.com/zenn-user-upload/375412e18ad7-20240824.png)

② 「日本語Remixイメージのダウンロード」を選択する
![](https://storage.googleapis.com/zenn-user-upload/71dc7e99cc6c-20240824.png)
日本語Remixイメージだと、日本語で快適に使用できるように加工されている。
オリジナルバージョンでも問題ない。

③ミラーサイトを選択する
![](https://storage.googleapis.com/zenn-user-upload/5be525bfb8f2-20240824.png)
「富山大学」と「株式会社アプセル」はhttp接続なので注意
「株式会社アプセル」を選択するとダウンロードに23時~翌2時までかかった

④ubuntu-ja-22.04-desktop-amd64.iso(3.2GBのもの)を選択し、ダウンロード開始
![](https://storage.googleapis.com/zenn-user-upload/fd058b54af17-20240824.png)

⑤[rufusダウンロードページ](https://rufus.ie/ja/)にアクセスする
Portableタイプのものを選択してダウンロード
![](https://storage.googleapis.com/zenn-user-upload/fb1b127ebd9b-20240824.png)
Portable版だとインストール不要なのでおすすめ

### 2.ブータブルUSBの作成
ダウンロードしたrufusのexeファイルをダブルクリック
自動更新について質問されるが、どちらでも構わない

①USBメモリ(4GB以上)が認識されていることを確認し、「選択ボタン」押下
 ダウンロードしたUbuntu ISOイメージを選択する
![](https://storage.googleapis.com/zenn-user-upload/1e5508e8d081-20240824.png)

②選択したら、「スタート」ボタンを押下
![](https://storage.googleapis.com/zenn-user-upload/3151e12ff014-20240824.png)

③「ISOHybridイメージ」が検出されるので、推奨されている「ISOイメージモードで書き込む」を選択し、OKボタンを押す
![](https://storage.googleapis.com/zenn-user-upload/b3da422a0824-20240824.png)

④Ubuntuとrufusの間でGRUBのバージョンが異なる場合、以下のメッセージが出る。
 「はい」を選択して適合するバージョンをダウンロードする
![](https://storage.googleapis.com/zenn-user-upload/bfe420a8728b-20240824.png)

⑤USBデバイスのデータを完全消去する警告が出るので、OKを押す
![](https://storage.googleapis.com/zenn-user-upload/14abb0310fa7-20240824.png)

⑥複数のパーティションを含む場合は以下の警告が出る。
 上書きしても問題ないので、OKを押す
![](https://storage.googleapis.com/zenn-user-upload/cc698b64818f-20240824.png)

⑦USBへの書き込みが完了すると、緑色のバーに「準備完了」と表示される
![](https://storage.googleapis.com/zenn-user-upload/e88f70eaeadb-20240824.png)
「閉じる」ボタンを押してOK

今回の使用容量は3.19GBだった。
![](https://storage.googleapis.com/zenn-user-upload/287b7525f717-20240824.png)

以上①~⑦の操作でブータブルUSBが完成

### 3.中古PCにUbuntu Serverをインストール

①BIOSモードまたはUEFIモードを起動する
東芝Dynabookの場合はShift+シャットダウンで完全シャットダウン後、F12を連打するとUEFIモードが実行できた

ブータブルUSBが認識されていることを確認し、選択する(この画面では矢印キーしか使用できないので注意)
![](https://storage.googleapis.com/zenn-user-upload/b8f4cd03f72b-20240824.jpg)
※BIOSモードで起動する場合は他ブロガー様の記事を参照してください

しばらく待つと黒い選択画面(GRUBメニュー)が出てくる
選択は2回発生するが、両方とも一番上を選択する
矢印キーしか使用できないので注意

選択1回目「Ubuntu」or「Advanced Options For Ubuntu」⇒「Ubuntu」を選択
※画像を失念しました。申し訳ございません


