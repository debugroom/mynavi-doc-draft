.. include:: ../module.txt

.. _section-automation-infra-devops-cloudformation-14-label:

基盤・デプロイ自動化実践
==================================================================

マイクロサービスアーキテクチャの基盤・デプロイ自動化
-------------------------------------------------------------------------------------------------------------------------------------

|br|

本連載では、以下のイメージの構成にあるAWSリソース基盤自動化環境の構築を実践しています。

|br|

.. figure:: img/automation_infra_devops_cloudformation/cloudformation-scope.png

|br|

前回は、テンプレートの親となるネストテンプレートを作成し、複数のテンプレートに定義したリソースを一括構築しました。続く今回は、作成したスタックの情報をSpring Cloud AWSを用いてアプリケーションから参照する実装を紹介します。
実際のソースコードは `GitHub <https://github.com/debugroom/mynavi-sample-cloudformation>`_ 上にコミットしています。
ソースコード中で本質的でない記述を一部省略しているので、実行コードを作成する場合は、必要に応じて適宜GitHub上のソースコードも参照してください。

|br|

.. _section-cloudformation-spring-cloud-aws-support-label:

Spring Cloud AWSのCloudFormationサポート
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

`クラウドネイティブ第12回 <https://news.mynavi.jp/itsearch/article/devsoft/4426>`_ で、Spring Cloud AWSがCloudFormationで定義した論理名を元にリソース情報へアクセスする機能をサポートしていることを紹介しました。
これまでの連載の中で、実装してきたアプリケーションでは以下のようにAWSリソースを利用しています。

* ALBを経由したバックエンドサービスの呼び出し(ALBのDNSをRestTemplateに指定)
* エンドポイントを指定したRDSアクセス
* サービスエンドポイントを指定したDynamoDBアクセス
* エンドポイントを指定したElastiCacheアクセス
* S3バケットへのアクセス
* サービスエンドポイントを指定したSQSへのキュー送信、ポーリングによる取得

アプリケーションではこれまで設定ファイルもしくは環境変数への指定でリソースアクセスに必要な設定を行ってきましたが、今回はSpring Cloud AWSのCloudFormationサポート機能を使って、必要な設定を行ってみましょう。
必要に応じて、`Spring Cloud AWSの公式リファレンス <https://cloud.spring.io/spring-cloud-aws/reference/html/>`_ も参照してください。

|br|

.. note:: これまでの連載ではSpring Cloud AWSを使用する場合、以下の通り、スタックの自動検出を全てOFFにした状態で解説を進めてきました(application.ymlでstack.aws.stack.autoをfalseに設定)。

   .. sourcecode:: none

      cloud:
        aws:
          stack:
            auto: false

   今回からの解説ではこちらの設定を有効化した状態での設定方法の解説を進めていきます。

|br|

Spring Cloud AWSからアクセスするCloudFormationスタックおよびリソースは前回複数のリソースを一括定義した親テンプレートのNestedStackを指定して取得します。なお、各ライブラリの動作環境は以下のバージョンで実施しています。

|br|

.. list-table::
   :widths: 5, 5

   * - 動作対象
     - バージョン

   * - Java
     - 1.8

   * - Spring Boot
     - 2.2.4.RELEASE

   * - Spring Cloud AWS
     - 2.2.1.RELEASE

   * - Spring Data JPA
     - 2.2.4.RELEASE

   * - Spring Data DynamoDB
     - 5.2.1

   * - Spring Session Data Redis
     - 2.2.0.RELEASE

|br|

アプリケーションはFrontendサブネットに配置するECSコンテナ上のWebアプリケーションと、Backendサブネットに配置するECSコンテナ上のサービスアプリケーションの２種類あります。それぞれアプリケーションから参照するAWSリソースは以下の通りです。

|br|

.. list-table::
   :widths: 5, 5, 5

   * - AWSリソース
     - Frontend WebApp
     - Backend Service

   * - ALB
     - ○
     -

   * - RDS
     -
     - ○

   * - DynamoDB
     -
     - ○

   * - ElastiCache
     - ○
     -

   * - S3
     - ○
     -

   * - SQS
     - ○
     - ○

