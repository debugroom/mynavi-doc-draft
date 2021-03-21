.. include:: ../module.txt

.. _section-aws-microservice-cognito-oauth2-login-7-label:

【第18回】Amazon Cognito + Spring Sercurityを使ったOAuth2 Loginの実装(7)
===============================================================================================

|br|

前回から、以下のイメージのようにOAuth2 Loginをベースとしたアーキテクチャを想定した環境構築を進めています。

|br|

.. figure:: img/cognito-oauth2-login/oauth2-login-flow.png

|br|

前回は、Cognitoへユーザ登録し、サインアップステータスを変更するLambdaファンクションを実装しました。
今回は引き続き、作成したLambdaファンクションをカスタムリソースとして登録し、実行するCloudFormationの実装方法を解説していきます。

なお、実際のソースコードは `GitHub <https://github.com/debugroom/mynavi-sample-aws-microservice/tree/feature_4_cognito-oauth2-login>`_ 上にコミットしています。以降のソースコードでは本質的でない記述を一部省略しているので、実行コードを作成する場合は、必要に応じて適宜GitHub上のソースコードも参照してください。

|br|

LambdaファンクションのCloudFormationテンプレートの実装
----------------------------------------------------------------------------------

|br|

まずは前回作成したLambdaファンクションをビルドしたパッケージを配置するS3バケットを作成するためのCloudFormationテンプレートを作成します。簡単なので説明は省略します。

|br|

.. sourcecode:: none

   AWSTemplateFormatVersion: '2010-09-09'

   Parameters:
     VPCName:
       Description: Target VPC Stack Name
       Type: String
       MinLength: 1
       MaxLength: 255
       AllowedPattern: ^[a-zA-Z][-a-zA-Z0-9]*$
       Default: mynavi-sample-microservice-vpc

   Resources:
     S3Bucket:
       Type: AWS::S3::Bucket
       Properties:
         BucketName: debugroom-mynavi-microservice-cfn-lambda-bucket
         AccessControl: "Private"
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
         Name: !Sub ${VPCName}-Lambda-S3Bucket

     S3BucketArn:
       Description: S3 for Lambda bucket arn
       Value: !GetAtt S3Bucket.Arn
       Export:
         Name: !Sub ${VPCName}-Lambda-S3Bucket-Arn

|br|

下記のスクリプトのように、作成したテンプレートを実行し、LambdaファンクションをビルドしてS3に配置します。

|br|

.. sourcecode:: bash

   #!/usr/bin/env bash

   export JAVA_HOME=/usr/lib/jvm/java-11-amazon-corretto.x86_64
   bucket_name="debugroom-mynavi-microservice-cfn-lambda-bucket"
   stack_name="mynavi-microservice-s3-lambda"
   template_path="cloudformation/2-s3-for-lambda-deploy-cfn.yml"
   s3_objectkey="cognito-init-lambda-0.0.1-SNAPSHOT-aws.jar"

   if [ "" == "`aws s3 ls | grep $bucket_name`" ]; then
       aws cloudformation deploy --stack-name ${stack_name} --template-file ${template_path} --capabilities CAPABILITY_IAM
   fi

   cd cognito-init-lambda
   ./mvnw clean package
   aws s3 cp target/${s3_objectkey} s3://${bucket_name}/

|br|

続いて、Lambdaファンクションを定義するCloudFormationテンプレートを作成します。


|br|

