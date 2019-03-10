.. include:: ../module.txt

.. _section-cloud-native-nosql-label-2-2:

AWSで作るクラウドネイティブアプリケーションの基本
========================================================================================

.. _section-cloud-native-nosql-2nd-2-label:

第3回 AWS上に構築するNoSQLアプリケーション(2)-2
----------------------------------------------------------------------------------------

|br|

.. _section-cloud-native-nosql-spring-applicaiton-2-2-label:

Amazon DynamoDBへアクセスするSpringアプリケーション
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

クラウドの普及に伴い、ビッグデータやキーバリュー型データの格納など、ますます活用の機会が広がりつつあるNoSQLデータベース。
第3回は代表的なNoSQLプロダクトであるAmazon DynamoDBやApache Cassandra、Amazon ElastiCacheへアクセスするSpringアプリケーションを開発する方法について、わかりやすく解説します。

本連載では、以下の様なステップで進めています。

|br|

#. NoSQLデータベースの特徴とデータ特性

   * CAP定理を元にしたデータベースの分類とデータ特性
   * AP型データベースAmazon DynamoDBとApache Cassandraの特徴

#. Amazon DynamoDBへアクセスするSpringアプリケーション

   * Amazon DynamoDBの概要及び構築と認証情報の作成
   * **Spring Data DynamoDBを用いたアプリケーション(1)**
   * Spring Data DynamoDBを用いたアプリケーション(2)

#. Apache CassandraへアクセスするSpringアプリケーション

   * Apache Cassandraの概要及びローカル環境構築
   * Spring Data Cassandraを用いたアプリケーション(1)
   * Spring Data Cassandraを用いたアプリケーション(2)

#. Amazon ElastiCacheへアクセスするSpringアプリケーション

   * AmazonElasiCacheの概要及びローカル環境でのRedisServer構築
   * Spring SessionとSpring Data Redisを用いたアプリケーション(1)
   * Spring SessionとSpring Data Redisを用いたアプリケーション(2)
   * Amazon ElastiCacheの設定
   * セッション共有するECSアプリケーションの構築(1)
   * セッション共有するECSアプリケーションの構築(2)

|br|

前回 :ref:`section-cloud-native-dynamodb-overview-label` では、AP型データベースである、Amazon DynamoDBの概要を説明し、テーブル構築しました。また、アプリケーションでアクセスするためのユーザを作成しています。
今回はSpring Data DynamoDBを使ってデータベースアクセスするアプリケーションを実装してみます。

|br|

.. _section-cloud-native-spring-data-dynamodb-overview-label:

Spring Data DynamoDBの概要
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

|br|

`Spring Data DynamoDB <https://github.com/derjust/spring-data-dynamodb>`_ は Spring DataのコミュニティプロジェクトでSpring Data JPAなどと同様、
データベースアクセス処理を抽象化・簡易化し、CRUD操作などの処理実装を提供するフレームワークです。
主な特徴は以下の通りです。

* DynamoDBテーブルに対するCRUDメソッドの実装の提供
* キーワードを元にしたクエリメソッドによる動的クエリの生成
* アノテーションベースの簡易なスキーマ表現
* ページネーション、カスタムデータアクセス・データ射影
* Spring Data RestとのRESTサポート

今回はこのフレームワークを用いて、DynamoDBにアクセスするアプリケーションを実装してみましょう。
なお、実際に作成したアプリケーションは `GitHub <https://github.com/debugroom/mynavi-sample-spring-data-dynamodb>`_ 上にコミットしています。
以降記載されているソースコードで、インポート文など本質的でない記述を省略している部分がありますので、実行コードを作成する場合は、
必要に応じて適宜GitHubソースコードも参照してください。

|br|

.. _section-cloud-native-spring-data-dynamodb-implementation-1-label:

Spring Data DynamoDBを使ったアプリケーション実装(1)
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

|br|

