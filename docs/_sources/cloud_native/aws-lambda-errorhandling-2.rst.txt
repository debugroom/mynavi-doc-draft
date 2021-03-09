.. include:: ../module.txt

.. _section-cloud-native-lambda-error-handling-2nd-label:

【第7回】AWS Lambdaにおけるサーバレスエラーハンドリング(2)
----------------------------------------------------------------------------------------

|br|

本連載では、以下の図のようなAWS Lambdaを使ったサーバレス処理でのエラーハンドリング方法を解説しています。

|br|

.. figure:: img/aws-lambda-errorhandling/errorhandling-overview.png

|br|



前回は、Lambdaファンクションのエラーハンドリングの特徴と代表的なパターン、エラーの概念・対処の考え方を説明しました。
今回からはAPI GatewayとLambdaおよびSpringCloudFuntionを使って構築したLambdaを同期的に呼び出した場合に発生したエラーの取り扱い方や環境構築方法を解説します。

なお、Lambdaファンクションは以下のランタイム・ライブラリを使って実装してくものとします。

|br|


.. list-table::
   :widths: 5, 5

   * - 動作対象
     - バージョン

   * - Java
     - 11

   * - Spring Boot
     - 2.3.8.RELEASE

   * - Spring Cloud Function
     - 3.0.11.RELEASE

   * - Spring Cloud AWS
     - 2.2.5.RELEASE

   * - AmazonSDK Lambda Java event
     - 2.0.2

   * - AmazonSDK Lambda Java core
     - 1.1.0

   * - AmazonSDK Java
     - 1.11.923

|br|

.. _section-cloud-native-lambda-error-handling-spring-cloud-function-3-1-label:

Spring Cloud Function 3.1以降を使った、AWS Lambdaの同期ビジネスエラー実装とライブラリ・パッケージの構成
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

まず最初のケースではAWS Lambdaで同期呼び出しのビジネスエラーについて考えます。同期型のビジネスエラーはLambdaファンクション中で発生したエラーを、
ユーザに通知すれば良いので、下記のイメージの赤字の矢印がスコープとなります。ここのLambdaファンクションの実装を進めていきます。

|br|

.. figure:: img/aws-lambda-errorhandling/errorhandling-sync-business-error.png

|br|

これまでの連載(`「基本編第1回」 <https://news.mynavi.jp/itsearch/article/devsoft/4316>`_ や `「応用編第4回」 <https://news.mynavi.jp/itsearch/article/devsoft/5082>`_ )でも、
Spring Cloud Functionを使用したLambdaファンクションの実装の解説を行ってきましたが、Spring Cloud Functionがバージョン3.1以降、フレームワークの使い方に少し大きめの修正があり、
org.springframework.cloud.function.adapter.aws.FunctionInvokerを使う方法にシフトしています。
そこで、今回はこのFunctionInvokerを使ったプログラミングモデルでの同期エラー実装と、アプリケーションのパッケージ構成について、3.1より前のバージョンとの差分について触れながら解説を進めていきたいと思います。

また、実際に作成したアプリケーションは `GitHub <https://github.com/debugroom/mynavi-sample-aws-lambda-errorhandling>`_ 上にコミットしています。
以降記載されているソースコードで、インポート文など本質的でない記述を省略している部分がありますので、実行コードを作成する場合は、必要に応じて適宜GitHubソースコードも参照してください。

アプリケーションの実装に必要なライブラリはこれまでと同様、以下の通りですが、後の非同期での実装を含め、SQSやSystemsManager ParameterStoreへアクセスするので、Spring Cloud AWSとSDKのライブラリのライブラリを追加します。
また、Mattermostのようなチャットコミュニケーションツールのチャネルにメッセージを通知するためにWebClientを使用するので、spring-boot-starter-webfluxを追加します。
各マネージドサービスなどのエンドポイント情報などはCloudFormationのスタック経由から取得しますが、スタックでエクスポートした名称を保持しておくためのプロパティクラスを使用するために、
spring-boot-configuration-processorを含めておいてください。

|br|