.. sourcecode:: none

   AWSTemplateFormatVersion: '2010-09-09'

   # omit

   Resources:
     LambdaForChangeCognitoUserStatusFunction: #(A)
       Type: AWS::Lambda::Function
       Properties:
         Code:
           S3Bucket:
             Fn::ImportValue: !Sub ${VPCName}-Lambda-S3Bucket #(B)
           S3Key: cognito-init-lambda-0.0.1-SNAPSHOT-aws.jar #(C)
         Handler: org.debugroom.mynavi.sample.aws.microservice.lambda.app.handler.CloudFormationTriggerHandler::handleRequest #(D)
         FunctionName: mynavi-microservice-cfn-cognito-user-status-change-function #(E)
         Environment:
           Variables:
             SPRING_CLOUD_FUNCTION_DEFINITION: changeCognitoUserStatusFunction #(F)
         MemorySize: 1024
         Runtime: java11
         Timeout: 120
         Role: !GetAtt LambdaRole.Arn

     LambdaForAddClientSecretToParameterStoreFunction: #(G)
       Type: AWS::Lambda::Function
       Properties:
         Code:
           S3Bucket:
             Fn::ImportValue: !Sub ${VPCName}-Lambda-S3Bucket
           S3Key: cognito-init-lambda-0.0.1-SNAPSHOT-aws.jar
         Handler: org.debugroom.mynavi.sample.aws.microservice.lambda.app.handler.CloudFormationTriggerHandler::handleRequest
         FunctionName: mynavi-microservice-cfn-cognito-add-client-secret-function
         Environment:
           Variables:
             SPRING_CLOUD_FUNCTION_DEFINITION: addClientSecretToParameterStoreFunction
         MemorySize: 1024
         Runtime: java11
         Timeout: 120
         Role: !GetAtt LambdaRole.Arn

     LambdaForAddCognitoUserFunction: #(H)
       Type: AWS::Lambda::Function
       Properties:
         Code:
           S3Bucket:
             Fn::ImportValue: !Sub ${VPCName}-Lambda-S3Bucket
           S3Key: cognito-init-lambda-0.0.1-SNAPSHOT-aws.jar
         Handler: org.debugroom.mynavi.sample.aws.microservice.lambda.app.handler.CloudFormationTriggerHandler::handleRequest
         FunctionName: mynavi-microservice-cfn-cognito-add-cognito-user-function
         Environment:
           Variables:
             SPRING_CLOUD_FUNCTION_DEFINITION: addCognitoUserFunction
         MemorySize: 1024
         Runtime: java11
         Timeout: 120
         Role: !GetAtt LambdaRole.Arn

     LambdaRole: #(I)
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
           - arn:aws:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole

     CloudFormationAccessPolicy: #(J)
       Type: AWS::IAM::Policy
       Properties:
         PolicyName: mynavi-microservice-cfn-cognito-lambda-CloudFormationAccessPolicy
         PolicyDocument:
           Statement:
             - Effect: Allow
               Action:
                 - "cloudformation:*"
               Resource: "*"
         Roles:
           - !Ref LambdaRole

     SSMAccessPolicy: #(K)
       Type: AWS::IAM::Policy
       Properties:
         PolicyName: mynavi-microservice-cfn-cognito-lambda-SSMAccessPolicy
         PolicyDocument:
           Statement:
             - Effect: Allow
               Action:
             # omit
         Roles:
           - !Ref LambdaRole

     CognitoPowerUserPolicy: #(L)
       Type: AWS::IAM::Policy
       Properties:
         PolicyName: mynavi-microservice-cfn-cognito-lambda-CognitoccessPolicy
         PolicyDocument:
           Statement:
             - Effect: Allow
               Action:
                 - "cognito-identity:*"
               # omit
         Roles:
           - !Ref LambdaRole

   Outputs: (M)
     LambdaForChangeCognitoUserStatusFunction:
       Description: Lambda function for changing cognito user status function.
       Value: !GetAtt LambdaForChangeCognitoUserStatusFunction.Arn
       Export:
         Name: !Sub ${VPCName}-LamdaForChangeCognitoUserStatusFunctionArn

     LambdaForAddClientSecretToParameterStoreFunction:
       Description: Lambda function for addtion of client sercret to parameter store function.
       Value: !GetAtt LambdaForAddClientSecretToParameterStoreFunction.Arn
       Export:
         Name: !Sub ${VPCName}-LamdaForAddClientSecretToParameterStoreFunctionArn

     LambdaForAddCognitoUserFunction:
       Description: Lambda function for addtion of cognito user function.
       Value: !GetAtt LambdaForAddCognitoUserFunction.Arn
       Export:
         Name: !Sub ${VPCName}-LamdaForAddCognitoUserFunctionArn

|br|