|br|

ただし、開発環境ではローカル端末内に2つのアプリケーションを起動するため、ALBの参照は行わないものとします。それでは、まず最初にBackend Serviceから、Spring Cloud AWSを使って CloudFormationのスタックから情報を取得する設定を実装してみましょう。

|br|

.. _section-cloudformation-backend-service-sample-label:

Backend ServiceにおけるCloudFormationスタック情報を利用した設定
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

まず、Mavenプロジェクトのpom.xmlで、spring-boot-starter-web、spring-boot-starter-data-jpa、spring-cloud-starter-awsのライブラリを定義します。
また、RDSへアクセスするためにはspring-cloud-starter-aws-jdbcを、DynamoDBへのアクセスにはspring-data-dynamodb、SQSのアクセスにはspring-cloud-starter-aws-messagingを合わせて追加する必要があります。
なお、PostgreSQLのドライバや、エンティティクラスの実装を簡易化するためにLombokライブラリを追加しておきましょう。

|br|

.. sourcecode:: xml

   <dependencies>
     <dependency>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter-web</artifactId>
     </dependency>
     <dependency>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter-data-jpa</artifactId>
     </dependency>
     <dependency>
       <groupId>org.springframework.cloud</groupId>
       <artifactId>spring-cloud-starter-aws</artifactId>
     </dependency>
     <dependency>
       <groupId>org.springframework.cloud</groupId>
       <artifactId>spring-cloud-starter-aws-jdbc</artifactId>
     </dependency>
     <dependency>
       <groupId>org.springframework.cloud</groupId>
       <artifactId>spring-cloud-starter-aws-messaging</artifactId>
     </dependency>
     <dependency>
       <groupId>io.github.boostchicken</groupId>
       <artifactId>spring-data-dynamodb</artifactId>
       <version>5.2.1</version>
     </dependency>
     <dependency>
       <groupId>org.postgresql</groupId>
       <artifactId>postgresql</artifactId>
       <scope>runtime</scope>
     </dependency>
     <dependency>
       <groupId>org.projectlombok</groupId>
       <artifactId>lombok</artifactId>
       <optional>true</optional>
     </dependency>

   </dependencies>

|br|

Backend Serviceアプリケーションでは以下のような設定クラスの構成で実装します。以前の連載で実装した内容から差分があるものを中心に解説します。

.. list-table:: アプリケーション
   :widths: 2, 7, 1

   * - コンポーネント
     - 説明
     - 必須

   * - App
     - SpringBootアプリケーションを実行する起動クラス。以前の連載と設定は変わらないため説明は省略。
     - ◯

   * - DomainConfig
     - サービスレイヤの設定を行うクラス。以前の連載と設定は変わらないため説明は省略。
     -

   * - DynamoDBConfig
     - DynamoDBに関する設定クラス
     - ◯

   * - JpaConfig
     - JPAに関する設定クラス。以前の連載と設定は変わらないため説明は省略。
     - ◯

   * - MvcConfig
     - Webサービスに関する設定クラス。以前の連載と設定は変わらないため説明は省略。
     - ◯

   * - SQSConfig
     - SQSに関する設定クラス
     - ◯

   * - DevConfig
     - 開発環境に依存する内容の設定クラス。RDS、DynamoDBで定義が必要なもので開発環境に依存する部分を切り出して設定する。
     - ◯

   * - StagingConfig
     - 開発環境に依存する内容の設定クラス。RDS、DynamoDBで定義が必要なものでステージング環境に依存する部分を切り出して設定する。
     - ◯

   * - ProductionConfig
     - 開発環境に依存する内容の設定クラス。RDS、DynamoDBで定義が必要なもので商用環境に依存する部分を切り出して設定する。ステージング環境とほぼ同等となる(本来プロダクションの設定がステージングで動くようにするのがあるべき設定)のため、説明は省略する。
     - ◯

