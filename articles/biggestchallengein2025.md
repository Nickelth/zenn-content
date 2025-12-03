---
title: "AWSアーキテクチャ作成振り返り"
emoji: "💭"
type: "idea"
topics: ["aws", "terraform", "ecs", "rds", "githubactions"]
published: true
---

## 帳票出力アプリ

@[card](https://github.com/Nickelth/papyrus-invoice)

### 目的 / Purpose

過去に業務で携わった「帳票出力アプリ」について、
資材モダナイズ+継続保守・監視を目標にリメイクした。

![帳票出力画面.gif](https://storage.googleapis.com/zenn-user-upload/9d9814f572f3-20250724.gif)
*動作イメージ*

### 背景 / Context

今年7月にZennアカウントを新規作成。
Zennにお試しで記事投稿をしつつ、
AWSを使用したポートフォリオ用アプリケーションの構想を開始。
諸々の難所があったが、今月11月に完成扱いとした。
投稿記事が若干あおり気味・手順書チックになったので
ブックでは報告書寄りになるように心掛けた。
今後投稿する記事も報告書形式にする予定である。

### 困難だった箇所 / Difficulties

- IAMロール、ポリシーとOIDCの連携に苦慮
- 「後追いTerraform」実装によるアプリ全体のIaC移行の難航
- 監査内容の膨大化とCI実装
- 完成定義の設定

### 拡張可能性 / Expandable

- ALBが高価である都合上、疎通テストはTerraformによる一時ALB作成→疎通→破壊にとどめた。
    本PJではAWSアーキテクチャ構築・監査をメインとしているため、
    フロントエンドの本番環境でアプリ起動・動作確認は割愛した。

### 関連記事 / Articles

@[card](https://zenn.dev/nickelth/books/papyrus-book)
@[card](https://zenn.dev/nickelth/articles/reportapp03docker)
@[card](https://zenn.dev/nickelth/articles/reportapp02auth0)
@[card](https://zenn.dev/nickelth/articles/reportapp01flask)

