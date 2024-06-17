---
title: "【Lambda/API Gateway】翻訳Web APIの構築"
emoji: "🔠"
type: "tech"
topics: ["aws"]
published: false
---

https://pages.awscloud.com/rs/112-TZM-766/images/Hands-On-for-Beginners_2022_Serverless1_0819_v1.png

# 目標
- サーバレスアーキテクチャで翻訳Web APIを構築する
- Amazon Translateで実装する。JSON形式で英訳を返させる

# 前提条件
- ハンズオン用の新しいAWSアカウントを用意する
- AdministratorAccess を持つIAMユーザー

# 今回作成したのリソースのIaC
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

# 実装手順


## 1.Lambda Functionを作成する
Lambdaのページで「関数を作成する」をクリック


|項目名|設定値|
|---|---|
|オプション|一から作成|
|ランタイム|Python(最新版)|
|アーキテクチャ|x86_64|

「関数を作成する」をクリック

関数が作成されたら以下のページのソースコードをlambda_function.pyに貼り付ける
https://pages.awscloud.com/rs/112-TZM-766/images/04_lambda_translate.py

このソースコードは、以下のAWS SDK for Python (Boto3) のページの「APIリファレンス」にてTranslate及びtranslate_textと検索すると確認できる
https://aws.amazon.com/jp/sdk-for-python/


ソースコードを貼り付けたら、「Deploy」をクリックし、「Changes not deployed」のサインが消えるのを確認する

## 2.AWS TranslateとLambdaを連携させる

「設定」タブ>アクセス権限を開く
IAMロールへのリンクがあるのでクリック

「許可」タブを開き、許可の追加>ポリシーのアタッチを選択する

「TranslateFullAccess」権限を選択し、追加する
※実際に実装する場合はもう少し権限の弱いものが推奨される


「TranslateFullAccess」権限が追加されたことを確認する


Lambdaの画面に戻る
Lambda>関数>(関数名)>設定>アクセス権限
「リソースの概要」のトグルを開くとAWS Translateが追加されていることが確認できる
※表示されない場合はリロードする

「コード」タブを開き、「Test」をクリック


イベント名を入力し、保存する
ウィンドウが閉じたら、もう一度「Test」をクリック


イベントは実行され、ログが取得できる
output_textに「こんにちは」の英訳の"Hi"が格納されている


これで、LambdaからAWS Translateを呼び出すことができた

## 3.API GatewayとLambdaを連携させる
API Gatewayへ移動し、REST API の項目で「構築」ボタンをクリック

「新しいAPI」を選択する
API名を入力後、APIエンドポイントタイプを「リージョン」にする
※世界中から呼び出せる「エッジ最適化」、インターナルに利用できる「プライベート」が選択できる
「APIを作成」をクリック



APIが作成されるので、「リソースを作成」をクリックする


リソース名を入力し、「リソースを作成」をクリックする


リソースが作成される。続いて「メソッドの作成」をクリック


メソッドタイプを「GET」、統合タイプを「Lambda」に設定する
※REST APIの設計上GETでなくPOSTでやるべきだが、POSTだとローカル環境にターミナルアプリケーションが必要で、Windows環境だとクライアントをインストールする必要があり煩雑な作業が増える


Lambdaプロキシ統合をオンにする(API Gatewayからのinputとoutputをそのままパススルーする)
検索窓から先ほど作成したLambda関数を選択する
「メソッドを作成」をクリック


GETメソッドが作成された
(統合レスポンスでプロキシ統合されていることを確認)


「メソッドリクエスト」を選択し、「編集」をクリックする



「URLクエリ文字列パラメータ」クエリ文字列を追加する
名前をinput_textにして必須にチェックを入れる。その後保存する



URLクエリ文字列パラメータにクエリ文字列が追加された


「Lambda統合」をクリックすると連携されているLambda関数のリンクが表示されるのでクリック


API Gateway側のルールに従ってレスポンスを返すようにソースを修正する
以下のページ中のソースと現在のソースを比較すると、headersとisBase54Encodedが抜けているので追加し、デプロイする
https://docs.aws.amazon.com/ja_jp/apigateway/latest/developerguide/api-gateway-integration-settings-integration-response.html


「Test」ボタンのトグルから「Configure test event」(テストイベントの設定)をクリック


イベントアクションは「新しいイベントを作成」を選択、イベント共有は「プライベート」を選択
イベント名を入力
テンプレートオプションは API Gateway AWS Proxy を選択



「イベントJSON」のソースから queryStringParameters の項目を探し、内容を "input_text": (翻訳したい文字列)に変更し、「保存」を押す
(今回は「こんばんは」)


input_text に代入する値を event['queryStringParameters']['input_text'] に変更し、デプロイする



テスト実行をすると、「Good evening」が返っているので、API Gatewayと連携できていることがわかる


API Gateway の画面に戻り、APIをデプロイする


ステージは「新しいステージ」を選択し、ステージ名を入力する
「デプロイ」をクリックする


デプロイ完了


ステージの詳細>URLを呼び出す からURLをコピーし、検索する


API Gateway REST API エンドポイントの 403 エラー「Missing Authentication Token"」が出た場合は、パスが合っているかどうか確認する


GETメソッドを選択している状態でURLをコピー


Internal server errorが出た場合、URLにクエリ文字を追加する


APIが作動していることが確認できた

https://docs.aws.amazon.com/ja_jp/apigateway/latest/developerguide/amazon-api-gateway-using-stage-variables.html

## 4.Dynamo DBとLambdaを連携させる

Dynamo DB>テーブル より、「テーブルを作成」をクリック


テーブル名とパーティションキーを記入


|項目名|設定値|
|---|---|
|テーブル設定|設定をカスタマイズ|
|テーブルクラス|Dynamo DB 標準|

### 読み込み/書き込みキャパシティーの設定 

|項目名|設定値|
|---|---|
|キャパシティーモード|プロビジョンド|
|読込キャパシティ|オフ|
|プロビジョンドキャパシティーユニット|1|
|書込キャパシティ|オフ|
|プロビジョンドキャパシティーユニット|1|

その他項目はデフォルトのまま、「テーブルを作成」


時間経過後、テーブルが作成される


AWS SDK for Python (Boto3) のページの「APIリファレンス」にて dynamo db と検索する
https://aws.amazon.com/jp/sdk-for-python/


dynabo db>Resources>Table を選択する


Actions>put_item>Request Syntaxに移動


ソース修正後、デプロイ


AWS Translate の TranslateFullAccess の時と同様に、 DynamoDBFullAccess IAMポリシーをアタッチする


Lambdaの画面をリロードし、アクセス権限のトグルを開くと、Dynamo DB へのアクセス権限が追加されていることが確認できる


確認のために、イベントJSONのinput_textの値を変更する


"Good morning" に変更されているのが確認できる


Dynamo DB>項目を探索>(データベース名) を確認すると、先ほどクエリした結果が格納されていることがわかる


再び、API Gateway からURLを取得


APIが叩けていることを確認


Dynamo DB に格納されていることを確認

アーキテクチャの構築完了

https://pages.awscloud.com/rs/112-TZM-766/images/10_lambda_dynamodb.py