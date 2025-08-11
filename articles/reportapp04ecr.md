---
title: "【#4】GitHub ActionsでDockerをECRに登録する手順"
emoji: "🦄"
type: "tech"
topics: ["githubactions", "docker", "ecr", "github"]
published: false
---

## GitHub Actions登録手順

### 0. はじめに


#### 使用技術の選定理由



#### 1. GitHubでの操作

「該当リポジトリ」＞「Setting」＞「Actions」をクリック

![Actionsを選択](https://storage.googleapis.com/zenn-user-upload/0cafadbbe168-20250810.png)

![Settings](https://storage.googleapis.com/zenn-user-upload/68051b626775-20250810.png)

![SecurityからActionsへ](https://storage.googleapis.com/zenn-user-upload/5174213ae1e2-20250810.png)

![Variablesを選択](https://storage.googleapis.com/zenn-user-upload/db71600d3fcd-20250810.png)

「Maneage environment variables」＞「New environment」の順にクリックして以下の変数を記入する。

![Repository Variablesをクリック](https://storage.googleapis.com/zenn-user-upload/e543e79be625-20250811.png)

![New Variables](https://storage.googleapis.com/zenn-user-upload/21ee424ba83f-20250811.png)

機密情報を入力する場合は「Secrets」タブに切り替える。
「Actions secrets/New secret」の画面が出るので、同様に入力する。

![Add Secrets](https://storage.googleapis.com/zenn-user-upload/6874b3b685d6-20250811.png)

```plaintext
AWS_ACCOUNT_ID

AWS_REGION（例: ap-northeast-1）

ECR_REPOSITORY（例: papyrus）

AWS_IAM_ROLE_ARN（GitHub OIDCを信頼するIAMロールのARN）


```
token.actions.githubusercontent.com
#### アプリ構成図



### 2. 

#### 2.1 


#### 2.2



#### 2.3 

### 3. 

#### 3.1 


#### 3.2 



#### 3.3 ス