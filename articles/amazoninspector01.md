---
title: "【Amazon Inspector】対応パターン集"
emoji: "⚠️"
type: "tech"
topics: ["AWS", "Docker", "Linux", "セキュリティ"]
published: false
---

すべての「Amazon Inspectorの警告は絶対に消せ」という上司を持った方へ<br>
大変お辛い時間を過ごされたことかと思います。<br>
解決パターン集を用意しましたのでご一読ください。

※本記事は個人検証環境を元に構成されています。実際の業務内容や企業の設定とは無関係です。

### A. package.jsonいじれば治るパターン

一番オーソドックスなパターン<br>

- ```npm outdated```で該当パッケージ調査
- ```npm list```で依存関係チェック
- 治らなければ```overrides```に追記してバージョンを強制上書き

### B. OSをアップデートすれば治るパターン
Dockerfileにコマンドを記述してからデプロイ

```RUN apt-get update && apt-get upgrade -y```

::: message
```-y```オプションを外すと更新が止まるので外さないように
:::

Dockerfileにnpm系コマンドを書いている場合はそれよりも前に書く

### C. FWが古いのでソースごと改修するパターン
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
   

### D. そんなものはない　震えて眠れ
  パターンDebian。さっさと修正パッチ出せ

   ......一応パターンBに```apt list --upgradable```を差し込んであげる方法がある

   ```RUN apt-get update && apt list --upgradable && apt-get upgrade -y```

   これで解決するといいね


### E. ウラワザ的回避
  本番環境に上がることでInspectorが怒りだすので、<br>
  ```--omit=dev```オプションを使用してパッケージを開発環境に封印する

  こうすることでCode Buildは```dependencies```のみを見て```npm install```する (```devDependencies```は見ない)

  ```eslint```などに効果的

### 総括
Amazon Inspectorは、まるで「自分でやれ」と言わんばかりに通知だけ送ってきます。<br>
でも、それを全部真に受けて修正してたら、こっちのサービスが死にます。

本記事の内容は、実際にぶつかった地雷を元に「現実的に対応した」記録です。
