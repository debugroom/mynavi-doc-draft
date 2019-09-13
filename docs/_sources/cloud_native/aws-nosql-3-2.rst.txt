.. include:: ../module.txt

.. _section-cloud-native-nosql-label-3-2:

AWSで作るクラウドネイティブアプリケーションの基本
========================================================================================

.. _section-cloud-native-nosql-3rd-2-label:

第3回 AWS上に構築するNoSQLアプリケーション(3)-2
----------------------------------------------------------------------------------------

|br|

.. _section-cloud-native-nosql-spring-applicaiton-3-2-label:

Apache CassandraへアクセスするSpringアプリケーション
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
   * Spring Data DynamoDBを用いたアプリケーション(1)
   * Spring Data DynamoDBを用いたアプリケーション(2)

#. Apache CassandraへアクセスするSpringアプリケーション

   * Apache Cassandraの概要及びローカル環境構築
   * **Spring Data Cassandraを用いたアプリケーション(1)**
   * Spring Data Cassandraを用いたアプリケーション(2)

#. Amazon ElastiCacheへアクセスするSpringアプリケーション

   * AmazonElasiCacheの概要及びローカル環境でのRedisServer構築
   * Spring SessionとSpring Data Redisを用いたアプリケーション(1)
   * Spring SessionとSpring Data Redisを用いたアプリケーション(2)
   * Amazon ElastiCacheの設定
   * セッション共有するECSアプリケーションの構築(1)
   * セッション共有するECSアプリケーションの構築(2)

|br|

前回 :ref:`section-cloud-native-nosql-spring-applicaiton-3-1-label` では、AP型データベースである、Apache Cassandraの概要を説明し、
シングルノードでローカル環境へテーブル構築しました。今回はSpring Data DynamoDBを使ってデータベースアクセスするアプリケーションを実装してみます。

|br|

.. _section-cloud-native-spring-data-dynamodb-cassandra-label:

Spring Data Cassandraの概要
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

|br|

`Spring Data Cassandra <https://spring.io/projects/spring-data-cassandra>`_ は Spring Data JPAなどと同様、データベースアクセス処理を抽象化・簡易化し、CRUD操作などの処理実装を提供するフレームワークです。
Spring Data Cassandraが提供するメリットの内容としては、以下の通りです。

* Cassandraへのデータアクセスするためのボイラープレートコードの隠蔽。
* Spring Frameworkとの統合、依存性の注入によるCassandaTemplateクラスの提供。
* エンティティクラスアノテーションによるオブジェクトマッピング機能
* 設定クラスによるAP起動時のCassandraテーブル構築オプションの提供
* CrudRepositoryの継承インターフェースによる基本的なデータアクセスコード実装量の軽減
* メソッド命名規約によるCQL自動組み立てによるコード実装量の軽減
* Spring Data のRepositoryCustomやResultsetExecutorによる拡張性の保持
* SrpngのDataアクセス例外を提供することによる、サービスロジックの抽象化による差分の隠蔽。

今回はこのフレームワークを用いて、Apache Cassandraにアクセスするアプリケーションを実装してみましょう。
なお、実際に作成したアプリケーションは `GitHub <https://github.com/debugroom/mynavi-sample-spring-data-cassandra>`_ 上にコミットしています。
以降記載されているソースコードで、インポート文など本質的でない記述を省略している部分がありますので、実行コードを作成する場合は、
必要に応じて適宜GitHubソースコードも参照してください。

|br|

.. _section-cloud-native-spring-data-cassandra-implementation-1-label:

Spring Data Cassandraを使ったアプリケーション実装(1)
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

|br|

Spring Data Cassandraで簡単なCRUDアプリケーションを作成して見ましょう。
Spring Data DynamoDBを使用するには、まず、Mavenプロジェクトのpom.xmlで、spring-data-cassandraのライブラリを定義します。
加えて、Datastax社から提供されているCassandraのDriverを定義してください。
なお、Apache Cassandraへのアクセスを画面から実行するアプリケーションを作成するために、Webアプリケーションを作るための
spring-boot-starter-web、thymeleafテンプレートエンジンを使用するためのspring-boot-starter-thymeleaf、
Cassandraテーブルのエンティティクラスの実装を簡易化するためにLombokライブラリを追加しておきましょう。

|br|

.. sourcecode:: xml

   <dependencies>
     <dependency>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter-data-cassandra</artifactId>
     </dependency>
     <dependency>
       <groupId>com.datastax.cassandra</groupId>
       <artifactId>cassandra-driver-core</artifactId>
     </dependency>
     <dependency>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter-web</artifactId>
     </dependency>
     <dependency>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter-thymeleaf</artifactId>
     </dependency>
     <dependency>
       <groupId>org.projectlombok</groupId>
       <artifactId>lombok</artifactId>
       <optional>true</optional>
     </dependency>
   </dependencies>

|br|

今回、アプリケーションは以下のコンポーネントで構成しますが、Spring Data cassandraで最低限作成しなければならない必須のクラスは以下の通りです。
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

   * - CassandraConfig
     - Cassandraへの接続に関する設定クラス
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
     - Cassandraのテーブルを表現するエンティティクラス
     - ◯

|br|

それでは、実装していくクラスを説明しますが、Cassandraのエンティティクラスや接続に関わるコンポーネント以外はほとんど :ref:`section-cloud-native-spring-data-dynamodb-implementation-1-label` のときと変わりません。
まず、最初にSpringBoot起動クラス及び、各種設定クラスです。@SpringBootApplicaitonアノテーションが付与された起動クラスは、
同一パッケージにある@Configurationアノテーションが付与された設定クラス及び、
設定クラス内で@ComponentScanされたパッケージにあるクラスを読み取ります。今回は目的に応じて以下の3つに分類して定義します。