|br|

.. note:: アプリケーションでは実際にRDSやDynamoDBへアクセスする実装、SQSへポーリングするリスナー実装クラスなどもありますが、以前の連載で踏襲した実装にしているのでここでは詳しい解説は行いません。
          詳細な説明は `クラウドネイティブ第12回 <https://news.mynavi.jp/itsearch/article/devsoft/4426>`_ や `第13回 <https://news.mynavi.jp/itsearch/article/cloud/4449>`_ 、
          `第17回 <https://news.mynavi.jp/itsearch/article/cloud/4506>`_ 、`第18回 <https://news.mynavi.jp/itsearch/article/devsoft/4512>`_ 、
          `第32回 <https://news.mynavi.jp/itsearch/article/devsoft/4767>`_ を参照してください。

|br|

まず、DynamoDBの設定クラスであるDynamoDBConfigですが、以下の通り、環境に依存しない@EnableDynamoDBRepositoriesアノテーションの設定のみを行います。

|br|

.. sourcecode:: java

   package org.debugroom.mynavi.sample.cloudformation.backend.config;

   import org.socialsignin.spring.data.dynamodb.repository.config.EnableDynamoDBRepositories;

   import org.springframework.context.annotation.Configuration;

   @Configuration
   @EnableDynamoDBRepositories(
      basePackages = "org.debugroom.mynavi.sample.cloudformation.backend.domain.repository.dynamodb"
   )
   public class DynamoDBConfig {
   }

|br|

なお、以前の連載の実装では、クライアントとなるcom.amazonaws.services.dynamodbv2.AmazonDynamoDBのBean定義も合わせてこのクラス内で行っていました。
必要なDynamoDBのサービスエンドポイントやリージョンを環境変数の形で取得しており、環境ごとに設定内容に違いがなかったためです。
今回は環境変数を用いず、CloudFormationのスタック情報経由で取得するため、DevConfigクラス内に定義します。説明は後述します。

|br|

続いて、SQSConfigですが、SQSポーリングを行う場合では、同様にリスナークラスが存在するパッケージをコンポーネントスキャンする定義を追加するかたちで実装します。

|br|

.. sourcecode:: java

   package org.debugroom.mynavi.sample.cloudformation.backend.config;

   import org.springframework.context.annotation.ComponentScan;
   import org.springframework.context.annotation.Configuration;

   @Configuration
   @ComponentScan("org.debugroom.mynavi.sample.cloudformation.backend.app.listener")
   public class SQSConfig {
   }

|br|

こちらも同じく、以前の連載の実装では、SQSのクライアントとなるQueueのエンドポイントとリージョンの設定をこのクラス内で行っていましたが、
環境変数を用いず、CloudFormationのスタック情報経由で取得します。

|br|

そして、開発環境向けのの設定を行うDevConfigクラスを実装します。ここでは、CloudFormationで構築したスタックの情報を取得し、開発環境固有となる設定を行います。今回サンプルとして実装した例は以下の通りです。

|br|

