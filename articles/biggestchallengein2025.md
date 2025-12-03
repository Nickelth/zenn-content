---
title: "AWSアーキテクチャ作成振り返り"
emoji: "⌨"
type: "idea"
topics: ["aws", "terraform", "ecs", "rds", "s3"]
published: false
---

## 帳票出力アプリ

### 目的 / Purpose

過去に業務で携わった「帳票出力アプリ」について、
資材モダナイズ+継続保守・監視を目標にリメイクした。

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

- ALBが高価である都合上、本番環境でアプリ起動・動作確認は実行しなかった。経済的余裕が出たのちに実践してみたい。

### 関連記事 / Articles

@[card](https://zenn.dev/books/papyrus-book)
@[card](https://zenn.dev/articles/reportapp03docker)
@[card](https://zenn.dev/articles/reportapp02auth0)
@[card](https://zenn.dev/articles/reportapp01flask)

