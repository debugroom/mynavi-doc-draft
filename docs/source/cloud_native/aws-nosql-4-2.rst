.. include:: ../module.txt

.. _section-cloud-native-nosql-label-4-2:

AWSで作るクラウドネイティブアプリケーションの基本
========================================================================================

.. _section-cloud-native-nosql-4th-2-label:

第3回 AWS上に構築するNoSQLアプリケーション(4)-2
----------------------------------------------------------------------------------------

|br|

.. _section-cloud-native-nosql-spring-applicaiton-4-2-label:

Amazon ElastiCacheへアクセスするSpringアプリケーション
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

   * Amazon DynamoDBの概要及び構築と認証情報の設定
   * Spring Data DynamoDBを用いたアプリケーション(1)
   * Spring Data DynamoDBを用いたアプリケーション(2)

#. Apache CassandraへアクセスするSpringアプリケーション

   * Apache Cassandraの概要及びローカル環境構築
   * Spring Data Cassandraを用いたアプリケーション(1)
   * Spring Data Cassandraを用いたアプリケーション(2)

#. Amazon ElastiCacheへアクセスするSpringアプリケーション

   * AmazonElasiCacheの概要及びローカル環境でのRedisServer構築
   * **Spring SessionとSpring Cloud Data Redisを用いたアプリケーション(1)**
   * Spring SessionとSpring Cloud Data Redisを用いたアプリケーション(2)
   * Amazon ElastiCacheの設定
   * セッション共有するECSアプリケーションの構築(1)
   * セッション共有するECSアプリケーションの構築(2)

|br|

前回 :ref:`section-cloud-native-nosql-spring-applicaiton-4-1-label` では、CP型データベースである、Amazon ElastiCacheの概要を説明し、
シングルノードでローカル環境へテーブル構築しました。今回は、Spring SessionおよびSpring Data Redisを使ってアクセスするアプリケーションを実装してみます。

|br|

.. _section-cloud-native-spring-session-and-spring-data-redis-label:

Spring Data SessionとSpring Data Redisの概要
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

|br|

`Spring Session <https://spring.io/projects/spring-session>`_ はSpringのトップレベルプロジェクトの１つで、ユーザのセッション情報に関するAPIとその実装を提供します。
以下のセッション情報を透過的に(従来のセッション情報と変わらないやり方で)扱うことが可能です。

* HttpSession：従来アプリケーションサーバで保持していたセッション情報を共有キャッシュへ置き換える実装を提供します。
* WebSockets：従来のスティッキーセッションセッションが必要なくなり、WebSocketsメッセージを取り扱うことが可能になります。
* WebSession：Spring5からサポートされたノンブロッキングなWebSessionを取り扱うことが可能になります。

|br|

また、 `Spring Data Redis <https://spring.io/projects/spring-data-redis>`_ は Spring Data JPAやこれまで紹介してきたSpring Data Cassandara、DynamoDBと同じく、
Spring Dataのメインモジュールで、統一的なインターフェースでデータアクセスが行えます。Spring Data Redisの特徴としては、以下の通りです。

* Jedisとlettuceなど、複数のRedisドライバを通した抽象的なアクセス手段の提供
* CrudRepositoryの継承インターフェースによる基本的なデータアクセスコード実装量の軽減
* @EnableRedisRepsitoriesアノテーションを使ったRepositoryインターフェースの拡張実装の提供
* Redis driverがスローする例外の、共通的なDataAccessExceptionなどへの透過的な変換
* Spring Frameworkとの統合、依存性の注入によるRedisTemplateクラスの提供。
* Pub/Subモデルのサポート

今回はこれらのフレームワークを用いて、セッション情報をRedisで共有するアプリケーションを実装してみましょう。
前回構築したローカル環境のRedisへのアクセス及び、次回以降構築するAWS ECSコンテナでアプリケーションをデプロイし、ElastiCacheへアクセスができるようにアプリケーションを実装していきます。

なお、実際に作成したアプリケーションは `GitHub <https://github.com/debugroom/mynavi-sample-aws-elasticache>`_ 上にコミットしています。
以降記載されているソースコードで、インポート文など本質的でない記述を省略している部分がありますので、実行コードを作成する場合は、
必要に応じて適宜GitHubソースコードも参照してください。