.. sourcecode:: java

   package org.debugroom.mynavi.sample.cloudformation.backend.config;

   // omit

   import com.amazonaws.client.builder.AwsClientBuilder;
   import com.amazonaws.services.cloudformation.AmazonCloudFormationClient;
   import com.amazonaws.services.cloudformation.model.Export;
   import com.amazonaws.services.cloudformation.model.ListExportsRequest;
   import com.amazonaws.services.cloudformation.model.ListExportsResult;
   import com.amazonaws.services.dynamodbv2.AmazonDynamoDB;
   import com.amazonaws.services.dynamodbv2.AmazonDynamoDBClientBuilder;
   import com.amazonaws.services.dynamodbv2.datamodeling.DynamoDBMapperConfig;

   import org.springframework.beans.factory.InitializingBean;
   import org.springframework.beans.factory.annotation.Autowired;
   import org.springframework.cloud.aws.context.config.annotation.EnableStackConfiguration;
   import org.springframework.cloud.aws.core.env.ResourceIdResolver;
   import org.springframework.cloud.aws.jdbc.config.annotation.EnableRdsInstance;
   import org.springframework.context.annotation.Bean;
   import org.springframework.context.annotation.Configuration;
   import org.springframework.context.annotation.Profile;

   @Profile("dev")                                                                         // …(A)
   @EnableStackConfiguration(stackName = "mynavi-sample-infra-dev")                        // …(B)
   @EnableRdsInstance(
       dbInstanceIdentifier = "mynavi-sample-cloudformation-dev-postgresql",
       password = "${rds.password}",
       readReplicaSupport = false)                                                         // …(C)
   @Configuration
   public class DevConfig implements InitializingBean {                                    // …(D)

       @Autowired
       AmazonCloudFormationClient amazonCloudFormationClient;                              // …(E)

       @Autowired(required = false)
       ResourceIdResolver resourceIdResolver;                                              // …(F)

       @Bean
       public DynamoDBMapperConfig dynamoDBMapperConfig(){                                 // …(G)
          return DynamoDBMapperConfig.builder()
             .withTableNameOverride(
                     DynamoDBMapperConfig.TableNameOverride
                             .withTableNamePrefix("dev_"))
             .build();
      }

      @Bean
      public AmazonDynamoDB amazonDynamoDB(){
          ListExportsResult listExportsResult = amazonCloudFormationClient
             .listExports(new ListExportsRequest());                                       // …(H)
          List<Export> exportList = listExportsResult.getExports();
          List<Export> dynamoDBStackExportList = exportList.stream()
             .filter(export -> export.getExportingStackId().equals(
                     resourceIdResolver.resolveToPhysicalResourceId("DynamoDBDevStack")))  // …(I)
             .collect(Collectors.toList());
          Export dynamoDBServiceEndpointExport = dynamoDBStackExportList.stream()
             .filter(export -> export.getName().equals("MynaviSampleDynamoDB-Dev-ServiceEndpoint"))
             .findFirst().get();                                                           // …(J)
          Export environmentRegionExport = dynamoDBStackExportList.stream()
             .filter(export -> export.getName().equals("MynaviSampleDynamoDB-Dev-Region"))
             .findFirst().get();                                                           // …(K)
          return AmazonDynamoDBClientBuilder.standard()
             .withEndpointConfiguration(
                 new AwsClientBuilder.EndpointConfiguration(
                     dynamoDBServiceEndpointExport.getValue(), environmentRegionExport.getValue()))
             .build();                                                                     // …(L)
       }
       // omit
   }

|br|

DevConfig設定クラスの実装のポイントは(A)〜(L)の通りです。

|br|

