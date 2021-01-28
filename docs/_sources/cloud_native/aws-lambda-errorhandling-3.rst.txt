.. include:: ../module.txt

.. _section-cloud-native-lambda-error-handling-3rd-label:

【第8回】AWS Lambdaにおけるサーバレスエラーハンドリング(3)
----------------------------------------------------------------------------------------

|br|

本連載では、AWS Lambdaを使ったサーバレス処理でのエラーハンドリング方法を解説しています。
前回は以下の赤字の矢印のスコープにおいて、同期型でビジネスエラーが発生することを想定したSpring Cloud Function 3.1以降のLambdaファンクションの実装方法やパッケージ構成を解説しました。

|br|

.. figure:: img/aws-lambda-errorhandling/errorhandling-sync-business-error.png

|br|

今回はAPI GatewayとLambda実行環境をCloudFormationを使って構築し、同期的にLambdaを呼び出してみます。

なお、API Gatewayおよび、Lambdaをマネジメントコンソール上から手動で構築する場合は、「AWSで作るクラウドネイティブアプリケーションの基本」の
`第2回 <https://news.mynavi.jp/itsearch/article/devsoft/4318>`_ と `第3回 <https://news.mynavi.jp/itsearch/article/devsoft/4321>`_ と同一です。設定内容はほぼ同一ですので、構築に必要な設定要素を理解するにはこちらも参照すると良いでしょう。
また、CloudFormationを使った環境構築の説明は割愛しますが、基本的なお作法は「AWSで実践! 基盤構築・デプロイ自動化」 `第21回 <https://news.mynavi.jp/itsearch/article/devsoft/4725>`_ 以降に解説していますので、適宜参照してください。

|br|

.. _section-cloud-native-lambda-error-handling-lambda-cloud-formation-label:

LambdaファンクションのCloudFormationテンプレート
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

AWS LambdaをCloudFormationで構築する時のテンプレートの記述要領は「AWSで実践! 基盤構築・デプロイ自動化」 `第42回 <https://news.mynavi.jp/itsearch/article/devsoft/5172>`_ でも解説済みですが、
事前にLambdaをデプロイするためにビルドされたアプリケーションパッケージを配置するS3バケットを作成しておきます。また、Lambdaのテンプレートから参照するので、バケット名やARNをエクスポートします。

|br|

.. sourcecode:: bash

   AWSTemplateFormatVersion: '2010-09-09'

   Description: S3 Bucket for Lambda function template with YAML - S3 Bucket Definition

   Resources:
     S3Bucket:
       Type: AWS::S3::Bucket
       Properties:
         BucketName: debugroom-mynavi-sample-lambda-errorhandling-for-deploy
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
         Name: debugroom-mynavi-sample-lambda-errorhandling-deploy-s3-bucket

     S3BucketArn:
       Description: Deploy S3 for Lambda bucket arn
       Value: !GetAtt S3Bucket.Arn
       Export:
         Name: debugroom-mynavi-sample-lambda-errorhandling-deploy-s3-bucket-arn

|br|

続いて、前回実装したLambdaファンクションをビルドして、このバケットにアップロードしますが、以下のようなスクリプトを実行すると簡易です。Java仮想マシンのルートディレクトリとなる
JAVA_HOME環境変数やManvenのビルドコマンドのパスなどは適宜自分の環境に合わせて書き換えてください。なお、コマンドを実行するためのAWS CLIのインストールや開発端末の認証情報の設定は、
「AWSで実践! 基盤構築・デプロイ自動化」 `第22回 <https://news.mynavi.jp/itsearch/article/devsoft/4734>`_ で記載の手順に沿って行っておくとよいでしょう。

|br|

.. sourcecode:: bash

   #!/usr/bin/env bash

   export JAVA_HOME=/usr/lib/jvm/java-11-amazon-corretto.x86_64
   bucket_name="debugroom-mynavi-sample-lambda-errorhandling-for-deploy"
   stack_name="mynavi-sample-deploy-s3-for-lambda-errorhandling"
   template_path="cloudformation/1-s3-for-lambda-deploy-cfn.yml"
   s3_objectkey="spring-cloud-3-1-lambda-function-0.0.1-SNAPSHOT-aws.jar"

   if [ "" == "`aws s3 ls | grep $bucket_name`" ]; then
       aws cloudformation deploy --stack-name ${stack_name} --template-file ${template_path} --capabilities CAPABILITY_IAM
   fi

   cd spring-cloud-3-1-lambda-function
   ./mvnw clean package
   aws s3 cp target/${s3_objectkey} s3://${bucket_name}/