.. sourcecode:: xml

   <dependency>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter</artifactId>
   </dependency>
   <dependency>
       <groupId>org.springframework.cloud</groupId>
       <artifactId>spring-cloud-function-context</artifactId>
   </dependency>
   <dependency>
       <groupId>org.springframework.cloud</groupId>
       <artifactId>spring-cloud-function-adapter-aws</artifactId>
   </dependency>
   <dependency>
       <groupId>org.springframework.cloud</groupId>
       <artifactId>spring-cloud-starter-aws</artifactId>
   </dependency>
   <dependency>
       <groupId>org.springframework.cloud</groupId>
       <artifactId>spring-cloud-starter-aws-messaging</artifactId>
   </dependency>
   <dependency>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter-webflux</artifactId>
   </dependency>
   <dependency>
       <groupId>com.amazonaws</groupId>
       <artifactId>aws-lambda-java-events</artifactId>
       <version>2.0.2</version>
   </dependency>
   <dependency>
       <groupId>com.amazonaws</groupId>
       <artifactId>aws-lambda-java-core</artifactId>
       <version>1.1.0</version>
   </dependency>
   <dependency>
       <groupId>com.amazonaws</groupId>
       <artifactId>aws-java-sdk-ssm</artifactId>
       <version>1.11.923</version>
   </dependency>
   <dependency>
       <groupId>com.amazonaws</groupId>
       <artifactId>aws-java-sdk-core</artifactId>
       <version>1.11.923</version>
   </dependency>
   <dependency>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-configuration-processor</artifactId>
       <optional>true</optional>
   </dependency>

|br|

さて、Spring Cloud Functionを使用した使用方法ですが、これまで使用してきた、リクエストをハンドリングするSpringBootRequestHandlerクラスは非推奨となり、ハンドラクラスを実装せずに、
FunctionInvokerを使って、ファンクションが実行されます。FunctionInvokerはLambdaのJava言語実装で必要になるhandleメソッドが実装されており、開発者はファンクションクラスを作成するだけですみます。
FunctionInvokerは、API Gatewayをはじめ、S3イベントやKinesis、SNSといった複数のAWSマネージドサービスからのリクエストをサポートしていますが、 `「AWSで実践！基盤・デプロイ自動化 第41回」 <https://news.mynavi.jp/itsearch/article/devsoft/5135>`_ で解説した
CloudFormationのLambda-backedカスタムリソースのように、対応していないマネージドサービスのリクエストがいくつかあるので、その場合はFunctionInvokerを継承して、必要なリクエストハンドリング処理をオーバーライドして実装するようにしましょう。
FunctionInvokerから実行される、同期的なビジネスエラーハンリングを実装したファンクションクラスは以下の通りです。

|br|

.. sourcecode:: java

   package org.debugroom.mynavi.sample.aws.lambda.errorhandling.app.function;

   import java.util.function.Function;

   import org.springframework.messaging.Message;
   import org.springframework.messaging.MessageHeaders;
   import org.springframework.messaging.support.MessageBuilder;

   import org.debugroom.mynavi.sample.aws.lambda.errorhandling.app.model.SampleResource;
   import org.debugroom.mynavi.sample.aws.lambda.errorhandling.common.exception.BusinessException;
   import org.debugroom.mynavi.sample.aws.lambda.errorhandling.domain.service.CheckParameterService;

   import lombok.extern.slf4j.Slf4j;

   @Slf4j
   public class SyncExecuteBusinessErrorFunction implements Function<Map<String, Object>, Message<SampleResource>> {
                                                                                                 // (1)

       @Autowired
       CheckParameterService checkParameterService; //(2)

       @Override
       public Message<SampleResource> apply(Map<String, Object> stringObjectMap) { //(3)

           log.info(this.getClass().getName() + " has started!");
           log.info("[Input]" + stringObjectMap.toString());

           Map<String, Object> queryStringParameters = (Map)stringObjectMap.get("queryStringParameters");
           try{
               checkParameterService.execute(queryStringParameters); //(4)
           }catch (BusinessException e){ //(5)
               Map<String, Object> headers = new HashMap<>();
               headers.put("statusCode", 400); //(6)
               MessageHeaders messageHeaders = new MessageHeaders(headers);
               return MessageBuilder.createMessage(
                    SampleResource.builder().message("No query string parameters.").build(), messageHeaders); //(7)
           }
           return MessageBuilder.withPayload(
                SampleResource.builder().message(queryStringParameters.toString()).build()).build(); //(8)
       }

   }

|br|

各コードの詳細は以下の通りです。

|br|