Spring Data DynamoDBを使用するには、まず、Mavenプロジェクトのpom.xmlで、spring-data-dynamodbのライブラリを定義します。
なお、DynamoDBへのアクセスを画面から実行するアプリケーションを作成するために、Webアプリケーションを作るための
spring-boot-starter-web、thymeleafテンプレートエンジンを使用するためのspring-boot-starter-thymeleaf、
DynamoDBテーブルのエンティティクラスの実装を簡易化するためにLombokライブラリを追加しておきましょう。

|br|

.. sourcecode:: xml

   <dependencies>
     <dependency>
       <groupId>com.github.derjust</groupId>
       <artifactId>spring-data-dynamodb</artifactId>
     </dependency>
     <dependency>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter-thymeleaf</artifactId>
     </dependency>
     <dependency>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter-web</artifactId>
     </dependency>
     <dependency>
       <groupId>org.projectlombok</groupId>
       <artifactId>lombok</artifactId>
       <optional>true</optional>
     </dependency>
   </dependencies>

|br|

今回、アプリケーションは以下のコンポーネントで構成しますが、Spring Data DynamoDBで最低限作成しなければならない必須のクラスは以下の通りです。
なお、thymeleafテンプレートエンジンに関する説明は省略しますが、index.htmlからfindOne、findAll、add、update、deleteなどのCRUD処理を呼び出し、
Controllerを通じて、結果に遷移する画面HTMLを各々実装しています。

|br|

.. list-table:: アプリケーション
   :widths: 3, 6, 1

   * - コンポーネント
     - 説明
     - 必須

   * - WebApp
     - SpringBootアプリケーションを実行する起動クラス
     - ◯

   * - MvcConfig
     - Webアプリケーションの設定を行うクラス
     -

   * - DomainConfig
     - サービスレイヤの設定を行うクラス
     -

   * - DynamoDBConfig
     - DynamoDBへの接続に関する設定クラス
     - ◯

   * - SampleModel
     - 画面からのインプットパラメータを表すモデル
     -
   * - SampleController
     - 画面からのリクエストを受け付け、Serviceを実行し、結果をテンプレートエンジンに渡す
     -

   * - SampleModelMapper
     - SampleModelとMynaviSampleTableのパラメータをマッピングする。
     -

   * - SampleService
     - トランザクション境界となり、Repositoryを実行する。
     -

   * - SampleRepository
     - データベースアクセスを実行する。
     - ◯

   * - MynaviSampleTable
     - DynamoDBのテーブルを表現するエンティティクラス
     - ◯

|br|

以降、各々のクラスについて解説を進めていきますが、事前に `AWS公式ページ「設定ファイルと認証情報ファイル」 <https://docs.aws.amazon.com/ja_jp/cli/latest/userguide/cli-configure-files.html>`_ を参考に
ユーザホームフォルダに.awsディレクトリを作成し、credentialというファイル名で、前回取得したCSV形式の認証キーに記載しているユーザ認証情報を、以下の形式で保存してください。

|br|

.. sourcecode:: bash

   [default]
   aws_access_key_id=XXXXXXXXXXXXXXXX
   aws_secret_access_key=YYYYYYYYYYYYYYYYYYYYYYYYYYYYY

|br|

それでは、実装していくクラスを説明します。まず、最初にSpringBoot起動クラス及び、各種設定クラスです。
@SpringBootApplicaitonアノテーションが付与された起動クラスは、同一パッケージにある@Configurationアノテーションが付与された設定クラス及び、
設定クラス内で@ComponentScanされたパッケージにあるクラスを読み取ります。今回は目的に応じて以下の3つに分類して設定クラスを作成します。

* SpringMVCの設定クラスであるWebMvcConfigurerを実装した設定クラス：MvcConfigクラス
* Serviceをコンポーネントスキャンで読み取る設定クラス：DomainConfigクラス
* DynamoDBの接続を行う設定クラス：DynamoDBConfigクラス

設定クラスは必ずしも複数である必要はなく一つにまとめても動作上問題ありませんが、クラス名と役割を対応づけて作成していた方が、
後々設定内容を混乱することなく、クラス名から識別できてベターです。