.. list-table:: DevConfigクラス実装のポイント
   :widths: 1, 9

   * - 記述
     - 説明

   * - (A)
     - Springの起動時にプロファイル"dev"が指定されたときに有効化されるよう@Profileアノテーションを設定します。

   * - (B)
     - @EnableStackConfigurationでstackNameに前回NestedStackとして作成したスタック名を指定します。この設定により指定したスタックに定義されたリソースにおける、SpringCloudAWSがサポートする自動設定が可能になります。

   * - (C)
     - RDS構築のCloudFormationテンプレートで指定した開発環境用の設定値"dbInstanceIdentifier"の値を@EnableRdsInstanceに指定します。また、RDSのパスワードは秘匿性が高い情報のため、
       ステージング環境やプロダクション環境でアプリケーションが実行されるECSコンテナでSecretManagerやSystemsManager ParameterStoreから取得する想定とし、コンテナの環境変数として設定します。
       この設定方法は次回以降のECSコンテナのタスク定義を作成するCloudFormationテンプレートの解説で詳述しますが、開発環境でも同様に環境変数(application.ymlに記載したプロパティ)から取得する実装とします。

   * - (D)
     - 本来必要ではありませんが、このクラスではafterPropertiesSet()メソッドでスタック情報の取得方法をサンプル実装しているので、InitializingBeanインターフェースを継承しておきます。
       afterPropertiesSet()メソッドで実装する理由は、SpringによるResourceIdResolverのBean生成が完了してから情報を取得する必要があるためです。

   * - (E)
     - スタックの論理名やエクスポート名からリソース情報を取得するためにAmazonCloudFormationClientをインジェクションします。

   * - (F)
     - スタックの論理名から物理的にリソースのARNを取得するため、Spring Cloud AWSにより提供されているResourceIdResolverをインジェクションします。

   * - (G)
     - 開発環境用のDynamoDBにアクセスするためにテーブル名に「dev」というプリフィックスを使って読み込む設定を加えておきます。:ref:`section-cloudformation-dynamodb-sample-label` で実装したDynamoDBのCloudFormationテンプレートではこのプリフィックスの設定を見越した形でテーブル名を設定しています。

   * - (H)
     - (E)でインジェクションしたAmazonCloudFormationClientを使用して、スタックで定義されているOutputs要素のExport一覧を指定します。
       注意点として、AmazonCloudFomationClientで取得したスタック情報は、(B)で指定したスタック以外の、全てのCloudFomationスタック情報を取得します。

   * - (I)
     - 全てのエクスポートリストの中から、(B)で指定したスタックで定義されている開発環境DynamoDBの論理名を元に、子のDynamoDBテンプレートで定義されているスタック情報の一覧を取得します。
       論理名からリソースの物理ID(ARN)を取得し、エクスポートしているスタックのIDと一致しているものをピックアップして抽出しています。

   * - (J)
     - 抽出したDynamoDBスタックのエクスポートリストのうち、開発環境のDynamoDBサービスエンドポイントに関するエクスポートを取得します。

   * - (K)
     - 抽出したDynamoDBのエクスポートリストのうち、開発環境のDynamoDBサービスリージョンに関するエクスポートを取得します。

   * - (L)
     - エクスポートから出力した値を取得し、DynamoDBClientのエンドポイント情報を設定します。


.. note:: スタックのエクスポートで定義する名前は、CloudFormationで定義している全てのスタックで一意になるよう設定しなければならないため、上記の(I)は必ずしも必要ではありません。
   ただし、エクスポートリストは数が多くなりがちのため、ResourceIdResolverをうまく活用し、(B)で指定したリソースの物理IDから直接サービスクライアントで情報を取得する方法が検索効率が高い場合もあります。

|br|

続いて、ステージング(プロダクションも同等)環境固有の設定を行うStagingConfigの実装サンプルは以下の通りです。
可読性が向上するよう、CloudFormationから情報を取得する共通のユーティリティクラス(CloudFormationStackInfo)を実装し利用しています。

|br|

