---
title: "FlaskアプリにAuth0つけようとしたらapp.pyがぶっ壊れた話"
emoji: "⭐"
type: "tech"
topics: ["Auth0", "Python", "Flask"]
published: false
---

>Flaskは1ファイルで済むって言ったやつ出てこい

### 0. はじめに
本番環境に公開するためにAuth0入れてみたけどapp.pyが肥大化して詰んだ。
当初はAuth0をapp.pyに直接投入していたが、ルーティングぐちゃぐちゃになってページが開けなくなったので泣く泣く分割。
ハマった知見とかも共有しときます。

### 1. 初期状態：フル装備`app.py`の地獄

* ファイルサイズ・ルートの数・Authのロジック混在
* これを直さないとつらい例（認証ルートがPDFルートと隣にある罪）

```python
@app.route("/generate_pdf")
def pdf(): ...
@app.route("/login")
def login(): ...
```

### 2. 目標：**「構成と認証をちゃんと分離」**

* `create_app()`を導入してFlaskアプリを工場生産
* `routes.py`, `auth.py`, `db.py` に分割
* `.env`活用、セキュアな環境変数管理

図解あると神（「Before: app.pyに全部詰まってる」「After: モジュールで分割」）



### 3. Auth0導入：そこそこ痛いけどやる価値あり

* Flask + Auth0 の実装で詰まるポイント（callback地獄、セッション管理）
* `@requires_auth` デコレータの意味と使い所
* 認証ルートは **auth.pyに分離すべし** の哲学

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
![](https://storage.googleapis.com/zenn-user-upload/3b0f7df4f37a-20250729.png)

```plaintext
# 許可するコールバックURL
http://localhost:5000/callback,http://127.0.0.1:5000/callback

# 許可するログアウトURL
http://localhost:5000/logout,http://127.0.0.1:5000/logout

```

![](https://storage.googleapis.com/zenn-user-upload/0bcf927fb6dc-20250729.png)


![](https://storage.googleapis.com/zenn-user-upload/270816d67e11-20250729.png)

ターミナルで`python3 -c 'import secrets; print(secrets.token_urlsafe(32))'`を実行し、その結果を`FLASK_SECRET_KEY=`に貼りつけ

```python:run.py
from papyrus import create_app
from dotenv import load_dotenv

load_dotenv()
app = create_app()

if __name__ == "__main__":
    app.run(debug=True)
```

```python:__init__.py
from flask import Flask
from papyrus.auth import init_auth
from .db import init_db
from dotenv import load_dotenv

load_dotenv()

def create_app():
    app = Flask(__name__)
    app.secret_key = "your_secret"
    init_auth(app)
    init_db(app)

    from .routes import register_routes
    register_routes(app)

    return app
```



### 4. 構成後の構成（爆発後に残った美しい世界）

* `run.py` から `create_app()` 呼び出し
* `Blueprint` は導入してないけど、その布石にはなってる話
* セッション管理で delivery\_list のグローバル脱却もサラッと紹介（ここ読者刺さる）



### 5. おわりに
最終的にAuth0取付できて、次回予定の本番環境構築準備ができた。

* app.pyがでかくなるのは誰にでも起きる
* でも、**構成を見直すタイミングとやり方を知ってるかどうかが違いになる**



* Flaskは気軽に始められるぶん、「構成」や「認証まわり」で死ぬ初心者が多い
* `create_app()`とAuth0で「実用的なFlaskアプリ」にグレードアップできる
* 小さなアプリでも、**構成を分けて育てると再利用できる技術になる**