|br|

.. _section-cloud-native-spring-session-data-redis-implementation-1-label:

Spring SessionとSpring Data Redisを使ったアプリケーション実装(1)
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

|br|

Spring SessionとSpring Data Redisを使用するには、Mavenプロジェクトのpom.xmlで、spring-session-data-redisのライブラリを定義します。
加えて、lettuceのDriverを定義してください。なお、共有セッションへのアクセスを画面から確認するアプリケーションを作成するために、
spring-boot-starter-web、thymeleafテンプレートエンジンを使用するためのspring-boot-starter-thymeleafを追加しておきましょう。

|br|

.. sourcecode:: xml

   <dependencies>
     <dependency>
       <groupId>org.springframework.session</groupId>
       <artifactId>spring-session-data-redis</artifactId>
     </dependency>
     <dependency>
       <groupId>io.lettuce</groupId>
       <artifactId>lettuce-core</artifactId>
     </dependency>
     <dependency>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter-thymeleaf</artifactId>
     </dependency>
     <dependency>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter-web</artifactId>
     </dependency>
  </dependencies>

|br|

今回、アプリケーションは以下のコンポーネントで構成しますが、Spring Session とSpring Data Redisで最低限作成しなければならない必須のクラスは以下の通りです。
なお、thymeleafテンプレートエンジンに関する説明は省略しますが、index.htmlからセッション情報を更新し、結果に遷移する画面HTMLを各々実装しています。

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

   * - RedisConfig
     - Redisへの接続に関する設定クラス
     -

   * - ElastiCacheConfig
     - ElastiCacheに接続する際に設定するクラス
     - ◯

   * - SampleSession
     - セッション情報に格納するモデルクラス
     - ◯

   * - SampleController
     - 画面からのリクエストを受け付け、セッション情報の更新を行い、結果をテンプレートエンジンに渡す
     - ◯


|br|

それでは、実装していくクラスを説明します。まず、最初にSpringBoot起動クラス及び、各種設定クラスです。@SpringBootApplicaitonアノテーションが付与された起動クラスは、同一パッケージにある@Configurationアノテーションが付与された設定クラス及び、
設定クラス内で@ComponentScanされたパッケージにあるクラスを読み取ります。今回は目的に応じて以下の4つに分類して定義します。

* アプリケーションを起動するためのWebAppクラス
* SpringMVCの設定クラスであるWebMvcConfigurerを実装した設定クラス：MvcConfigクラス
* Redisへアクセスを行う設定クラス：RedisConfigクラス
* ElasticCache接続に必要な設定を行う設定クラス：ElastiCacheConfigクラス

以下にアプリケーション起動クラスを示します。@SpringBootApplicaitonアノテーションを設定し、SpringAppcalition.run()を実行するmain()メソッドを実装します。
設定クラスは必ずしも複数である必要はなく一つにまとめても動作上問題ありませんが、クラス名と役割を対応づけて作成していた方が、後々設定内容を混乱することなく、クラス名から識別できてベターです。

|br|

.. sourcecode:: java

   package org.debugroom.mynavi.sample.aws.elasticache.config;

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
また、Controllerクラスを読み取るために、ComponentScanアノテーションにパッケージを指定しておきます。

|br|

.. sourcecode:: java

   package org.debugroom.mynavi.sample.aws.elasticache.config;

   import org.springframework.context.annotation.ComponentScan;
   import org.springframework.context.annotation.Configuration;
   import org.springframework.web.servlet.config.annotation.ResourceHandlerRegistry;
   import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

   @Configuration
   @ComponentScan("org.debugroom.mynavi.sample.aws.elasticache.app.web")
   public class MvcConfig implements WebMvcConfigurer {

       @Override
       public void addResourceHandlers(ResourceHandlerRegistry registry) {
           registry.addResourceHandler("/static/**")
                .addResourceLocations("classpath:/static/");
       }

   }

|br|