.. list-table::
   :widths: 1, 9

   * - 項番
     - 説明

   * - A
     - ユーザのサインアップステータスを変更するLambdaファンクションを定義します。

   * - B
     - 上記で作成したS3バケット名をクロススタックリファレンスで参照します。

   * - C
     - ビルドしたLambdaファンクションのファイル名を指定します。

   * - D
     - 前々回実装したハンドラクラスを完全修飾クラス名で指定します。

   * - E
     - ファンクション名を半角英数字64文字以内で指定します。

   * - F
     - 環境変数SPRING_CLOUD_FUNCTION_DEFINITIONに、実行するファンクションクラスのBean名を指定します。

   * - G
     - アプリクライアントのクライアントシークレットをParameter Storeに設定するLambdaファンクションを定義します。ファンクション名と環境変数SPRING_CLOUD_FUNCTION_DEFINITION以外は(A)と同一の定義で問題ありません。環境変数で実行するファンクションクラスを切り替えることができます。

   * - H
     - Cognitoユーザプールにユーザを登録するLambdaファンクションを定義します。ファンクション名と環境変数SPRING_CLOUD_FUNCTION_DEFINITION以外は(A)と同一の定義で問題ありません。環境変数で実行するファンクションクラスを切り替えることができます。

   * - I
     - Lambdaファンクションにアタッチするロールを定義します

   * - J
     - (I)のロールにアタッチするポリシーを定義します。LambdaファンクションでCloudFormationのスタック情報にアクセスするため権限を付与しておきます。

   * - K
     - (I)のロールにアタッチするポリシーを定義します。LambdaファンクションでSystems Manager Parameter Storeにアクセスするため権限を付与しておきます。

   * - L
     - (I)のロールにアタッチするポリシーを定義します。LambdaファンクションでCognitoにアクセスするため権限を付与しておきます。

   * - M
     - CloudFormationのカスタムリソースで参照するため、ARNをOutput要素で出力しておきます。

|br|

最後に、このLambdaファンクションを実行するための、カスタムリソースを定義します。このテンプレートを実行すると、Lambdaファンクションが実行されます。先に実行する必要があるファンクションはDependsOn属性を指定しておきます。

|br|

.. sourcecode:: none

   AWSTemplateFormatVersion: '2010-09-09'

   # omit

   Resources:
     LambdaForChangeCognitoUserStatusFunctionTrigger:
       Type: Custom::LambdaTrigger
       DependsOn: LambdaForAddCognitoUserFunctionTrigger
       Properties:
         ServiceToken:
           Fn::ImportValue: !Sub ${VPCName}-LamdaForChangeCognitoUserStatusFunctionArn
         Region: !Ref "AWS::Region"


     LambdaForAddCognitoUserFunctionTrigger:
       Type: Custom::LambdaTrigger
       DependsOn: LambdaForAddClientSecretToParameterStoreFunctionTrigger
       Properties:
         ServiceToken:
           Fn::ImportValue: !Sub ${VPCName}-LamdaForAddCognitoUserFunctionArn
         Region: !Ref "AWS::Region"

     LambdaForAddClientSecretToParameterStoreFunctionTrigger:
       Type: Custom::LambdaTrigger
       Properties:
         ServiceToken:
           Fn::ImportValue: !Sub ${VPCName}-LamdaForAddClientSecretToParameterStoreFunctionArn
         Region: !Ref "AWS::Region"

|br|

テンプレートを実行すると、ユーザプールにステータスが「CONFIRMED」となるユーザが作成された状態となります。

|br|

.. figure:: img/cognito-oauth2-login/management-console-cognito-confirm-user.png

|br|


今回は、Lambdaファンクションをカスタムリソースとして実行するCloudFormationテンプレートを作成し、実行してみました。
次回以降は、いよいよSpring Securityを使ってOAuth2 Loginを行うアプリケーション実装の解説を進めていきます。

|br|

著者紹介
------------------------------------------------------------------

川畑 光平(KAWABATA Kohei) - NTTデータ エグゼクティブ ITスペシャリスト ソフトウェアアーキテクト・デジタルテクノロジーストラテジスト(クラウド)

.. figure:: img/overview/aws_361383_075.jpeg

金融機関システム業務アプリケーション開発・システム基盤担当、ソフトウェア開発自動化関連の研究開発を経て、デジタル技術関連の研究開発・推進に従事。

Red Hat Certified Engineer、Pivotal Certified Spring Professional、AWS Certified Solutions Architect Professional等の資格を持ち、アプリケーション基盤・クラウドなど様々な開発プロジェクト支援にも携わる。

`AWS Top Engineers & Ambassadors <https://aws.amazon.com/jp/partners/ambassadors/?cards-body.sort-by=item.additionalFields.ambassadorName&cards-body.sort-order=asc&cards-body.q=kawabata&cards-body.q_operator=AND>`_ 選出。

本連載記事の内容に対するご意見・ご質問は `Facebook <https://www.facebook.com/kohei.kawabata.5>`_ まで。