|br|

.. sourcecode:: java

   package org.debugroom.mynavi.sample.spring.data.dynamodb.config;

   import org.springframework.boot.SpringApplication;
   import org.springframework.boot.autoconfigure.SpringBootApplication;

   @SpringBootApplication
   public class WebApp {

       public static void main(String[] args) {
           SpringApplication.run(WebApp.class, args);
       }

   }

|br|

同一パッケージに配置するMvcConfigクラスでは、HTMLやCSSなどの静的リソースのURLと実際のリソースの物理配置の対応づけの定義をResourceHandlerRegistryに追加しておきます。
また、Controllerクラスを読み取るために、ComponentScanアノテーションに該当のパッケージを指定しておきます。

|br|

.. sourcecode:: java

   package org.debugroom.mynavi.sample.spring.data.dynamodb.config;

   import org.springframework.context.annotation.ComponentScan;
   import org.springframework.context.annotation.Configuration;
   import org.springframework.web.servlet.config.annotation.ResourceHandlerRegistry;
   import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

   @ComponentScan("org.debugroom.mynavi.sample.spring.data.dynamodb.app.web")
   @Configuration
   public class MvcConfig implements WebMvcConfigurer {

       @Override
       public void addResourceHandlers(ResourceHandlerRegistry registry) {
           registry.addResourceHandler("/static/**")
             .addResourceLocations("classpath:/static/");
       }

   }

|br|

DomainConfigクラスではServiceなどビジネスロジックレイヤに属するコンポーネントがあるパッケージをスキャンする定義を追加しておきます。

|br|

.. sourcecode:: java

   package org.debugroom.mynavi.sample.spring.data.dynamodb.config;

   import org.springframework.context.annotation.ComponentScan;
   import org.springframework.context.annotation.Configuration;

   @ComponentScan("org.debugroom.mynavi.sample.spring.data.dynamodb.domain")
   @Configuration
   public class DomainConfig {
   }

|br|

続いて、DynamoDBへの接続で必須になる設定クラスがDynamoDBConfigクラスです。@EnableDynamoDBRepositoriesアノテーションを付与すると、
DynamoDBの接続に必要なクラスがアプリケーション起動時に自動で構築されていく様になります。@EnableDynamoDBRepositoriesには
同時にDynamoDBテーブルへアクセスするRepositoryクラスがあるベースパッケージを同時に指定しなければなりません。

DynamoDB接続に必要なクラスはSpringBootが自動構築してくれますが、最低限、接続先のDynamoDBのリージョンやエンドポイント、認証情報の３つを設定する必要があります。
Spring Data DynamoDBも内部的には、DynamoDBとの接続にAWS SDKが提供しているcom.amazonaws.services.dynamodbv2.AmazonDynamoDBを利用しており、
com.amazonaws.services.dynamodbv2.AmazonDynamoDBClientBuilderを使って、リージョンや認証情報を含めてインスタンスを作成するのが一般的です。
そのため、設定クラス上でAmazonDynamoDBクラスの設定を行ってBean定義します。

なお、AmazonDynamoDBClientBuilderは、デフォルトで、事前に設定しておいた.aws内にあるCredentialの中にある
アクセスIDとシークレットキーを参照しにいく実装になっています。たまに古いWeb記事のサンプルでアプリケーション内で直接
ClientBuilderにアクセスIDとシークレットキーを設定するコードを見かけますが、AWSとしてこのやり方は推奨していないので、
アプリケーションにキーを設定するコードを記載するのはやめておきましょう。以下のコードでは、
AwsClientBuilderのEndpointConfigrationに、東京リージョンとDynamoDBのサービスエンドポイントを設定ファイルapplication.ymlから取得して、
AmazonDynamoDBClientBuilderへ設定しています。

|br|

