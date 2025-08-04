---
title: "【#1】Javaレガシー帳票出力アプリをPythonにリメイクした話"
emoji: "📄"
type: "tech"
topics: ["flask", "postgres", "python", "json"]
published: true
---

## Flask帳票出力アプリ

### 0. はじめに

昔SIerにて、以下の構築の案件に関わったことがある。

![](https://storage.googleapis.com/zenn-user-upload/5ed119463edb-20250722.png)

この帳票出力アプリの構築では、
ユーザがJavaScript UIの画面に入力した値をバックエンド側で処理し,
生成されたSQLでDBに問い合わせ、結果をもとにPDFを生成する。

しかし、帳票出力処理とバックエンド処理が密結合なため動作が重く、
保守性も拡張性も絶望的であった。
フロントとバックが密に絡んでて「改修したくても壊れる」状態であった。

これらの経験から、軽量でモダンな構築で実現できないか考察した。

---

### 1. Flaskで作り直したときの構成メモ

以下の記事で Python + Postgres + Apache2 の開発環境を作成済み(この構築ではApacheは未使用)
@[card](https://zenn.dev/nickelth/articles/ubuntuenvsetup)

また、WSL上でVSCodeを起動できるようにしている。

![](https://storage.googleapis.com/zenn-user-upload/7a496e42cdd7-20250724.png)
- アプリ側は　Python + Flask + Jinja2 + PostgreSQL（※ORMはSQLAlchemyでもいいし、生でもいい）
- 帳票生成ロジックは　入力HTML → Flask（処理&テンプレート）→ HTML（Jinja2） → weasyprint → PDF

``` markdown:ディレクトリ構成
.
├── Dockerfile
├── README.md
├── app.py               --アプリ実行ファイル
├── docker-compose.yml
├── init.sql             --DB初期化用SQL
├── requirements.txt     --Pythonパッケージの依存関係記録
├── static               
│   └── style.css
└── templates
    ├── index.html
    ├── layout.html
    └── report.html
```
- アプリ実行部は`app.py`のみのシンプル実装。
- `layout.html`を繰返しのベースとし、画面部は`index.html`、帳票部は`report.html`に分けて責務を明確化。
- バックエンド主軸のため、CSSなどUI部分は最小限の構成にとどめる。レスポンシブ対応も省略。

---

### 2. 実装内容の詳細

#### 2.1 バックエンドロジックの分離とPDF出力

- 入力補助（オートコンプリート）の際、バックエンドからキーと値の`JSON`形式で返却
  - 保守性・拡張性が高い形式であり、Flaskの`jsonify()`とも好相性。

- 保守性のための画面部と帳票部の分離構造を実現


#### 2.2 フロント側の実装ロジック

- 商品コード、商品名が入力されたときに、DBをクエリして問合せ結果が返る場合、商品コード、商品名、単価（円）の情報をオートコンプリートする。
- 商品コード、商品名、数量、単価は入力必須とし、不正値が入力された状態で「納品リストに追加」ボタンが押されるとエラーが出るようにHTML実装。
- スマホ対応は今回の主旨から外れるため省略

---

### 3. 入力画面＋PDF出力結果

![帳票出力画面.gif](https://storage.googleapis.com/zenn-user-upload/9d9814f572f3-20250724.gif)
![納品書.png](https://storage.googleapis.com/zenn-user-upload/e0d08375a742-20250724.png)


---

### 4. おわりに

今回は1画面のみの実装であるが、「保守しやすさ」「分かりやすい責務分離」を意識してバックエンド構築ができた。PDF帳票のようなシンプルな機能でも、裏側の処理整理と役割分離は実務で重要になる。CI/CDやクラウド連携も視野に見据えて、「見た目より構造で勝負する」設計姿勢を残しておきたい。

### 次章 - app.py分割 & Auth0連携
@[card](https://zenn.dev/nickelth/articles/reportapppart02auth0)