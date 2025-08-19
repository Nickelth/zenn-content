---
title: "【#4 2/4】アプリ本番環境公開に向けてAWS基盤を準備する"
emoji: "🔑"
type: "tech"
topics: ["aws", "auth0"]
published: false
---

## AWS基盤を準備する（CloudWatch, SSM/Secrets, ECR, Auth0編）

### 0. はじめに

### 1. CloudWatch Logs ロググループ作成

#### コンソールで作成
1. CloudWatch → ログ → ロググループ → Create log group

2. Name: `/ecs/papyrus`、Region: `us-west-2`

3. 保持期間: 1 day（デモ用途ならこれで十分）

#### CLIで作成

```bash
LOG_GROUP=/ecs/papyrus
REGION=us-west-2   # か ap-northeast-1

aws logs create-log-group --log-group-name "$LOG_GROUP" --region "$REGION" 2>/dev/null || true
aws logs put-retention-policy --log-group-name "$LOG_GROUP" --retention-in-days 1 --region "$REGION"
aws logs tag-log-group --log-group-name "$LOG_GROUP" --tags project=portfolio --region "$REGION"
```

### 2. SSM/Secrets

AWS Secret Managerの登録用コマンド。
`--type`オプションにて、`String`で非機密、`SecureString`で機密に設定することができる。

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
```

#### aws ssm コマンドのレスポンスの意味
```json
{
  "Version": 1,
  "Tier": "Standard"
}
```
Version: そのパラメータのバージョン番号。初回は1、値を更新するたびに+1される。
Tier: そのパラメータの階層（Standard / Advanced / Intelligent-Tiering）。Standardなら課金なし。


AWS System Managerのダッシュボード > 「パラメータストア」をクリック
![AWS System Manager ダッシュボード](https://storage.googleapis.com/zenn-user-upload/a1a9be9a6438-20250819.png)

先ほど作成したパラメータとその型があっていることを確認する。
![AWS System Manager パラメータストア](https://storage.googleapis.com/zenn-user-upload/624de395033a-20250819.png)



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