.. sourcecode:: java

   package org.debugroom.mynavi.sample.spring.data.dynamodb.config;

   import com.amazonaws.client.builder.AwsClientBuilder;
   import com.amazonaws.services.dynamodbv2.AmazonDynamoDB;
   import com.amazonaws.services.dynamodbv2.AmazonDynamoDBClientBuilder;
   import org.socialsignin.spring.data.dynamodb.repository.config.EnableDynamoDBRepositories;
   import org.springframework.beans.factory.annotation.Value;
   import org.springframework.context.annotation.Bean;
   import org.springframework.context.annotation.Configuration;

   @Configuration
   @EnableDynamoDBRepositories(basePackages = "org.debugroom.mynavi.sample.spring.data.dynamodb.domain.repository")
   public class DynamoDBConfig {

       @Value("${amazon.dynamodb.region}")
       private String region;
       @Value("${amazon.dynamodb.endpoint}")
       private String endpoint;

       @Bean
       public AmazonDynamoDB amazonDynamoDB(){
           return AmazonDynamoDBClientBuilder.standard().withEndpointConfiguration(
                new AwsClientBuilder.EndpointConfiguration(endpoint, region)).build();
       }

   }

|br|

application.yml

.. sourcecode:: bash

   amazon:
     dynamodb:
       region: ap-northeast-1
       endpoint: https://dynamodb.ap-northeast-1.amazonaws.com

|br|

.. note:: DynamoDBオブジェクトの生成に利用したAWS SDKのAmazonDynamoDBClientBuilderですが、正確に言えば以下の順序で認証情報を取得していく実装になっています。

   #. 環境変数AWS_ACCESS_KEY_IDとAWS_SECRET_ACCESS_KEY
   #. システムプロパティaws.accessKeyIdとaws.secretKey
   #. ユーザのAWS認証情報ファイル
   #. AWSインスタンスプロファイルの認証情報

   これまでの設定では、ローカル環境ではユーザのAWS認証情報ファイルで取得する状態になっていますが、この実装のまま本番環境へ持って行っても、最後のAWSインスタンスプロファイルから認証情報を取得できます。
   この辺の挙動の詳細を知りたい方は `DefaultAWSCredentialsProviderChainの実装 <https://github.com/aws/aws-sdk-java/blob/master/aws-java-sdk-core/src/main/java/com/amazonaws/auth/DefaultAWSCredentialsProviderChain.java>`_ を参照してみてください。
   最後のインスタンスプロファイル情報はアプリケーションを実行するEC2インスタンスもしくはECSコンテナがもつ認証情報で、
   `インスタンスやコンテナを実行する際、IAMロールから権限を割り当てるオプション <https://docs.aws.amazon.com/ja_jp/AWSEC2/latest/UserGuide/iam-roles-for-amazon-ec2.html>`_ があります。
   開発環境・本番環境が変わるからといって、環境依存をなくすためにアプリケーション内でアクセスキーIDやシークレットキーを割り当てる実装は必要ありませんので、本稿の様に、
   開発環境(AWS内ではないローカル環境の場合)は.aws配下の認証情報を取得し、本番環境では、インスタンスのプロファイル上から認証情報を取得する実装を心がけてください。

|br|

アプリケーションの設定に関する実装はここまでです。引き続き、次回、アプリケーションのDBアクセス処理等の実装を進めていきます。

|br|

著者紹介
------------------------------------------------------------------

川畑 光平(KAWABATA Kohei)

.. figure:: img/aws-lambda-and-api-gateway/pic_image01.jpg
   :scale: 100%

某システムインテグレータにて、金融機関システム業務アプリケーション開発・システム基盤担当を経て、現在はソフトウェア開発自動化関連の研究開発・推進に従事。

Red Hat Certified Engineer、Pivotal Certified Spring Professional、AWS Certified Solutions Architect Professional等の資格を持ち、アプリケーション基盤・クラウドなど様々な開発プロジェクト支援にも携わる。

本連載記事の内容に対するご意見・ご質問は `Facebook <https://www.facebook.com/kohei.kawabata.5>`_ まで。