続いて、Redisのアクセス設定をRedisConfigクラス及びapplication.ymlで行います。基本的には、Spring SessionでRedisを使用する場合、Mavenのpom.xmlでライブラリを追加した後、
設定クラスに@EnableRedisHttpSessionアノテーションを付与するか、application.ymlでspring.session.store-typeプロパティをredisに設定にすることで、
同等の効果を得ることができます。

.. sourcecode:: bash

   spring:
     session:
       store-type: redis

|br|

application.ymlの設定があれば、Configクラスの設定は必ずしも必要ではありませんが、デフォルトではセッション情報保存時にJavaオブジェクトをシリアライズして保存するため、
Redisクライアントから中身を確認するためにJSON形式で保存するようにデフォルトのシリアライザを変更します。

.. sourcecode:: java

   package org.debugroom.mynavi.sample.aws.elasticache.config;

   import org.springframework.context.annotation.Bean;
   import org.springframework.context.annotation.Configuration;
   import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;
   import org.springframework.data.redis.serializer.RedisSerializer;

   @Configuration
   public class RedisConfig {

       @Bean
       public RedisSerializer<Object> springSessionDefaultRedisSerializer() {
           return new GenericJackson2JsonRedisSerializer();
       }

   }

|br|

また、ローカル環境のRedisでは問題ありませんが、ElastiCacheではconfigコマンドの実行が禁止されているため、ConfigureRedisActionをNO_OPに設定しておく必要があります。
ElastiCache接続時に有効化すれば良いため、実行時にプロファイル「production」を指定するものとして、以下のようなConfigクラスを追加しておきましょう。

.. sourcecode:: java

   package org.debugroom.mynavi.sample.aws.elasticache.config;

   import org.springframework.context.annotation.Bean;
   import org.springframework.context.annotation.Configuration;
   import org.springframework.context.annotation.Profile;
   import org.springframework.session.data.redis.config.ConfigureRedisAction;

   @Profile("production")
   @Configuration
   public class ElastiCacheConfig {

       @Bean
       public ConfigureRedisAction configureRedisAction() {
           return ConfigureRedisAction.NO_OP;
       }

   }

|br|

加えて、Redisに接続するためのIPアドレス及びポートをspring.redis.host及びspring.redis.portプロパティに指定しておく必要があります。

applicaiton.yml

.. sourcecode:: bash

   spring:
     redis:
       host: localhost
       port: 6379

|br|

また、ElastiCacheに接続するAWS環境では、ECSタスクに設定する環境変数からElastiCacheのエンドポイントを取得するよう、application-production.ymlを作成しておきます。

|br|

applicaiton-production.yml

.. sourcecode:: bash

   spring:
     redis:
       host: ${REDIS_CLUSTER_ENDPOINT:localhost}
       port: ${REDIS_CLUSTER_PORT:6379}

|br|

.. note:: application-production.ymlはアプリケーション起動時にプロファイル「production」を指定することで有効化(applicaiton.ymlは上書き)される設定ファイルです。
   プロファイル名に合わせた設定ファイルを作成しておくことで設定を柔軟に切り替えることができます。

|br|

アプリケーションの設定に関する実装はここまでです。次回以降、引き続きセッション共有処理に関わる実装を進めていきます。

|br|

著者紹介
------------------------------------------------------------------

川畑 光平(KAWABATA Kohei)

.. figure:: img/aws-lambda-and-api-gateway/pic_image01.jpg
   :scale: 100%

某システムインテグレータにて、金融機関システム業務アプリケーション開発・システム基盤担当を経て、現在はソフトウェア開発自動化関連の研究開発・推進に従事。

Red Hat Certified Engineer、Pivotal Certified Spring Professional、AWS Certified Solutions Architect Professional等の資格を持ち、アプリケーション基盤・クラウドなど様々な開発プロジェクト支援にも携わる。

`2019 APN AWS Top Engineers & Ambassadors <https://aws.amazon.com/jp/blogs/psa/japan-apn-ambassador-2019/>`_ 選出。

本連載記事の内容に対するご意見・ご質問は `Facebook <https://www.facebook.com/kohei.kawabata.5>`_ まで。