.. list-table:: ビジネスエラーが発生するファンクションクラスの実装コードの詳細
   :widths: 1, 19

   * - 項番
     - 説明

   * - (1)
     - java.util.Functionクラスを実装します。インプットデータの型としてMapクラスを、アウトプット型として、返却するクラスオブジェクトSampleResourceを型パラメータに指定したorg.springframework.messaging.Messageを指定します。

   * - (2)
     - ビジネス例外(アプリケーションのビジネスロジックとして想定される例外)が発生するサービスクラスとして、リクエストパラメータの有無をチェックし、ビジネス例外クラスをスローするCheckParameterServiceを使用します。

   * - (3)
     - FunctionInvokerクラスから、オーバーライドしたapply()メソッドがコールされます。リクエストパラメータおよびメタデータをMap型で受け取れます。

   * - (4)
     - インプットMapオブジェクトからクエリ文字列パラメータを取得し、チェックします。ここでは、クエリ文字列がないとビジネス例外をスローするようにしています(クエリ文字列があれば何もしません)。

   * - (5)
     - ビジネス例外が発生した場合、例外をキャッチして、呼び出し元のユーザにエラー内容を通知する処理を実装します。

   * - (6)
     - SpringCloud Funtion3.1以降では、MessageHeaderに"statusCode"を設定して返却すると、API Gatewayから返却するレスポンスのステータスコードを指定することができます。ここでは、送信されたリクエストが不適切なことを示す400を設定しています。

   * - (7)
     - ビジネスエラー発生時、返却するメッセージのボディ部に設定するSampleResourceオブジェクトとして、メッセージを指定したMessageクラスを返却します。

   * - (8)
     - 正常終了時、返却するメッセージのボディ部に設定するSampleResourceオブジェクトとして、メッセージを指定したMessageクラスを返却します。

|br|

上記の実装を見て、気付いた方がいるかもしれませんが、当クラスの責務としてリクエストからパラメータを取り出し、ビジネス処理を実行して、エラーであればステータスコードを設定するといった、
Spring MVCでいうところのControllerクラスに該当する処理が実装されています。ファンクションクラス自体にビジネス処理を実装するのもシンプル化の観点から悪くない方法ですが、
例外ハンドリングの実装で複雑化することや単体テストの観点から、ファンクションクラスはコントローラがもつ単一責務を負うものとして実装した方がコードの見通しがすっきりします。
そこで、本連載では、 `「AWSで実践！基盤・デプロイ自動化 第4回マイクロサービスにおけるテスト自動化とテスト戦略」 <https://news.mynavi.jp/itsearch/article/devsoft/4475>`_ で解説したパッケージ構成に則り、以下のようにLambdaファンクションのパッケージを構成します。

|br|

.. sourcecode:: bash

   [spring-cloud-3-1-lambda-function]
     └src
       └main
         ├java
         │ └org
         │   └debugroom
         │     └mynavi
         │       └sample
         │         └aws
         │           └lambda
         │             └errorhandling
         │               └app                                         ... アプリケーション層のパッケージ
         │               │ ├funtion                                   ... ファンクションクラスのパッケージ
         │               │ │ ├・・・
         │               │ | └SyncExecuteBusinessErrorFuntion.java    ... FunctionInvokerから実行されるファンクションクラス
         │               │ ├model                                     ... アプリケーション層モデルクラスパッケージ
         │               │ | └XxxxxResource.java                      ... Lambdaファンクションが返却するResourceクラス
         │               │ ├ServiceProperties                         ... マネージドサービスのエンドポイントなど必要なプロパティクラスを保持するクラス
         │               │ └CloudFormationStackResolver.java          ... CloudFormationからスタック情報を取得するためのユーティリティクラス
         │               ├common                                      ... レイヤ共通で使用されるパッケージ
         │               │ └exception                                 ... 例外クラス用のパッケージ
         │               │   ├BusinessException.java                  ... ビジネスエラーを表す例外クラス
         │               │   └SystemException.java                    ... システムエラーを表す例外クラス
         │               ├domain                                      ... DomainConfigでコンポーネントスキャンの対象とするドメイン層のパッケージ
         │               │ ├model                                     ... ドメイン層のモデルクラスパッケージ
         │               │ | └Xxxxx.java                              ... ドメイン層のモデルクラス
         │               │ ├repository                                ... レポジトリクラスパッケージ
         │               │ │ ├XxxxxRepository.java                    ... レポジトリインタフェースクラス
         │               │ │ └XxxxxRepositoryYYYImpl.java             ... マネージドサービスや外部サーバへアクセスするWebClient等を使用したレポジトリ実装クラス
         │               │ └service                                   ... サービスクラスパッケージ
         │               │   ├CheckParameterService.java              ... パラメータをチェックし、ビジネス例外をスローするサービスクラス
         │               │   ├・・・
         │               │   └CreateSystemErrorService.java           ... システム例外をスローするサービスクラス
         │               └config                                      ... 設定クラス用のパッケージ
         │                   ├App.java                                ... アプリケーション起動クラス
         │                   ├DomainConfig.java                       ... ドメイン層に関する設定クラス
         │                   ├S3Config.java                           ... S3に関する設定クラス
         │                   └SqsConfig.java                          ... SQSに関する設定クラス
         └resources
           ├application.yml                                        ... アプリケーション設定ファイル
           ├messages.properties                                    ... デフォルトメッセージ定義ファイル
           └messages_ja.properties                                 ... ロケールがjaの際に有効になるメッセージ定義ファイル


