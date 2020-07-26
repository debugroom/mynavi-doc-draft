.. include:: ../module.txt

.. _section-cloud-native-s3-and-lambda-2nd-label:

【第5回】AmazonS3で発生したイベント契機で実行するサーバレスアプリケーション(2)
----------------------------------------------------------------------------------------

|br|

前回は、S3へダイレクトアップロードした後の後続処理をAWS Lambdaファンクションを使ったサーバレスアプリケーションとして実装しました。続く今回は実際にAWSに環境を構築してLambdaファンクションをデプロイして実行してみます。

|br|

.. _section-cloud-native-prepare-s3-label:

事前準備：デプロイのためのS3バケット作成とアップロード
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

前回も解説した通り、以下の構成図通り(7)の部分でLambdaファンクションとして、DynamoDBへのアクセスとSQSキュー送信を実装しています。

|br|

.. figure:: img/aws-s3-and-lambda/serverless-postprocess.png

|br|

今回はAWS LambdaのデプロイとアクセスするDynamoDB(7')、SQS(7'')の構築、トリガーとなるS3およびイベント設定(6)をCloudFormationで実装します。
CloudFormationの基本的な使い方は「AWSで実践！デプロイ・基盤自動化」の `第21回 <https://news.mynavi.jp/itsearch/article/devsoft/4725>`_ 以降を、S3やDynamoDBをCloudFormationで作成する要領は、
`第30回 <https://news.mynavi.jp/itsearch/article/devsoft/4896>`_ および `第32回 <https://news.mynavi.jp/itsearch/article/devsoft/4926>`_ も参照してください。
ただし、Lambdaのデプロイを行う前に、図中のものとは別のデプロイ用S3バケットを作成して、前回作成したLambdaファンクションをアップロードしておく必要があります。
そこで、S3バケットを作成するCloudFormationテンプレートとLambdaファンクションをビルドしてアップロードするスクリプトをまず実装します。

S3バケットを作成するCloudFormationテンプレートではバケットの名称やARNをOutput要素で指定しておきます、これは、後に作成するLambdaファンクションのデプロイを行うCloudFormationテンプレートでクロススタックリファレンスで参照するためです。

.. sourcecode:: none

   AWSTemplateFormatVersion: '2010-09-09'

   Description: S3 Bucket for Lambda function template with YAML - S3 Bucket Definition

   Resources:
     S3Bucket:
       Type: AWS::S3::Bucket
       Properties:
         BucketName: debugroom-mynavi-sample-lambda-s3event-for-deploy
         AccessControl: Private
         PublicAccessBlockConfiguration:
           BlockPublicAcls: True
           BlockPublicPolicy: True
           IgnorePublicAcls: True
           RestrictPublicBuckets: True

   Outputs:
     S3Bucket:
       Description: Lambda deploy S3 bucket name
       Value: !Ref S3Bucket
       Export:
         Name: MynaviSampleLambdaS3Event-deployS3Bucket

     S3BucketArn:
       Description: Deploy S3 for Lambda bucket arn
       Value: !GetAtt S3Bucket.Arn
       Export:
         Name: MynaviSampleLambdaS3Event-deployS3BucketArn

|br|

続いて、前回実装したLambdaファンクションをビルドして、上記のS3バケットにアップロードするシェルスクリプトを作成します。

|br|

.. sourcecode:: bash

   #!/usr/bin/env bash

   bucket_name=debugroom-mynavi-sample-lambda-s3event-for-deploy
   stack_name="mynavi-sample-s3-lambda-s3event"
   template_path="src/main/cloudformation/s3-deploy-lambda-cfn.yml"
   s3_objectkey="mynavi-sample-aws-lambda-s3event-0.0.1-SNAPSHOT-aws.jar"

   if [ "" == "`aws s3 ls | grep $bucket_name`" ]; then
      aws cloudformation deploy --stack-name ${stack_name} --template-file ${template_path} --capabilities CAPABILITY_IAM
   fi

   ./mvnw package
   aws s3 cp target/${s3_objectkey} s3://${bucket_name}/


|br|

上記のスクリプトを実行して、バケット内にJarファイルがあることを確認しましょう。

|br|

.. figure:: img/aws-s3-and-lambda/management-console-s3.png

|br|

.. _section-cloud-native-cloudformation-deploy-lambda-label:

CloudFormationを使った環境構築とLambdaファンクションのデプロイ
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

引き続き、DynamoDBおよびSQSを構築するCloudFormationを実装します。
DynamoDBのテンプレートでは、テーブル定義をリソースとして定義しますが、前回の実装でDynamoDBのキーとして指定したfileIdをHASH属性で指定します。またLambdaファンクションから参照するため、Output要素にエンドポイントとリージョンを指定しておきます。

|br|

.. sourcecode:: none

   AWSTemplateFormatVersion: '2010-09-09'

   Description: Sample CloudFormation template with YAML - DynamoDB Definition

   Resources:
     DynamoDBUploadFileTable:
       Type: AWS::DynamoDB::Table
       Properties:
         TableName: "upload-file-table"
         BillingMode: PROVISIONED
         AttributeDefinitions:
           - AttributeName: fileId
             AttributeType: S
         KeySchema:
           - AttributeName: fileId
             KeyType: HASH
         ProvisionedThroughput:
           ReadCapacityUnits: 5
           WriteCapacityUnits: 5

   Outputs:
     EnvironmentRegion:
       Description: Dev Environment Region
       Value: !Sub ${AWS::Region}
       Export:
         Name: MynaviSampleLambdaS3Event-DynamoDB-Region
     DynamoDBServiceEndpoint:
       Description: DynamoDB service endipoint
       Value: !Sub https://dynamodb.${AWS::Region}.amazonaws.com
       Export:
         Name:  MynaviSampleLambdaS3Event-DynamoDB-ServiceEndpoint

|br|

同様にSQSのテンプレートでも、リソースの定義に加えて、Output要素にエンドポイントとリージョン、そしてキュー名をLambdaファンクションからの参照用に出力しておきます。

|br|

.. sourcecode:: none

   AWSTemplateFormatVersion: '2010-09-09'

   Description: Sample CloudFormation template with YAML - SQS Definition

   Resources:
     SQSSampleQueue:
       Type: AWS::SQS::Queue
       Properties:
         QueueName: mynavi-sample-lambda-s3event-queue
         VisibilityTimeout: 30
         DelaySeconds: 5
         MaximumMessageSize: 26144
         MessageRetentionPeriod: 345600
         ReceiveMessageWaitTimeSeconds: 0

   Outputs:
     SQSServiceEndpoint:
       Description: SQS service endipoint
       Value: !Sub https://sqs.${AWS::Region}.amazonaws.com
       Export:
         Name: MynaviSampleLambdaS3Event-SQS-ServiceEndpoint

     SQSServiceRegion:
       Description: SQS service region
       Value: !Sub ${AWS::Region}
       Export:
         Name: MynaviSampleLambdaS3Event-SQS-Region

     SQSSampleQueue:
       Description: SQS sample queue name
       Value: !Ref SQSSampleQueue
       Export:
         Name: MynaviSampleLambdaS3Event-SQS-QueueName

|br|

続いて、LambdaファクションをデプロイするCloudFormationテンプレートを実装します。

|br|

.. sourcecode:: none

   AWSTemplateFormatVersion: '2010-09-09'

   Description: Lambda template with YAML

   Resources:
     SampleLambdaS3Event:
       Type: AWS::Lambda::Function                                              # (A)
       Properties:
         Code:
           S3Bucket:
             Fn::ImportValue: MynaviSampleLambdaS3Event-deployS3Bucket          # (B)
         S3Key: mynavi-sample-aws-lambda-s3event-0.0.1-SNAPSHOT-aws.jar         # (C)
         Handler: org.debugroom.mynavi.sample.aws.lambda.s3event.app.handler.S3UploadEventHandler::handleRequest
                                                                                # (D)
         FunctionName: mynavi-sample-aws-lambda-s3event-function
         Environment:
           Variables:
             FUNCTION_NAME: sampleFunction                                      # (E)
         MemorySize: 1024                                                       # (F)
         Runtime: java8
         Timeout: 120
         Role: !GetAtt LambdaRole.Arn                                           # (G)

     LambdaRole:                                                                # (H)
       Type: AWS::IAM::Role
       Properties:
         Path: /
         AssumeRolePolicyDocument:
           Statement:
             - Action: sts:AssumeRole
               Effect: Allow
               Principal:
                 Service: lambda.amazonaws.com
         ManagedPolicyArns:
           - "arn:aws:iam::aws:policy/CloudWatchLogsFullAccess"

     SQSAccessPolicy:                                                           # (I)
       Type: AWS::IAM::Policy
       Properties:
         PolicyName: mynavi-sample-lambda-s3event-sqs-access-policy
         PolicyDocument:
           Statement:
             - Effect: Allow
               Action:
                 - "sqs:*"
               Resource: "*"
         Roles:
           - !Ref LambdaRole

     DynamoDBAccessPolicy:                                                      # (J)
       Type: AWS::IAM::Policy
       Properties:
         PolicyName: mynavi-sample-lambda-s3event-dynamodb-access-policy
         PolicyDocument:
           Statement:
             - Effect: Allow
               Action:
                 - "dynamodb:*"
                 # omit
           Resource: "*"
         Roles:
           - !Ref LambdaRole

     CloudFormationAccessPolicy:                                                # (K)
       Type: AWS::IAM::Policy
       Properties:
         PolicyName: mynavi-sample-lambda-s3event-cloudformation-access-policy
         PolicyDocument:
           Statement:
             - Effect: Allow
               Action:
                 - "cloudformation:*"
               Resource: "*"
       Roles:
         - !Ref LambdaRole

     SSMAccessPolicy:                                                           # (L)
       Type: AWS::IAM::Policy
       Properties:
       PolicyName: mynavi-sample-lambda-s3event-ssm-access-policy
       PolicyDocument:
         Statement:
           - Effect: Allow
             Action:
               - "cloudwatch:PutMetricData"
               # omit
       Roles:
         - !Ref LambdaRole

   Outputs:
     SampleLambdaS3EventArn:                                                    # (M)
       Value: !GetAtt SampleLambdaS3Event.Arn
       Export:
       Name: MynaviSampleLambdaS3Event-LambdaArn

|br|

Lambdaをデプロイするテンプレートのポイントになる設定箇所は以下の通りです。

|br|

.. list-table:: Lambdaファンクションのデプロイ用のテンプレートの作成
   :widths: 1, 19

   * - 項番
     - 説明

   * - (A)
     - Lambdaファンクションとしてのリソースを定義します。定義の詳細は `AWS::Lambda::Function <https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-lambda-function.html>`_ も参照してください。

   * - (B)
     - 前節で作成したデプロイ用S3テンプレートのOutput要素で出力したバケット名をクロススタックリファレンスを使って取得します。

   * - (C)
     - 前節のシェルスクリプト内でMavenビルドしたLambdaファンクションのJarファイル名を指定します。

   * - (D)
     - 前回実装したHandlerクラスのFQCNとハンドラメソッドを指定します。

   * - (E)
     - クラウドネイティブ基本編の `第2回 <https://news.mynavi.jp/itsearch/article/devsoft/4318>`_ と同様、環境変数にFUNCATION_NAMEとして、ファンクションクラスとなるBean名を指定しておきます(Spring Cloud Functionでは、実行する関数を環境変数FUNCTION_NAMEで指定したBean名で取得するためです。)。

   * - (F)
     - ファンクション実行時のメモリサイズを指定します。SpringアプリケーションをLambdaで実行する場合1024MB以上のメモリサイズを確保しなければ、起動時間が大幅にかかってしまうケースがあります。

   * - (G)
     - Lambdaに設定するIAMロールのARNを指定します。ここでは(H)で定義したARNを設定します。

   * - (H)
     - Lambdaに設定するIAMロールを定義します。ファンクション内の実装では、DynamoDBやSQSに加えて。CloudFormationのスタック参照、SystemsManager Parameter Storeの参照を行いますが、(I)〜(L)で定義したポリシーからアタッチして、このロールを参照するように設定します。

   * - (I)
     - SQSのアクセスを可能にするIAMポリシーを定義し、(H)のロールへアタッチします。

   * - (J)
     - DynamoDBのアクセスを可能にするIAMポリシーを定義し、(H)のロールへアタッチします。

   * - (K)
     - CloudFormationスタックのアクセスを可能にするIAMポリシーを定義し、(H)のロールへアタッチします。

   * - (L)
     - SystemsManager Parameter Storeへのアクセスを可能にするIAMポリシーを定義し、(H)のロールへアタッチします。

   * - (M)
     - 後述するアップロード用のS3テンプレートのイベント設定で参照するため、LambdaファンクションのARNをOutput要素として出力します。


|br|

続いて、アップロードによりイベントを発生させるためのS3の設定を行います。前回の連載までに使用してきたアップロード用のS3バケットを一度削除して、下記の通りLambdaへのイベントトリガーの設定を加えてCloudFormationで作成しなおします。

|br|

.. sourcecode:: none

   AWSTemplateFormatVersion: '2010-09-09'

   Description: Sample CloudFormation template with YAML - S3 Bucket Definition

   Resources:
     S3Bucket:                                                                  # (A)
       Type: AWS::S3::Bucket
       DependsOn: LambdaInvokePermission                                        # (B)
       Properties:
         BucketName: debugroom-mynavi-sample-lambda-s3event
         AccessControl: Private
         PublicAccessBlockConfiguration:
           BlockPublicAcls: True
           BlockPublicPolicy: True
           IgnorePublicAcls: True
           RestrictPublicBuckets: True
         NotificationConfiguration:                                             # (C)
           LambdaConfigurations:
             - Event: s3:ObjectCreated:*                                        # (D)
               Function:
                 Fn::ImportValue: MynaviSampleLambdaS3Event-LambdaArn           # (E)
     LambdaInvokePermission:                                                    # (F)
       Type: AWS::Lambda::Permission
       Properties:
         FunctionName:
           Fn::ImportValue: MynaviSampleLambdaS3Event-LambdaArn                 # (G)
         Principal: s3.amazonaws.com
         Action: lambda:InvokeFunction
         SourceArn: !Join
           - ""
           - - "arn:aws:s3:::"
             - "debugroom-mynavi-sample-lambda-s3event"                         # (H)

   Outputs:
     S3Bucket:
       Description: Lambda S3 bucket name
       Value: !Ref S3Bucket
       Export:
         Name: MynaviSampleLambdaS3Event-s3Bucket                               # (I)

     # omit

|br|

S3テンプレートの設定のポイントは以下の通りです。

.. list-table:: アップロードをイベントとしてLambdaファンクションを実行するS3テンプレートの詳細
   :widths: 1, 19

   * - 項番
     - 説明

   * - (A)
     - S3のリソース定義を行います。定義の要領はクラウドネイティブ基本編 `第30回 <https://news.mynavi.jp/itsearch/article/devsoft/4896>`_ と同様です。

   * - (B)
     - DependsOn属性を使って、リソースLambdaInvokePermissionの作成が完了してから、S3リソースの作成を行います。

   * - (C)
     - イベント通知の設定はNotificationConfigurationで行います。

   * - (D)
     - イベントとして、当バケットでオブジェクトが生成されたときを設定します。

   * - (E)
     - 実行するファンクションとして、上記で作成したLambdaファンクションのARNをクロススタックリファレンスで参照します。

   * - (F)
     - Lambdaの実行許可をリソースとして定義します。定義の詳細は `AWS::Lambda::Permission <https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-lambda-permission.html>`_ も参照してください。

   * - (G)
     - 対象のファンクションとして、上記で作成したLambdaファンクションのARNをクロススタックリファレンスで参照します。

   * - (H)
     - Lambdaの実行元となるARNを設定します。(A)の定義と相互参照になるため、!Ref参照は行わず、直接S3のARNを定義します。

   * - (I)
     - ダイレクトアップロードを行うアプリケーションから参照するため、Output要素として、S3のバケット名を出力しておきます。

|br|

各テンプレートを順次実行して環境を構築した後、対象のS3にファイルがアップロードされると、以下の通り、DynamoDBやS3に処理結果が残り、CloudWatch Logsにも処理のログが出力されます。

|br|

.. figure:: img/aws-s3-and-lambda/management-console-dynamodb.png

|br|

.. figure:: img/aws-s3-and-lambda/management-console-sqs.png

|br|

.. figure:: img/aws-s3-and-lambda/management-console-cloudwatch-logs.png

|br|

今回は、Lambdaファンクションの処理に必要な環境構築とLambdaのデプロイをCloudFormaitonを使って実装しました。次回はファンクション内でエラーが発生した際のCloudWatchでのイベント発生やエラーハンドリングの実装方法を解説していきます。

|br|

著者紹介
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

川畑 光平(KAWABATA Kohei) - NTTデータ

.. figure:: img/aws-s3-and-lambda/aws_361383_075.jpeg

金融機関システム業務アプリケーション開発・システム基盤担当、ソフトウェア開発自動化関連の研究開発を経て、デジタル技術関連の研究開発・推進に従事。

Red Hat Certified Engineer、Pivotal Certified Spring Professional、AWS Certified Solutions Architect Professional等の資格を持ち、アプリケーション基盤・クラウドなど様々な開発プロジェクト支援にも携わる。

`AWS Top Engineers & Ambassadors <https://aws.amazon.com/jp/partners/ambassadors/?cards-body.sort-by=item.additionalFields.ambassadorName&cards-body.sort-order=asc&cards-body.q=kawabata&cards-body.q_operator=AND>`_ 選出。

本連載記事の内容に対するご意見・ご質問は `Facebook <https://www.facebook.com/kohei.kawabata.5>`_ まで。
