---
title: "【#2】FlaskアプリにAuth0つけようとしたらapp.pyがぶっ壊れた話"
emoji: "⭐"
type: "tech"
topics: ["Auth0", "Python", "Flask"]
published: true
---

>Flaskは1ファイルで済むって言ったやつ出てこい

### 0. はじめに
本番環境に公開するためにAuth0入れてみたけどapp.pyが肥大化して詰んだ。
当初はAuth0をapp.pyに直接投入していたが、ルーティングぐちゃぐちゃになってページが開けなくなったので泣く泣く分割。
ハマった知見とかも共有しときます。
@[card](https://zenn.dev/nickelth/articles/reportapp01flask)

### 1. フル装備app.pyの地獄

```plaintext:改修前のディレクトリ構成
.
├── static
│   └── style.css
├── templates
│   ├── index.html
│   ├── report.html
│   └── layout.html
├── .dockerignore
├── .env
├── .envsample
├── .gitignore
├── app.py
├── README.md
└── requirements.txt
```
この`app.py`に初期画面、PDFプレビュー画面、認証画面、APIの各ルーティングをすべて入れて`@require_auth`でロックを掛けた結果、想定通りにページが開けなくなって撃沈。
整理しきれないので泣く泣く`app.py`を分割することにした。

```python:app.py
@app.route("/generate_pdf")
@require_auth
def pdf(): ...
@app.route("/login")
@require_auth
def login(): ...
```

### 2. 構成と認証をちゃんと分離

```plaintext:改修後のディレクトリ構成
.
├── papyrus
│   ├── __init__.py
│   ├── api_routes.py
│   ├── auth.py
│   ├── auth_routes.py
│   ├── db.py
│   └── routes.py
├── static
│   └── style.css
├── templates
│   ├── index.html
│   ├── report.html
│   └── layout.html
├── .dockerignore
├── .env
├── .env.sample
├── .gitignore
├── README.md
├── requirements.txt
└── run.py
```

改修前：`app = Flask(__name__)`をグローバル変数定義して使用
改修後：`create_app()`を導入し、`app.py`の役割を分割

```python
from papyrus import create_app
from dotenv import load_dotenv

load_dotenv()
app = create_app()

if __name__ == "__main__":
    app.run(debug=True)
```

```python
from flask import Flask
from papyrus.routes import register_routes
from papyrus.api_routes import register_api_routes
from papyrus.auth import init_auth
from papyrus.auth_routes import register_auth_routes
from .db import init_db
from dotenv import load_dotenv
import os

load_dotenv()
def create_app():
    BASE_DIR = os.path.abspath(os.path.dirname(__file__))
    TEMPLATE_DIR = os.path.join(BASE_DIR, '../templates')
    app = Flask(__name__, template_folder=TEMPLATE_DIR)
    app.secret_key = os.getenv("FLASK_SECRET_KEY")

    init_auth(app)
    init_db(app)
    register_routes(app)
    register_api_routes(app)
    register_auth_routes(app)
    return app
```

#### ポイント
- `load_dotenv()`は`app = Flask(__name__)`よりも**前で実行**
    - `.env`ファイルの情報が抜けると`create_app()`でエラー

- `app = Flask(__name__)`だと`templates\`のファイルを読み込んでくれないので、中に`temlate_foler`を設定して明示的にディレクトリ指定する。

- `create_app()`内で読み込むルーティング関数は、必要なものすべてそろっているか確認する。

### 3. Auth0導入

Auth0にログインする。アカウントがなければ作る。
「使用を開始」>「アプリケーションの開始」をクリック
![](https://storage.googleapis.com/zenn-user-upload/95ffc2ebc0ed-20250728.png)

「一般的なWebアプリケーション」を選択する。
![](https://storage.googleapis.com/zenn-user-upload/d02f2823ef98-20250728.png)

`Python`を選択
![](https://storage.googleapis.com/zenn-user-upload/cc0a2b8b8fbd-20250728.png)

「クイックスタート」を選び下へスクロール
![](https://storage.googleapis.com/zenn-user-upload/ac855e86959c-20250728.png)

「Configure your`.env` file」をコピーして`APP_SECRET_KEY`以外を`.env`に貼りつけ
![](https://storage.googleapis.com/zenn-user-upload/d1e36c993fb8-20250728.png)

「設定」>「アプリケーションのURI」欄に入力
![](https://storage.googleapis.com/zenn-user-upload/5734ab0c2e3d-20250730.png)

```plaintext:Configuration URI
# 許可するコールバックURL
http://localhost:5000/callback

# 許可するログアウトURL
http://localhost:5000/logout

```

そして...Auth0と連携成功
![](https://storage.googleapis.com/zenn-user-upload/3a379312c7f3-20250730.png)


![](https://storage.googleapis.com/zenn-user-upload/270816d67e11-20250729.png)
*Googleアカウントと連携していると警告が出る(本番up禁止!)*

ターミナルで`python3 -c 'import secrets; print(secrets.token_urlsafe(32))'`を実行し、その結果を`.env`の`FLASK_SECRET_KEY=`に貼りつけ


### 4. 他の問題点の解消

元の`app.py`では、納品リストをグローバル変数 `delivery_list = []` として定義していた。
しかしこの実装では、**複数ユーザーが同時にアクセスした場合に納品リストの内容が全ユーザー間で共有されてしまう**ため、セッションごとの分離ができていなかった。

この問題を解消するため、Flaskのセッション（`session`）を使用して、ユーザーごとに`delivery_list`を個別管理するよう変更した。

#### 変更後の実装：

```python
# GET: 納品リストの表示
@app.route("/index")
@requires_auth
def index():
    form_data = {}
    delivery_list = session.get("delivery_list", [])
    return render_template("index.html", delivery_list=delivery_list, form_data=form_data)
```

```python
# POST: 納品リストの更新
@app.route("/index", methods=["POST"])
@requires_auth
def handle_submit():
    form_data = {}
    delivery_list = session.get("delivery_list", [])
    action = request.form.get("action")

    # ここで delivery_list を編集してから session に戻す処理を書く（省略）

    session["delivery_list"] = delivery_list
    return render_template("index.html", delivery_list=delivery_list, form_data=form_data)
```

### 5. おわりに
最終的にAuth0取付できて、次回予定の本番環境構築準備ができた。
`app.py`オンリーの構成が`app.py`分割, `create_app()`導入, `Auth0`導入, セッション管理で本格構成にすることができた。
次は`VPS`に上げて本番環境の構築を試したい。