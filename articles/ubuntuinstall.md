---
title: "【Ubuntu】中古PCにUbuntuをインストールする"
emoji: "⚛️"
type: "tech"
topics: ["Ubuntu"]
published: false
---

## 使用機材
東芝 Dynabook 第4世代 Intel Core i3
KIOXIA USBメモリ 32GB(推奨:4GB以上)

## 実装手順

### 1.Ubuntu Serverをインストール
①[ubuntuダウンロードページ](https://www.ubuntulinux.jp/home)にアクセスし、「Ubuntuのダウンロード」を選択する
![](https://storage.googleapis.com/zenn-user-upload/c468aed989de-20240825.png)

② 「日本語Remixイメージのダウンロード」を選択する
![](https://storage.googleapis.com/zenn-user-upload/846b2994231b-20240825.png)
日本語Remixイメージだと、日本語で快適に使用できるように加工されている。
オリジナルバージョンでも問題ない。

③ミラーサイトを選択する
![](https://storage.googleapis.com/zenn-user-upload/1d2dc7d56856-20240825.png)
「富山大学」と「株式会社アプセル」はhttp接続なので注意
「株式会社アプセル」を選択するとダウンロードに23時~翌2時までかかった

④ubuntu-ja-22.04-desktop-amd64.iso(3.2GBのもの)を選択し、ダウンロード開始
![](https://storage.googleapis.com/zenn-user-upload/e8df71a2f1fd-20240825.png)

⑤[rufusダウンロードページ](https://rufus.ie/ja/)にアクセスする
Portableタイプのものを選択してダウンロード
![](https://storage.googleapis.com/zenn-user-upload/50f4a8cd8975-20240825.png)
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

ブータブルUSBが認識されていることを確認し、選択する
(この画面では矢印キーしか使用できないので注意)
![](https://storage.googleapis.com/zenn-user-upload/b8f4cd03f72b-20240824.jpg)
※BIOSモードで起動する場合は他ブロガー様の記事を参照してください

しばらく待つと黒い選択画面(GRUBメニュー)が出てくる
選択は2回発生するが、両方とも一番上を選択する
矢印キーしか使用できないので注意

選択1回目「Ubuntu」or「Advanced Options For Ubuntu」⇒「Ubuntu」を選択
※画像を失念しました。申し訳ございません

②Ubuntuのデスクトップ画面が映るので、左枠の言語選択で日本語を選択し、「Ubuntuを試す」を押す
![](https://storage.googleapis.com/zenn-user-upload/d7e2a51605b1-20240825.jpg)

③デスクトップの「Ubuntuをインストール」をダブルクリック
![](https://storage.googleapis.com/zenn-user-upload/fc1514b7f74c-20240825.jpg)

④日本語が選択されていることを確認し、「続ける」を押す
![](https://storage.googleapis.com/zenn-user-upload/d9b2b1fd6791-20240825.jpg)

⑤キーボードレイアウトで日本語/日本語を選択する
![](https://storage.googleapis.com/zenn-user-upload/31daecf92ca2-20240825.jpg)

⑥無線LANネットワーク選択画面で、接続するネットワークを選択し、「接続」を押す
![](https://storage.googleapis.com/zenn-user-upload/6be78a9a9c9d-20240825.jpg)

⑦Wi-Fiのパスワードを入力し、「接続」を押す
![](https://storage.googleapis.com/zenn-user-upload/0da4bc9d7f23-20240825.jpg)

⑧「通常のインストール」と「最小のインストール」はどちらを選択してもOK
 Ubuntuを普段のWindowsのように使いたい場合は「通常」を選択
 筆者はCLIのみ使いたいので、インストールが速い「最小」を選択しました。
 「Ubuntuのインストール中にアップデートをダウンロードする」は時短になるので必ずチェックをいれる
![](https://storage.googleapis.com/zenn-user-upload/0d4136085e19-20240825.jpg)

 ⑨「UbuntuをWindowsと別にインストールする」を選択すると起動時にUbuntuとWindowsを選択できるらしい
 Ubuntuのみを使用する予定なら「ディスクを削除してUbuntuをインストール」を選択する
![](https://storage.googleapis.com/zenn-user-upload/d774e3b852cb-20240825.jpg)

⑩「続ける」を押す
![](https://storage.googleapis.com/zenn-user-upload/bd14dd566a55-20240825.jpg)

⑪タイムゾーンが東京であることを確認し、「続ける」を押す
![](https://storage.googleapis.com/zenn-user-upload/7b84f79b4c7e-20240825.jpg)

⑫名前とログイン用のパスワードを入力する
「続ける」を押すとインストールが始まる
![](https://storage.googleapis.com/zenn-user-upload/ba028f9a8ae1-20240825.jpg)

⑬インストールが完了したら「試用を続ける」を押す
![](https://storage.googleapis.com/zenn-user-upload/f221f64a0d78-20240825.jpg)

⑭右上のWi-Fiや電源マークを押すとメニューが出るので、電源オフ/ログアウトをクリックする
![](https://storage.googleapis.com/zenn-user-upload/30e5d5cee960-20240825.jpg)

⑮電源オフをクリックする
![](https://storage.googleapis.com/zenn-user-upload/6a82e2b8979b-20240825.jpg)

シャットダウン完了後、起動してUbuntuデスクトップが出たらセットアップ完了

## 今後の展望
- UbuntuでインストールしたGUIを削除する
- Ubuntuで自宅サーバーをセットアップする