.. sourcecode:: java

   package org.debugroom.mynavi.sample.cloudformation.backend.config;

   import com.amazonaws.client.builder.AwsClientBuilder;
   import com.amazonaws.services.dynamodbv2.AmazonDynamoDB;
   import com.amazonaws.services.dynamodbv2.AmazonDynamoDBClientBuilder;
   import com.amazonaws.services.dynamodbv2.datamodeling.DynamoDBMapperConfig;

   import org.springframework.cloud.aws.context.config.annotation.EnableStackConfiguration;
   import org.springframework.cloud.aws.jdbc.config.annotation.EnableRdsInstance;
   import org.springframework.context.annotation.Bean;
   import org.springframework.context.annotation.Configuration;
   import org.springframework.context.annotation.Profile;

   import org.debugroom.mynavi.sample.cloudformation.common.apinfra.cloud.aws.CloudFormationStackInfo;

   @Profile("staging")                                                                         // …(A)
   @EnableStackConfiguration(stackName = "mynavi-sample-infra-staging")                        // …(B)
   @EnableRdsInstance(
        dbInstanceIdentifier = "mynavi-sample-cloudformation-staging-postgresql",              // …(C)
        password = "${rds.password}",
        readReplicaSupport = false)
   @Configuration
   public class StagingConfig {

       private static final String DYNAMODB_STACK_NAME = "DynamoDBStagingStack";               // …(D)
       private static final String DYNAMODB_ENDPOINT_EXPORT = "MynaviSampleDynamoDB-Staging-ServiceEndpoint";
       private static final String DYNAMODB_REGION_EXPORT = "MynaviSampleDynamoDB-Staging-Region";

       @Bean
       CloudFormationStackInfo cloudFormationStackInfo(){                                      // …(E)
           return new CloudFormationStackInfo();
       }

       @Bean
       public DynamoDBMapperConfig dynamoDBMapperConfig() {                                    // …(F)
           return DynamoDBMapperConfig.builder()
                .withTableNameOverride(
                        DynamoDBMapperConfig.TableNameOverride
                                .withTableNamePrefix("staging_"))
                .build();
       }

       @Bean
       public AmazonDynamoDB amazonDynamoDB() {                                                // …(G)
           return AmazonDynamoDBClientBuilder.standard()
                .withEndpointConfiguration(
                        new AwsClientBuilder.EndpointConfiguration(
                                cloudFormationStackInfo().getExportValue(
                                        DYNAMODB_STACK_NAME, DYNAMODB_ENDPOINT_EXPORT),
                                cloudFormationStackInfo().getExportValue(
                                        DYNAMODB_STACK_NAME, DYNAMODB_REGION_EXPORT)))
                .build();
       }

   }

|br|

StagingConfig設定クラスの実装のポイントは(A)〜(L)の通りです。

|br|

.. list-table:: StagingConfigクラス実装のポイント
   :widths: 1, 9

   * - 記述
     - 説明

   * - (A)
     - Springの起動時にプロファイル"staging"が指定されたときに有効化されるよう@Profileアノテーションを設定します。

   * - (B)
     - @EnableStackConfigurationでスタック名にNestedStackとして作成するスタック名を指定します。この設定により指定したスタックに定義されたリソースにおける、SpringCloudAWSがサポートする自動設定が可能になります。

   * - (C)
     - RDS構築のCloudFormationテンプレートで指定したステージング環境用の設定値"dbInstanceIdentifier"の値を@EnableRdsInstanceに指定します。また、RDSのパスワードは秘匿性が高い情報のため、
       ステージング環境やプロダクション環境でアプリケーションが実行されるECSコンテナでSecretManagerやSystemsManager ParameterStoreから取得する想定とし、コンテナの環境変数として設定します。
       この設定方法は次回以降のECSコンテナのタスク定義を作成するCloudFormationテンプレートの解説で詳述しますが、開発環境でも同様に環境変数(application.ymlに記載したプロパティ)から取得する実装とします。

   * - (D)
     - CloudFomationテンプレートのOutputs要素で定義したエクスポート名を定数値として定義します。

   * - (E)
     - AmazonCloudFomationClientによるスタック情報の取得処理を共通化したユーティリティクラスをBean生成します。実装内容は後述します。

   * - (F)
     - ステージング環境用のDynamoDBにアクセスするためにテーブル名に「staging」というプリフィックスを使って読み込む設定を加えておきます。:ref:`section-cloudformation-dynamodb-sample-label` で実装したDynamoDBのCloudFormationテンプレートではこのプリフィックスの設定を見越した形でテーブル名を設定しています。

   * - (G)
     - エクスポートから出力した値を取得し、DynamoDBClientのエンドポイント情報を設定します。

|br|

AmazonCloudFormationClientでスタック情報を取得する処理を共通化したユーティリティクラスの実装ですが、基本的にはスタックの論理名およびエクスポート名から出力値を取得するものになります。サンプルとして作成したものは以下の通りです。

|br|