|br|

続いて、Lambda実行環境のCloudFormationテンプレートを実装します。

|br|

.. sourcecode:: bash

   AWSTemplateFormatVersion: '2010-09-09'

   #omit

   Resources:
     LambdaForSyncExecuteBusinessErrorFuntion:
       Type: AWS::Lambda::Function
       Properties:
         Code:
           S3Bucket:
             Fn::ImportValue: debugroom-mynavi-sample-lambda-errorhandling-deploy-s3-bucket  #(1)
           S3Key: spring-cloud-3-1-lambda-function-0.0.1-SNAPSHOT-aws.jar
         Handler: org.springframework.cloud.function.adapter.aws.FunctionInvoker::handleRequest #(2)
         FunctionName: mynavi-sample-aws-lambda-errorhandling-sync-business-error
         Environment:
           Variables:
             SPRING_CLOUD_FUNCTION_DEFINITION: syncExecuteBusinessErrorFunction #(3)
         MemorySize: 1024
         Runtime: java11
         Timeout: 120
         Role: !GetAtt LambdaRole.Arn #(4)

     #omit

     LambdaRole:
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

     #omit

   Outputs:
     #omit

     LambdaForSyncExecuteBusinessErrorFuntion: #(5)
       Description: Sync execute Lambda function for occuring business error function.
       Value: !Ref LambdaForSyncExecuteBusinessErrorFuntion
       Export:
         Name: mynavi-sample-lambda-errorhandling-sync-execute-business-error-function-name

     LambdaForSyncExecuteBusinessErrorFuntionArn: #(6)
       Description: Sync execute Lambda function for occuring business error function.
       Value: !GetAtt LambdaForSyncExecuteBusinessErrorFuntion.Arn
       Export:
         Name: mynavi-sample-lambda-errorhandling-sync-execute-business-error-function-arn

     #omit

|br|

Lambdaテンプレートのポイントの詳細は以下の通りです。

|br|

.. list-table:: Lambda
   :widths: 1, 19

   * - 項番
     - 説明

   * - (1)
     - S3Bucketは上述したデプロイ用のS3を構築したテンプレートでエクスポートしたS3のバケット名をクロススタックリファレンス参照します。

   * - (2)
     - リクエストのハンドラとして、org.springframework.cloud.function.adapter.aws.FunctionInvokerを指定します。

   * - (3)
     - 環境変数SPRING_CLOUD_FUNCTION_DEFINITIONに、前回実装したファンクションのBean名を指定します。

   * - (4)
     - 構築するLambdaファンクションに実行に必要なロールを設定します。

   * - (5)
     - 後述するAPI Gatewayの設定時に必要なLambdaファンクション名をエクスポートしておきます。

   * - (6)
     - 後述するAPI Gatewayの設定時に必要なLambdaファンクションのARNをエクスポートしておきます。

|br|

.. _section-cloud-native-lambda-error-handling-apigateway-cloud-formation-label:

API GatewayのCloudFormationテンプレート
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

続いて、Lambdaファンクションを呼び出すAPI GatewayのCloudformationテンプレートを実装します。

|br|