* SpringMVCの設定クラスであるWebMvcConfigurerを実装した設定クラス：MvcConfigクラス
* Serviceをコンポーネントスキャンで読み取る設定クラス：DomainConfigクラス
* Cassandraの接続を行う設定クラス：CassandraConfigクラス

設定クラスは必ずしも複数である必要はなく一つにまとめても動作上問題ありませんが、クラス名と役割を対応づけて作成していた方が、
後々設定内容を混乱することなく、クラス名から識別できてベターです。

|br|

.. sourcecode:: java

   package org.debugroom.mynavi.sample.spring.data.cassandra.config;

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

   package org.debugroom.mynavi.sample.spring.data.cassandra.config;

   import org.springframework.context.annotation.ComponentScan;
   import org.springframework.context.annotation.Configuration;
   import org.springframework.web.servlet.config.annotation.ResourceHandlerRegistry;
   import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

   @ComponentScan("org.debugroom.mynavi.sample.spring.data.cassandra.app.web")
   @Configuration
   public class MvcConfig implements WebMvcConfigurer {

    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/static/**")
                .addResourceLocations("classpath:/static/");
    }

}
|br|

DomainConfigクラスではServiceなどビジネスロジックレイヤに属するコンポーネントをスキャンする定義を追加しておきます。

|br|

.. sourcecode:: java

   package org.debugroom.mynavi.sample.spring.data.cassandra.config;

   import org.springframework.context.annotation.ComponentScan;
   import org.springframework.context.annotation.Configuration;

   @ComponentScan("org.debugroom.mynavi.sample.spring.data.cassandra.domain.service")
   @Configuration
   public class DomainConfig {
   }

|br|

続いて、Cassandraへの接続で必須になる設定クラスがCassandraConfigクラスです。AbstractCassandraFonfigrationを継承すれば、
必要なクラスはSpringBootがデフォルトで自動構築してくれますが、加えて、以下4つのメソッドをオーバーライドします。

* getSchemaAction()メソッド：アプリケーション構築時のテーブルに対するアクションを決定する。
* getEntityBasePackages()メソッド：Cassandraテーブルのエンティティクラスとなるベースパッケージを指定する。
* getMetricsEnabled()メソッド：MetrixとしてJMXを使用するか決定する。Spring Data Cassandra2.0以降で、起動エラーとなるため、falseに設定しておく。
* getKeyspaceName()メソッド：CassandraテーブルでアクセスするKeyspace名を指定する。

加えて、@EnableCassandraRepositoriesアノテーションを付与し、CassandraテーブルへアクセスするRepositoryクラスがあるベースパッケージを同時に指定しなければなりません。
また、下記はシングルノードでCassandraを構築する場合の設定です。複数ノードでクラスタを構築する場合、さらに
cluster()メソッドを構築し、ContactPointとなるIPアドレスやポートを指定しなければなりません。ここでは、ローカル環境で必要な設定のみを
有効化するために、@Profileアノテーションを指定して、Profileで"dev"が指定した時にこの設定クラスが有効化するようにしておきましょう。
実装としては、クラスタの起動時に専用のプロファイルを指定するものとして、デフォルトでは"dev"が有効になるようにapplication.ymlにデフォルトプロファイルを指定指定しておきます。

|br|

.. sourcecode:: java

   package org.debugroom.mynavi.sample.spring.data.cassandra.config;

   import org.springframework.context.annotation.Configuration;
   import org.springframework.context.annotation.Profile;
   import org.springframework.data.cassandra.config.AbstractCassandraConfiguration;
   import org.springframework.data.cassandra.config.CassandraClusterFactoryBean;
   import org.springframework.data.cassandra.config.SchemaAction;
   import org.springframework.data.cassandra.repository.config.EnableCassandraRepositories;

   @Profile("dev")
   @Configuration
   @EnableCassandraRepositories("org.debugroom.mynavi.sample.spring.data.cassandra.domain.repository")
   public class CassandraConfig extends AbstractCassandraConfiguration {

       @Override
       public SchemaAction getSchemaAction() {
           return SchemaAction.CREATE_IF_NOT_EXISTS;
       }

       @Override
       public String[] getEntityBasePackages(){
           return new String[] {"org.debugroom.mynavi.sample.spring.data.cassandra.domain.model.entity"};
       }

       @Override
       protected boolean getMetricsEnabled() {
           return false;
       }

       @Override
       protected String getKeyspaceName() {
           return "sample";
       }
   }

|br|

application.yml

.. sourcecode:: bash

   spring:
     profiles:
       active: dev

|br|

アプリケーションの設定に関する実装はここまでです。引き続きアプリケーションのコンポーネントに関わる実装を進めていきます。

|br|

著者紹介
------------------------------------------------------------------

川畑 光平(KAWABATA Kohei)

.. figure:: img/aws-lambda-and-api-gateway/pic_image01.jpg


某システムインテグレータにて、金融機関システム業務アプリケーション開発・システム基盤担当を経て、現在はソフトウェア開発自動化関連の研究開発・推進に従事。

Red Hat Certified Engineer、Pivotal Certified Spring Professional、AWS Certified Solutions Architect Professional等の資格を持ち、アプリケーション基盤・クラウドなど様々な開発プロジェクト支援にも携わる。

`2019 APN AWS Top Engineers & Ambassadors <https://aws.amazon.com/jp/blogs/psa/japan-apn-ambassador-2019/>`_ 選出。

本連載記事の内容に対するご意見・ご質問は `Facebook <https://www.facebook.com/kohei.kawabata.5>`_ まで。
