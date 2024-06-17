---
title: "【Lambda/API Gateway】翻訳Web APIの構築"
emoji: "🔠"
type: "tech"
topics: ["aws"]
published: true
---

https://pages.awscloud.com/rs/112-TZM-766/images/Hands-On-for-Beginners_2022_Serverless1_0819_v1.png

## 目標
- サーバレスアーキテクチャで翻訳Web APIを構築する
- Amazon Translateで実装する。JSON形式で英訳を返させる

## 前提条件
- ハンズオン用の新しいAWSアカウントを用意する
- AdministratorAccess を持つIAMユーザー

## 今回作成したリソースのIaC
:::details CloudFormationで生成されたコード(YAML)
```
---
Metadata:
  TemplateId: "arn:aws:cloudformation:ap-northeast-1:975050180086:generatedTemplate/770d767e-09bc-466a-bba6-0226a69462dd"
Parameters:
  LambdaFunction00Translate00LuE3MCodeS3BucketOneOfedcIS:
    NoEcho: "true"
    Type: "String"
    Description: "An Amazon S3 bucket in the same AWS-Region as your function. The\
      \ bucket can be in a different AWS-account.\nThis property can be replaced with\
      \ other exclusive properties"
  LambdaFunction00Translate00LuE3MCodeS3KeyOneOfJSSUQ:
    NoEcho: "true"
    Type: "String"
    Description: "The Amazon S3 key of the deployment package.\nThis property can\
      \ be replaced with other exclusive properties"
Resources:
  ApiGatewayStage00translate00SNmYN:
    UpdateReplacePolicy: "Retain"
    Type: "AWS::ApiGateway::Stage"
    DeletionPolicy: "Retain"
    Properties:
      DeploymentId:
        Fn::GetAtt:
        - "ApiGatewayDeployment001j3o3000pQ5pe"
        - "DeploymentId"
      StageName: "translate"
      TracingEnabled: false
      RestApiId:
        Ref: "ApiGatewayRestApi00fdwhye093700LbZUi"
      CacheClusterSize: "0.5"
      Tags:
      - Value: ""
        Key: "hands-on"
      CacheClusterEnabled: false
  ApiGatewayRestApi00fdwhye093700LbZUi:
    UpdateReplacePolicy: "Retain"
    Type: "AWS::ApiGateway::RestApi"
    DeletionPolicy: "Retain"
    Properties:
      ApiKeySourceType: "HEADER"
      EndpointConfiguration:
        Types:
        - "REGIONAL"
      DisableExecuteApiEndpoint: false
      Name: "translate-api"
  ApiGatewayDeployment001j3o3000pQ5pe:
    UpdateReplacePolicy: "Retain"
    Type: "AWS::ApiGateway::Deployment"
    DeletionPolicy: "Retain"
    Properties:
      RestApiId:
        Ref: "ApiGatewayRestApi00fdwhye093700LbZUi"
  LambdaPermission00functionTranslate00Gcat0:
    UpdateReplacePolicy: "Retain"
    Type: "AWS::Lambda::Permission"
    DeletionPolicy: "Retain"
    Properties:
      FunctionName:
        Fn::GetAtt:
        - "LambdaFunction00Translate00LuE3M"
        - "Arn"
      Action: "lambda:InvokeFunction"
      SourceArn: "arn:aws:execute-api:ap-northeast-1:975050180086:fdwhye0937/*/GET/dev"
      Principal: "apigateway.amazonaws.com"
  IAMManagedPolicy00policyserviceroleAWSLambdaBasicExecutionRole030be311094f443db3e977930bc4b97e00QncvE:
    UpdateReplacePolicy: "Retain"
    Type: "AWS::IAM::ManagedPolicy"
    DeletionPolicy: "Retain"
    Properties:
      ManagedPolicyName: "AWSLambdaBasicExecutionRole-030be311-094f-443d-b3e9-77930bc4b97e"
      Path: "/service-role/"
      Description: ""
      Groups: []
      PolicyDocument:
        Version: "2012-10-17"
        Statement:
        - Resource: "arn:aws:logs:ap-northeast-1:975050180086:*"
          Action: "logs:CreateLogGroup"
          Effect: "Allow"
        - Resource:
          - "arn:aws:logs:ap-northeast-1:975050180086:log-group:/aws/lambda/Translate:*"
          Action:
          - "logs:CreateLogStream"
          - "logs:PutLogEvents"
          Effect: "Allow"
      Roles:
      - Ref: "IAMRole00Translaterole8mxapp7a00Jrfko"
      Users: []
  LambdaFunction00Translate00LuE3M:
    UpdateReplacePolicy: "Retain"
    Type: "AWS::Lambda::Function"
    DeletionPolicy: "Retain"
    Properties:
      MemorySize: 128
      Description: ""
      TracingConfig:
        Mode: "PassThrough"
      Timeout: 3
      RuntimeManagementConfig:
        UpdateRuntimeOn: "Auto"
      Handler: "lambda_function.lambda_handler"
      Code:
        S3Bucket:
          Ref: "LambdaFunction00Translate00LuE3MCodeS3BucketOneOfedcIS"
        S3Key:
          Ref: "LambdaFunction00Translate00LuE3MCodeS3KeyOneOfJSSUQ"
      Role:
        Fn::GetAtt:
        - "IAMRole00Translaterole8mxapp7a00Jrfko"
        - "Arn"
      FileSystemConfigs: []
      FunctionName: "Translate"
      Runtime: "python3.12"
      PackageType: "Zip"
      LoggingConfig:
        LogFormat: "Text"
        LogGroup: "/aws/lambda/Translate"
      EphemeralStorage:
        Size: 512
      Tags:
      - Value: ""
        Key: "hands-on"
      Architectures:
      - "x86_64"
  IAMRole00Translaterole8mxapp7a00Jrfko:
    UpdateReplacePolicy: "Retain"
    Type: "AWS::IAM::Role"
    DeletionPolicy: "Retain"
    Properties:
      Path: "/service-role/"
      ManagedPolicyArns:
      - "arn:aws:iam::aws:policy/TranslateFullAccess"
      - "arn:aws:iam::aws:policy/AmazonDynamoDBFullAccess"
      - "arn:aws:iam::975050180086:policy/service-role/AWSLambdaBasicExecutionRole-030be311-094f-443d-b3e9-77930bc4b97e"
      MaxSessionDuration: 3600
      RoleName: "Translate-role-8mxapp7a"
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
        - Action: "sts:AssumeRole"
          Effect: "Allow"
          Principal:
            Service: "lambda.amazonaws.com"
      Tags:
      - Value: ""
        Key: "translate"
  DynamoDBTable00translatehistory00MndkT:
    UpdateReplacePolicy: "Retain"
    Type: "AWS::DynamoDB::Table"
    DeletionPolicy: "Retain"
    Properties:
      SSESpecification:
        SSEEnabled: false
      TableName: "translate-history"
      AttributeDefinitions:
      - AttributeType: "S"
        AttributeName: "timestamp"
      ContributorInsightsSpecification:
        Enabled: false
      BillingMode: "PROVISIONED"
      PointInTimeRecoverySpecification:
        PointInTimeRecoveryEnabled: false
      ProvisionedThroughput:
        WriteCapacityUnits: 1
        ReadCapacityUnits: 1
      KeySchema:
      - KeyType: "HASH"
        AttributeName: "timestamp"
      DeletionProtectionEnabled: false
      TableClass: "STANDARD"
      Tags:
      - Value: ""
        Key: "hands-on"
      TimeToLiveSpecification:
        Enabled: false

```
:::