.. sourcecode:: bash

   AWSTemplateFormatVersion: '2010-09-09'

   #omit

   Resources:
     ApiGatewayRestApi: #(1)
       Type: "AWS::ApiGateway::RestApi"
       Properties:
         Name: "mynavi-sample-lambda-errorhandling-rest-api"
         Description: "Mynavi sample sync execute function API"

     ApiGatewayDeployment: #(2)
       Type: "AWS::ApiGateway::Deployment"
       DependsOn:
         - ApiGatewayBusinessExceptionMethod
       Properties:
         RestApiId:
           Ref: ApiGatewayRestApi

     ApiGatewayStage: #(3)
       Type: "AWS::ApiGateway::Stage"
       Properties:
         StageName: "dev"
         Description: "dev stage"
         RestApiId:
           Ref: ApiGatewayRestApi
         DeploymentId:
           Ref: ApiGatewayDeployment

     ApiGatewayModel: #(4)
       Type: "AWS::ApiGateway::Model"
       Properties:
         RestApiId:
           Ref: ApiGatewayRestApi
         ContentType: "application/json"
         Name: SampleSchema
         Schema:
           "$schema": "http://json-schema.org/draft-04/schema#"
           title: SampleResource  #(5)
           type: object
           properties:
             message:
               type: string

     ApiGatewayBusinessExceptionResource:  #(6)
       Type: "AWS::ApiGateway::Resource"
       Properties:
         RestApiId:
           Ref: ApiGatewayRestApi
         ParentId:
           Fn::GetAtt:
             - ApiGatewayRestApi
             - RootResourceId
         PathPart: "business-exception-sample-resource"

      #omit

     ApiGatewayBusinessExceptionMethod: #(7)
       Type: "AWS::ApiGateway::Method"
       DependsOn: ApiGatewayModel
       Properties:
         RestApiId:
           Ref: ApiGatewayRestApi
         ResourceId:
           Ref: ApiGatewayBusinessExceptionResource
         HttpMethod: "GET"
         AuthorizationType: "NONE"
         Integration:
           Type: "AWS_PROXY"  #(8)
           Uri:  #(9)
             Fn::Join:
               - ""
               - - "arn:aws:apigateway"
                 - ":"
                 - Ref: AWS::Region
                 - ":"
                 - "lambda:path/2015-03-31/functions/"
                 - Fn::ImportValue: mynavi-sample-lambda-errorhandling-sync-execute-business-error-function-arn
                 - "/invocations"
           IntegrationHttpMethod: "POST"
           IntegrationResponses:
             - StatusCode: 400
               SelectionPattern: 400
             - StatusCode: 200
           PassthroughBehavior: WHEN_NO_MATCH
         MethodResponses:  #(10)
           - StatusCode: 200
             ResponseModels:
               application/json: SampleSchema
           - StatusCode: 400
             ResponseModels:
               application/json: Error

     #omit

     ApiGatewayUsagePlan: #(11)
       Type: AWS::ApiGateway::UsagePlan
       Properties:
         ApiStages:
           - ApiId:
               Ref: ApiGatewayRestApi
             Stage:
               Ref: ApiGatewayStage
         Quota:
           Limit: 100
           Period: DAY
         Throttle:
           BurstLimit: 10
           RateLimit: 2
         UsagePlanName: "SampleUsagePlan"

     BusinessErrorLambdaPermission: #(12)
       Type: "AWS::Lambda::Permission"
       Properties:
         FunctionName:
           Fn::ImportValue: mynavi-sample-lambda-errorhandling-sync-execute-business-error-function-name
         Action: "lambda:InvokeFunction"
         Principal: "apigateway.amazonaws.com"

|br|

API Gatewayテンプレートのポイントの詳細は以下の通りです。

|br|

