---
title: "Javaレガシー帳票出力アプリをPythonにリメイクする"
emoji: "📄"
type: "tech"
topics: ["flask", "postgres", "python", "json"]
published: false
---

## はじめに
昔SIerにて、以下の構築の案件に関わったことがある。

JavaScript UI
Java (Service Layer)
Java (Repository Layer)
XML(MySQL)
DB

JVMが非常に重いことが多々あったのを思い出し、モダン構築で実現できないか考察した。



### 1. Python開発環境構築
以下の記事で Python + Postgres + Apache2 の開発環境を作成済み
![](https://zenn.dev/nickelth/articles/ubuntuenvsetup)

また、WSL上でVSCodeを起動できるようにしている。

### 2. Flask, jinja2インストール