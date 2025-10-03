---
title: "【#5】アプリ本番環境公開に向けてAWS基盤を準備する"
emoji: "🔑"
type: "tech"
topics: ["aws", "auth0"]
published: false
---

## AWS基盤を準備する（CloudWatch, SSM/Secrets, ECR, Auth0編）

### 0. はじめに

※特に断りがない限り、リージョンは`us-west-2`とする。

### 1. CloudWatch Logs ロググループ作成

#### パターン1: コンソールで作成
1. CloudWatch → ログ → ロググループ → Create log group

2. Name: `/ecs/papyrus`、Region: `us-west-2`

3. 保持期間: 1 day（デモ用途ならこれで十分）

#### パターン2: CLIで作成

```bash
LOG_GROUP=/ecs/papyrus
REGION=us-west-2   # か ap-northeast-1

aws logs create-log-group --log-group-name "$LOG_GROUP" --region "$REGION" 2>/dev/null || true
aws logs put-retention-policy --log-group-name "$LOG_GROUP" --retention-in-days 1 --region "$REGION"
aws logs tag-log-group --log-group-name "$LOG_GROUP" --tags project=portfolio --region "$REGION"
```

### 2. 環境変数を SSM/Secrets または Secret Manager に格納

開発環境で使用していた`.env`ファイル中の環境変数は、機密度に応じて `SSM/Secrets` または `Secret Manager` から選択して格納する。

`Secret Manager` は 課金されるので注意。`JSON`形式に変数をまとめることで、呼び出し回数を節約できる。

**主な課金システム**
- 保管料（ストレージ）：$0.40/シークレット/月
- APIコール：$0.05 / 10,000コール（Create/Update/Get など全部カウント）
- 30日間の無料トライアルあり。

詳細は公式サイトへ。
@[card](https://aws.amazon.com/jp/secrets-manager/pricing/)

##### 主な変数の配置先

じゃあ具体的にどう振り分けたらいい？
という声にお応えして、主な変数の配置先をテーブルで示した。

| 変数 | 行き先 | 理由 |
| --- | --- | --- |
| `DB_PASSWORD`, `POSTGRES_PASSWORD`| **Secrets Manager**  | パスワード。将来ローテ前提。
| `AUTH0_CLIENT_SECRET`   | **Secrets Manager**  | 外部連携のクライアント秘密。ローテ候補。 
| `FLASK_SECRET_KEY`  | **Secrets Manager**  | セッション署名鍵。漏れたら全ログイン壊滅。 
| `DB_USER`, `DB_NAME`, `DB_HOST`, `POSTGRES_USER`, `POSTGRES_DB` | **どっちでもOK**（おすすめは**Secretsに同梱**） | 秘密ではないけど、**パスワードとセット**で1つのシークレットJSONにまとめると扱いやすい＆参照回数を減らせる。コスト最優先ならSSMでも可。 |
| `AUTH0_CLIENT_ID` | **どっちでもOK**（**Secretsに同梱**推し）  | これ自体は秘密じゃないが、Auth0のペアとしてまとめた方が実装が楽。 
| `AUTH0_DOMAIN`, `AUTH0_CALLBACK_URL` | **SSM Parameter Store**（String） | 単なる設定値。たまに変える。可視でもOKなら `String`、隠したいだけなら `SecureString`。 
| `GUNICORN_*` 一式   | **SSM Parameter Store**（String/Number） | チューニング値。機密ではない。頻繁に変える前提。

要するに、DB, Auth0, Flask は **Secret Manager**, それ以外は **SSM**。

#### 2.1 SSM/Secrets での登録方法
AWS Secret Managerの登録用コマンド。
`--type`オプションにて、`String`で非機密、`SecureString`で機密に設定することができる。
`--name` で指定する名称は自由

```bash
# 非機密
aws ssm put-parameter --region <REGION_ID> --name /papyrus/prd/AUTH0_DOMAIN \
  --type String --value your-tenant.us.auth0.com --overwrite

# 変動するコールバックURL（とりあえず仮。起動後に更新して再デプロイ）
aws ssm put-parameter --region <REGION_ID> --name /papyrus/prd/AUTH0_CALLBACK_URL \
  --type String --value http://0.0.0.0:5000/callback --overwrite

# 機密
aws ssm put-parameter --region <REGION_ID> --name /papyrus/prd/AUTH0_CLIENT_SECRET \
  --type SecureString --value 'xxxx' --overwrite

# 削除
aws ssm delete-parameter --region <REGION_ID> --name /papyrus/prd/AUTH0_CLIENT_SECRET
```

##### aws ssm コマンドのレスポンスの意味
```json
{
  "Version": 1,
  "Tier": "Standard"
}
```
Version: そのパラメータのバージョン番号。初回は1、値を更新するたびに+1される。
Tier: そのパラメータの階層（Standard / Advanced / Intelligent-Tiering）。Standardなら課金なし。

##### パラメータがSystem Managerに登録されたことを確認する

AWS System Managerのダッシュボード > 「パラメータストア」をクリック
![AWS System Manager ダッシュボード](https://storage.googleapis.com/zenn-user-upload/a1a9be9a6438-20250819.png)

先ほど作成したパラメータとその型があっていることを確認する。
![AWS System Manager パラメータストア](https://storage.googleapis.com/zenn-user-upload/624de395033a-20250819.png)

#### 2.2 Secret Manager での登録方法

1シークレットで$0.4/月もかかるので、3つに集約して節約。

```bash
aws secretsmanager create-secret \
  --name "myapp/prd/db" \
  --secret-string '{"username":"db_user","password":"yourpassword","host":"your_host","name":"db_name"}'

aws secretsmanager create-secret \
  --name "myapp/prd/auth0" \
  --secret-string '{"client_id":"auth0_client_id","client_secret":"auth0_client_secret","domain":"auth0_domain.com"}'

aws secretsmanager create-secret \
  --name "myapp/prd/flask" \
  --secret-string '{"secret_key":"flask_secret_key"}'
```

![Secret Managerにパラメータを格納完了](https://storage.googleapis.com/zenn-user-upload/6e419f17b98a-20250821.png)
*格納完了*

### 3. ECR作成 & 手動Push

**ECR作成**

* プライベートリポジトリ
* タグ=Mutable, 暗号化=AES-256

イメージタグはミュータブル、暗号化設定はAESを選択する。
![ECRのミュータブル・暗号化設定](https://storage.googleapis.com/zenn-user-upload/ee057d171e1f-20250811.png)

ECR作成完了
![ECR](https://storage.googleapis.com/zenn-user-upload/0db90c13e04f-20250811.png)

### 4. Auth0の対応

Auth0の設定、及び`.env`ファイルの`AUTH0_CALLBACK_URL`を変更しておく。

|項目名|値|
|---|---|
|許可するCallbackURL|`http://<PublicIP>:5000/callback`|
|許可するURL|`http://<PublicIP>:5000/logout`|
|許可するWebオリジン|`http://<PublicIP>:5000`
|`AUTH0_CALLBACK_URL`|`http://<PublicIP>:5000/callback`|

### 5. チェックリスト（ここまでの出来上がり）



### 出典
@[card](https://docs.aws.amazon.com/cli/latest/reference/ssm/put-parameter.html)

<!--
### 6. 次の記事

@[card](https://zenn.dev/nickelth/articles/reportapp06ecs)
-->