.. list-table:: Lambda
   :widths: 1, 19

   * - 項番
     - 説明

   * - (1)
     - API GatewayでRestAPIを定義します。各プロパティの詳細は `AWS::ApiGateway::RestApi <https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-apigateway-restapi.html>`_ を参照してください。

   * - (2)
     - API GatewayでRestAPIをデプロイさせた状態で構築します。各プロパティの詳細は `AWS::ApiGateway::Deployment <https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-apigateway-deployment.html>`_ を参照してください。

   * - (3)
     - API GatewayでRestAPIをデプロイさせるためのステージを定義します。各プロパティの詳細は `AWS::ApiGateway::Stage <https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-apigateway-stage.html>`_ を参照してください。ここでステージは開発用の"dev"を定義しておきます。

   * - (4)
     - API Gateway RestAPIで返却するモデルを定義します。各プロパティの詳細は `AWS::ApiGateway::Model <https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-apigateway-model.html>`_ を参照してください。

   * - (5)
     - モデルのスキーマとして、前回実装したリソースクラスSampleResourceおよびそのプロパティを定義します。

   * - (6)
     - API Gatewayが返却するリソースやパスを定義します。各プロパティの詳細は `AWS::ApiGateway::Resource <https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-apigateway-resource.html>`_ を参照してください。

   * - (7)
     - Rest APIのメソッドを定義します。各プロパティの詳細は `AWS::ApiGateway::Method <https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-apigateway-method.html>`_ を参照してください。

   * - (8)
     - API GatewayからLambdaへリクエストを転送する際のリクエストデータの変換方法を統合モデルとして定義します。Spring Cloud Functionでは、「Lambdaプロキシ統合」である設定値"AWS_PROXY"を前提としています。なお、Lambdaプロキシ統合の詳細については、
       `AWS公式ページ「API Gateway で Lambda プロキシ統合を設定する」 <https://docs.aws.amazon.com/ja_jp/apigateway/latest/developerguide/set-up-lambda-proxy-integrations.html>`_ も参照してください。

   * - (9)
     - 実行するLambdaファンクションのURIを定義します。前節のLambdaテンプレートで出力したファンクションのARNをクロススタックリファレンスで参照し、URIをJOIN関数で構築します。

   * - (10)
     - Lambda統合オプションでLambdaから返却されたアウトプットデータをAPI Gatewayでレスポンスデータとして変換する方法を定義します。正常終了時には(5)で定義したモデルクラスを返却し、ビジネスエラーが発生する400の場合は、デフォルトでAPI Gatewayが用意しているモデルErrorスキーマを用いてマッピングします。

   * - (11)
     - RestAPIのリクエスト量を調整する使用プランを定義します。使用量プランは、APIごとにスロットリングとクォータ制限が適用されます。各プロパティの詳細は `AWS::ApiGateway::UsagePlan <https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-apigateway-usageplan.html>`_ を参照してください。

   * - (12)
     - API GatewayがLambda関数を呼び出すのに必要な権限を定義します。各プロパティの詳細は `AWS::Lambda::Permission <https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-lambda-permission.html>`_ を参照してください。

|br|

作成したテンプレートをCLIから実行すると、環境が構築されます。マネジメントコンソール上で、以下のように作成したリソースのメソッドを選択し、赤枠で囲まれたテストを選択してください。

|br|

.. figure:: img/aws-lambda-errorhandling/management-console-apigateway-resource.png

|br|

クエリ文字列を指定しない状態で「テスト」ボタンを押下すると、ステータスコード400で返却されます。何かしらの文字列を指定すると、ステータスコード200で正常応答します。※なお、初回の実行はLambdaの起動でタイムアウトになるケースもあります。

|br|

.. figure:: img/aws-lambda-errorhandling/management-console-apigateway-resource-test.png

|br|


今回は、API GatewayとLambdaをCloudFormationを使って構築し、Lambdaを同期的に呼び出してビジネスエラーを発生させ、エラーに応じてステータスコードが適切にマッピングするように設定する例を解説しました。
次回以降は、同じくAPI GatewayとLambdaを使用した同期呼び出しで、システムエラーが発生した際に、CloudWatchに出力されたエラーログを契機として、システム管理者へ通知を行う実装を含めて解説していきます。

|br|

著者紹介
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

川畑 光平(KAWABATA Kohei) - NTTデータ エグゼクティブ ITスペシャリスト ソフトウェアアーキテクト・デジタルテクノロジーストラテジスト(クラウド)

.. figure:: img/aws-s3-and-lambda/aws_361383_075.jpeg

金融機関システム業務アプリケーション開発・システム基盤担当、ソフトウェア開発自動化関連の研究開発を経て、デジタル技術関連の研究開発・推進に従事。

Red Hat Certified Engineer、Pivotal Certified Spring Professional、AWS Certified Solutions Architect Professional等の資格を持ち、アプリケーション基盤・クラウドなど様々な開発プロジェクト支援にも携わる。

`AWS Top Engineers & Ambassadors <https://aws.amazon.com/jp/partners/ambassadors/?cards-body.sort-by=item.additionalFields.ambassadorName&cards-body.sort-order=asc&cards-body.q=kawabata&cards-body.q_operator=AND>`_ 選出。

本連載記事の内容に対するご意見・ご質問は `Facebook <https://www.facebook.com/kohei.kawabata.5>`_ まで。