|br|

また、アプリケーションの設定ファイルを以下の通り設定しておきます。

|br|

.. sourcecode:: bash

   cloud:
     aws:
       credentials:
         profileName:
         instanceProfile: false
       stack:
         auto: false
       region:
         auto: false
         static: ap-northeast-1                                                 #(A)
   spring:
     main:
       web-application-type: none                                               #(B)
     cloud:
       function:
         scan:
           packages: org.debugroom.mynavi.sample.aws.lambda.errorhandling.app.function
                                                                                #(C)
   logging:
     level:
       com:
         amazonaws:
           util:
             EC2MetadataUtils : error                                           #(D)
   service:
     systems-manager-parameter-store:
       mattermost:
         in-comming-webhook: sample-aws-lambda-errorhandling-mattermost-webhook-url
                                                                                #(E)

|br|

.. list-table:: アプリケーション設定ファイル
   :widths: 1, 19

   * - 項番
     - 説明

   * - (A)
     - Spring Cloud AWSで起動時に実行されるインスタンスプロファイル、実行リージョン、スタック情報の自動取得をオフにしておきます。

   * - (B)
     - WebClientの利用で追加したspring-boot-starter-webfluxがデフォルトだとファンクション実行時に組み込みアプリケーションサーバを起動してしまうため、設定をオフにしておきます。

   * - (C)
     - ファンクションクラスがあるパッケージを指定します。

   * - (D)
     - Lambdaで実行されるものの、Spring Cloud AWSがファンクション起動時にEC2インスタンスメタデータを取得しようとして失敗するため、エラーが出力されても問題なように設定しておきます。

   * - (E)
     - SystemsManager ParameterStoreからMattermostのWebhookURLを取得するためのキー名を指定します。MattermostのWebhookについては次回以降改めて説明します。

|br|


今回はSpring Cloud Function 3.1以降のLambdaファンクションの実装方法やパッケージ構成を解説しました。次回は、API GatewayとLambdaをCloudFormationを使って構築し、Lambdaを同期的に呼び出す環境構築方法を解説します。

|br|

著者紹介
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

川畑 光平(KAWABATA Kohei) - NTTデータ エグゼクティブ ITスペシャリスト ソフトウェアアーキテクト・デジタルテクノロジーストラテジスト(クラウド)

.. figure:: img/aws-s3-and-lambda/aws_361383_075.jpeg

金融機関システム業務アプリケーション開発・システム基盤担当、ソフトウェア開発自動化関連の研究開発を経て、デジタル技術関連の研究開発・推進に従事。

Red Hat Certified Engineer、Pivotal Certified Spring Professional、AWS Certified Solutions Architect Professional等の資格を持ち、アプリケーション基盤・クラウドなど様々な開発プロジェクト支援にも携わる。

`AWS Top Engineers & Ambassadors <https://aws.amazon.com/jp/partners/ambassadors/?cards-body.sort-by=item.additionalFields.ambassadorName&cards-body.sort-order=asc&cards-body.q=kawabata&cards-body.q_operator=AND>`_ 選出。

本連載記事の内容に対するご意見・ご質問は `Facebook <https://www.facebook.com/kohei.kawabata.5>`_ まで。
