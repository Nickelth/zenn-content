---
title: "【Amazon Inspector】対応パターン集"
emoji: "⚠️"
type: "tech"
topics: ["AWS", "Docker", "Linux", "Debian", "npm", "セキュリティ"]
published: false
---

すべての「Amazon Inspectorの警告は絶対に消せ」という上司を持った方へ<br>
大変お辛い時間を過ごされたことかと思います。<br>
解決パターン集を用意しましたのでご一読ください。

※本記事は個人検証環境を元に構成されています。実際の業務内容や企業の設定とは無関係です。
※Debian前提です。Ubuntuは多分sudoつければ同様にできます。

### A. package.jsonいじれば治るパターン

一番オーソドックスなパターン<br>
![](https://storage.googleapis.com/zenn-user-upload/7dfed7ae1890-20250711.png)
- ```npm outdated```で該当パッケージ調査
- ```npm list```で依存関係チェック
- 治らなければ```overrides```に追記してバージョンを強制上書き

``` json: package.json
"dependencies": {
  "axios": "^1.8.2"
},
"devDependencies":{},
"overrides":{
  "axios": "^1.8.2"
}
```

### B. まぎらわしいパターン

執筆時点ではおそらく```cross-spawn```のみ該当<br>
![](https://storage.googleapis.com/zenn-user-upload/e94de380756b-20250711.png)

Aと同種かと思いきや**AWSのサーバーにプリインストールされているパッケージをスキャン**してそいつに対して怒っています。<br>
どこ見とんねん

::: message
目印はパス名が```/usr/local/```になっていること<br>
普通なら```/usr/src/```と表示される
:::

この場合はDockerfileにコマンドを記述してデプロイ走らせます。
``` bash: Dockerfile
RUN npm install npm@9 cross-spawn@7.0.5 -y
```
```npm@10```以降は```cross-spawn@7.0.5```以降(脆弱性クリア済み)とは依存関係がないのでこのような対応とする
::: message
```-y```オプションを外すと更新が止まるので外さないように
:::

### C. OSをアップデートすれば治るパターン
![](https://storage.googleapis.com/zenn-user-upload/b4a87d6e80c1-20250711.png)<br>
Bと同じくDockerfileにコマンドを記述してからデプロイ

``` bash: Dockerfile
RUN apt-get update && apt-get upgrade -y
```
-yは外しちゃダメ

::: message
```Dockerfile```に```npm```系コマンドを書いている場合はそれよりも前に書く
```apt```系→```npm```系
:::

### D. 対策？ そんなものはない　震えて眠れ
  パターンDebian。まさかの対策方法なし。さっさと修正パッチ出せ<br>
![](https://storage.googleapis.com/zenn-user-upload/a27e30e42889-20250711.png)

   ......一応パターンBに```apt list --upgradable```を差し込んであげる方法がある

   ``` bash: Dockerfile
   RUN apt-get update && apt list --upgradable && apt-get upgrade -y
   ```

   これで解決するといいね

### E. FWが古いのでソースごと改修するパターン
  Nest.jsやNode.jsなどが絡むと一気にややこしくなる

  特にNest.jsバージョン10を使ってる場合は要注意<br>
  - Nest.js@10依存パッケージに固める
  - Nest.js@11にアップデートする
  の2択を迫られます

  Nest.js@11にアップデートする場合は、
  npm公式が仕事してないのでそのまま```npm i```すると怒られます

  ```npm install --legacy-peer-deps```してあげると依存関係チェックを無視してくれるのでアップデートできます。

  若干ヤバい気がしますがnpmが悪いので問題なし<br>
  というかnpm公式がNest.js@11構成を推奨している

### F. ウラワザ的回避
  本番環境に上がることでInspectorが怒りだすので、<br>
  ```--omit=dev```オプションを使用してパッケージを開発環境に封印する

  ``` bash: Dockerfile
  RUN npm install --legacy-peer-deps --omit=dev
  ```

  こうすることでCode Buildは```dependencies```のみを見て```npm install```する (```devDependencies```は見ない)

  ```eslint```などに効果的

### 総括
Amazon Inspectorは、まるで「自分でやれ」と言わんばかりに通知だけ送ってきます。<br>
でも、それを全部真に受けて修正してたら、こっちのサービスが死にます。

本記事の内容は、実際にぶつかった地雷を元に「現実的に対応した」記録です。