.. sourcecode:: java

   package org.debugroom.mynavi.sample.cloudformation.common.apinfra.cloud.aws;

   // omit
   import com.amazonaws.services.cloudformation.AmazonCloudFormationClient;
   import com.amazonaws.services.cloudformation.model.Export;
   import com.amazonaws.services.cloudformation.model.ListExportsRequest;
   import com.amazonaws.services.cloudformation.model.ListExportsResult;

   import org.springframework.beans.factory.annotation.Autowired;
   import org.springframework.beans.factory.InitializingBean;
   import org.springframework.cloud.aws.core.env.ResourceIdResolver;
   import org.springframework.stereotype.Component;

   @Component
   public class CloudFormationStackInfo implements InitializingBean {

       @Autowired
       AmazonCloudFormationClient amazonCloudFormationClient;

       @Autowired(required = false)
       ResourceIdResolver resourceIdResolver;

       private List<Export> exportList;

       public String getExportValue(String logicalStackName, String exportName){
           Optional<Export> targetExport = exportList.stream()
             .filter(export -> export.getExportingStackId().equals(
                     resourceIdResolver.resolveToPhysicalResourceId(logicalStackName)))
             .filter(export -> export.getName().equals(exportName))
             .findFirst();
           return targetExport.isPresent() ? targetExport.get().getValue() : null;
       }

       @Override
       public void afterPropertiesSet() throws Exception {
           ListExportsResult listExportsResult = amazonCloudFormationClient
             .listExports(new ListExportsRequest());
           this.exportList = listExportsResult.getExports();
       }

   }

|br|

.. note:: Backend ServiceアプリケーションではSQSキューのポーリング取得を実装していますが、SQSに関する設定は前述の通り、リスナークラスのコンポーネントスキャンしか実施していません。
   というのもSpring Cloud AWSでSQSキューを取得するための指定は以下の通り@SqsListenerアノテーションで指定したValue属性を使う方法がありますが、Configクラスの@EnableStackConfigurationでスタックを指定することにより、
   スタックの論理名からキューの物理ID(ARN)を取得する実装になっているためです。開発環境、ステージング環境、商用環境のスタック名を切り替えることで特に設定値を気にせず共通のアプリケーションコードでキューを取得できます。

   .. sourcecode:: java

      package org.debugroom.mynavi.sample.cloudformation.backend.app.listener;

      // omit
      import org.springframework.cloud.aws.messaging.config.annotation.EnableSqs;
      import org.springframework.cloud.aws.messaging.listener.SqsMessageDeletionPolicy;
      import org.springframework.cloud.aws.messaging.listener.annotation.SqsListener;
      import org.springframework.stereotype.Component;

      import lombok.extern.slf4j.Slf4j;

      @Component
      @EnableSqs
      @Slf4j
      public class MessageListener {

          @Autowired
          SampleService sampleService;

          @SqsListener(value = "SQSSampleQueue", deletionPolicy = SqsMessageDeletionPolicy.ON_SUCCESS)
          public void onMessage(String message){
              log.info(message);
              sampleService.addSample(message);
          }

      }

|br|

今回はBackend ServiceアプリケーションでCloudFormationのスタック情報を用いて、RDSやDynamoDB、SQSキュー取得の設定情報を取得する実装方法を紹介しました。
次回はElastiCacheや、S3、SQSキュー送信の設定情報を取得するFrontendアプリケーションの解説を進めていきます。

|br|

著者紹介
------------------------------------------------------------------

川畑 光平(KAWABATA Kohei) - NTTデータ 課長代理

.. figure:: img/automation_infra_devops_overview/pic_image01.jpg


金融機関システム業務アプリケーション開発・システム基盤担当を経て、現在はソフトウェア開発自動化関連の研究開発・推進に従事。

Red Hat Certified Engineer、Pivotal Certified Spring Professional、AWS Certified Solutions Architect Professional等の資格を持ち、アプリケーション基盤・クラウドなど様々な開発プロジェクト支援にも携わる。

`2019 APN AWS Top Engineers & Ambassadors <https://aws.amazon.com/jp/blogs/psa/japan-apn-ambassador-2019/>`_ 選出。