## 実装手順

### 1.Lambda Functionを作成する
Lambdaのページで「関数を作成する」をクリック
![](https://storage.googleapis.com/zenn-user-upload/c910a43fc21c-20240617.png)

|項目名|設定値|
|---|---|
|オプション|一から作成|
|ランタイム|Python(最新版)|
|アーキテクチャ|x86_64|

![](https://storage.googleapis.com/zenn-user-upload/b1b8c4580c90-20240617.png)

「関数を作成する」をクリック

関数が作成されたら以下のページのソースコードをlambda_function.pyに貼り付ける
https://pages.awscloud.com/rs/112-TZM-766/images/04_lambda_translate.py

このソースコードは、以下のAWS SDK for Python (Boto3) のページの「APIリファレンス」にてTranslate及びtranslate_textと検索すると確認できる
https://aws.amazon.com/jp/sdk-for-python/
![](https://storage.googleapis.com/zenn-user-upload/a3ab88b69170-20240617.png)
![](https://storage.googleapis.com/zenn-user-upload/5ba8d7a342b0-20240617.png)

ソースコードを貼り付けたら、「Deploy」をクリックし、「Changes not deployed」のサインが消えるのを確認する
![](https://storage.googleapis.com/zenn-user-upload/2fca5f47ad6d-20240617.png)
![](https://storage.googleapis.com/zenn-user-upload/dc8a0272110b-20240617.png)

### 2.AWS TranslateとLambdaを連携させる

「設定」タブ>アクセス権限を開く
IAMロールへのリンクがあるのでクリック
![](https://storage.googleapis.com/zenn-user-upload/a5c954fbf79a-20240617.png)

「許可」タブを開き、許可の追加>ポリシーのアタッチを選択する
![](https://storage.googleapis.com/zenn-user-upload/6c5f38d93505-20240617.png)

「TranslateFullAccess」権限を選択し、追加する
※実際に実装する場合はもう少し権限の弱いものが推奨される
![](https://storage.googleapis.com/zenn-user-upload/8d1259c9bd38-20240617.png)

「TranslateFullAccess」権限が追加されたことを確認する
![](https://storage.googleapis.com/zenn-user-upload/639654c8e937-20240617.png)

Lambdaの画面に戻る
Lambda>関数>(関数名)>設定>アクセス権限
「リソースの概要」のトグルを開くとAWS Translateが追加されていることが確認できる
※表示されない場合はリロードする
![](https://storage.googleapis.com/zenn-user-upload/c4a18dfc3b0f-20240617.png)

「コード」タブを開き、「Test」をクリック

イベント名を入力し、保存する
ウィンドウが閉じたら、もう一度「Test」をクリック
![](https://storage.googleapis.com/zenn-user-upload/cb07d9e33f98-20240617.png)

イベントは実行され、ログが取得できる
output_textに「こんにちは」の英訳の"Hi"が格納されている
![](https://storage.googleapis.com/zenn-user-upload/16b66db157fe-20240617.png)

これで、LambdaからAWS Translateを呼び出すことができた

### 3.API GatewayとLambdaを連携させる
API Gatewayへ移動し、REST API の項目で「構築」ボタンをクリック
![](https://storage.googleapis.com/zenn-user-upload/fc1b3a636a8d-20240617.png)

「新しいAPI」を選択する
API名を入力後、APIエンドポイントタイプを「リージョン」にする
※世界中から呼び出せる「エッジ最適化」、インターナルに利用できる「プライベート」が選択できる
「APIを作成」をクリック

APIが作成されるので、「リソースを作成」をクリックする
![](https://storage.googleapis.com/zenn-user-upload/ff1c7e809ed2-20240617.png)

リソース名を入力し、「リソースを作成」をクリックする
![](https://storage.googleapis.com/zenn-user-upload/aa76aa3aeb0e-20240617.png)

リソースが作成される。続いて「メソッドの作成」をクリック
![](https://storage.googleapis.com/zenn-user-upload/fda2216a2558-20240617.png)

メソッドタイプを「GET」、統合タイプを「Lambda」に設定する
※REST APIの設計上GETでなくPOSTでやるべきだが、POSTだとローカル環境にターミナルアプリケーションが必要で、Windows環境だとクライアントをインストールする必要があり煩雑な作業が増える
![](https://storage.googleapis.com/zenn-user-upload/01ee377cd161-20240617.png)

Lambdaプロキシ統合をオンにする(API Gatewayからのinputとoutputをそのままパススルーする)
検索窓から先ほど作成したLambda関数を選択する
「メソッドを作成」をクリック
![](https://storage.googleapis.com/zenn-user-upload/7059c502a4cc-20240617.png)

GETメソッドが作成された
(統合レスポンスでプロキシ統合されていることを確認)
![](https://storage.googleapis.com/zenn-user-upload/671bf776fea5-20240617.png)

「メソッドリクエスト」を選択し、「編集」をクリックする

「URLクエリ文字列パラメータ」クエリ文字列を追加する
名前をinput_textにして必須にチェックを入れる。その後保存する

URLクエリ文字列パラメータにクエリ文字列が追加された
![](https://storage.googleapis.com/zenn-user-upload/96e8e48088f6-20240617.png)

「Lambda統合」をクリックすると連携されているLambda関数のリンクが表示されるのでクリック
![](https://storage.googleapis.com/zenn-user-upload/35277ff81994-20240617.png)

API Gateway側のルールに従ってレスポンスを返すようにソースを修正する
以下のページ中のソースと現在のソースを比較すると、headersとisBase54Encodedが抜けているので追加し、デプロイする
https://docs.aws.amazon.com/ja_jp/apigateway/latest/developerguide/api-gateway-integration-settings-integration-response.html
![](https://storage.googleapis.com/zenn-user-upload/52684ed455bf-20240617.png)

「Test」ボタンのトグルから「Configure test event」(テストイベントの設定)をクリック
![](https://storage.googleapis.com/zenn-user-upload/237ccea7eb96-20240617.png)

イベントアクションは「新しいイベントを作成」を選択、イベント共有は「プライベート」を選択
イベント名を入力
テンプレートオプションは API Gateway AWS Proxy を選択
![](https://storage.googleapis.com/zenn-user-upload/36eb2ffbc609-20240617.png)

「イベントJSON」のソースから queryStringParameters の項目を探し、内容を "input_text": (翻訳したい文字列)に変更し、「保存」を押す
(今回は「こんばんは」)
![](https://storage.googleapis.com/zenn-user-upload/a12dcbe07d0d-20240617.png)

input_text に代入する値を event['queryStringParameters']['input_text'] に変更し、デプロイする
![](https://storage.googleapis.com/zenn-user-upload/930e9e1d0f6f-20240617.png)

テスト実行をすると、「Good evening」が返っているので、API Gatewayと連携できていることがわかる
![](https://storage.googleapis.com/zenn-user-upload/50c30706f20b-20240617.png)

API Gateway の画面に戻り、APIをデプロイする
![](https://storage.googleapis.com/zenn-user-upload/b24e93d47880-20240617.png)

ステージは「新しいステージ」を選択し、ステージ名を入力する
「デプロイ」をクリックする
![](https://storage.googleapis.com/zenn-user-upload/2f791105c821-20240617.png)

デプロイ完了
![](https://storage.googleapis.com/zenn-user-upload/37834e412804-20240617.png)

ステージの詳細>URLを呼び出す からURLをコピーし、検索する
![](https://storage.googleapis.com/zenn-user-upload/7a3fdeeda31b-20240617.png)

API Gateway REST API エンドポイントの 403 エラー「Missing Authentication Token"」が出た場合は、パスが合っているかどうか確認する
![](https://storage.googleapis.com/zenn-user-upload/30d7e03c08c7-20240617.png)

GETメソッドを選択している状態でURLをコピー
![](https://storage.googleapis.com/zenn-user-upload/4a83da98d9a6-20240617.png)

Internal server errorが出た場合、URLにクエリ文字を追加する
![](https://storage.googleapis.com/zenn-user-upload/c1564598cfbe-20240617.png)
![](https://storage.googleapis.com/zenn-user-upload/c6dea1dad9f4-20240617.png)

APIが作動していることが確認できた

https://docs.aws.amazon.com/ja_jp/apigateway/latest/developerguide/amazon-api-gateway-using-stage-variables.html

### 4.Dynamo DBとLambdaを連携させる

Dynamo DB>テーブル より、「テーブルを作成」をクリック
![](https://storage.googleapis.com/zenn-user-upload/ee29f6bf7660-20240617.png)

テーブル名とパーティションキーを記入
![](https://storage.googleapis.com/zenn-user-upload/caed2d5d4555-20240617.png)

|項目名|設定値|
|---|---|
|テーブル設定|設定をカスタマイズ|
|テーブルクラス|Dynamo DB 標準|

#### 読み込み/書き込みキャパシティーの設定 

|項目名|設定値|
|---|---|
|キャパシティーモード|プロビジョンド|
|読込キャパシティ|オフ|
|プロビジョンドキャパシティーユニット|1|
|書込キャパシティ|オフ|
|プロビジョンドキャパシティーユニット|1|

その他項目はデフォルトのまま、「テーブルを作成」

時間経過後、テーブルが作成される
![](https://storage.googleapis.com/zenn-user-upload/5c2ce6998e34-20240617.png)

AWS SDK for Python (Boto3) のページの「APIリファレンス」にて dynamo db と検索する
https://aws.amazon.com/jp/sdk-for-python/

dynabo db>Resources>Table を選択する
![](https://storage.googleapis.com/zenn-user-upload/0be7c8bb6292-20240617.png)

Actions>put_item>Request Syntaxに移動
![](https://storage.googleapis.com/zenn-user-upload/883e7570950b-20240617.png)
![](https://storage.googleapis.com/zenn-user-upload/1e4465bc7bdd-20240617.png)

ソース修正後、デプロイ
https://pages.awscloud.com/rs/112-TZM-766/images/10_lambda_dynamodb.py

AWS Translate の TranslateFullAccess の時と同様に、 DynamoDBFullAccess IAMポリシーをアタッチする
![](https://storage.googleapis.com/zenn-user-upload/84f949a0f299-20240617.png)

Lambdaの画面をリロードし、アクセス権限のトグルを開くと、Dynamo DB へのアクセス権限が追加されていることが確認できる
![](https://storage.googleapis.com/zenn-user-upload/3d0e5c2c3dc4-20240617.png)

確認のために、イベントJSONのinput_textの値を変更する
![](https://storage.googleapis.com/zenn-user-upload/d0724c1fbcac-20240617.png)

"Good morning" に変更されているのが確認できる
![](https://storage.googleapis.com/zenn-user-upload/fd4f7506eeda-20240617.png)

Dynamo DB>項目を探索>(データベース名) を確認すると、先ほどクエリした結果が格納されていることがわかる
![](https://storage.googleapis.com/zenn-user-upload/882bcfd4edfd-20240617.png)

再び、API Gateway からURLを取得
![](https://storage.googleapis.com/zenn-user-upload/88549b0fd743-20240617.png)

APIが叩けていることを確認
![](https://storage.googleapis.com/zenn-user-upload/5739a6b92f80-20240617.png)

Dynamo DB に格納されていることを確認
![](https://storage.googleapis.com/zenn-user-upload/db5997c42c3e-20240617.png)

アーキテクチャの構築完了