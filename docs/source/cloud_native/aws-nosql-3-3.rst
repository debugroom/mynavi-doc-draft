.. include:: ../module.txt

.. _section-cloud-native-nosql-label-3-3:

AWSで作るクラウドネイティブアプリケーションの基本
========================================================================================

.. _section-cloud-native-nosql-3rd-3-label:

第3回 AWS上に構築するNoSQLアプリケーション(3)-3
----------------------------------------------------------------------------------------

|br|

.. _section-cloud-native-nosql-spring-applicaiton-3-3-label:

Apache CassandraへアクセスするSpringアプリケーション
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

クラウド時代が到来し、ビッグデータやキーバリュー型データなどで、ますます活用の機会が広がりつつあるNoSQLデータベース。第3回は代表的なNoSQLプロダクトであるAmazon DynamoDBやApache Cassandra、
Amazon Elastic CacheへアクセスするSpringアプリケーションを構築する方法を説明します。本連載では、以下の様なステップで進めていきます。

|br|

#. NoSQLデータベースの特徴とデータ特性

   * CAP定理を元にしたデータベースの分類とデータ特性
   * AP型データベースAmazon DynamoDBとApache Cassandraの特徴

#. Amazon DynamoDBへアクセスするSpringアプリケーション

   * Amazon DynamoDBの概要及び構築と認証情報の設定
   * Spring Data DynamoDBを用いたアプリケーション(1)
   * Spring Data DynamoDBを用いたアプリケーション(2)

#. Apache CassandraへアクセスするSpringアプリケーション

   * AWS ECS上に構築するApache Cassandra
   * Spring Data Cassandraを用いたアプリケーション(1)
   * Spring Data Cassandraを用いたアプリケーション(2)

#. Amazon Elastic CacheへアクセスするSpringアプリケーション

   * Amazon Elastic Cacheの設定
   * Spring Cloud AWSを用いたアプリケーション

|br|

前回、:ref:`section-cloud-native-spring-data-cassandra-implementation-1-label` に引き続き、 今回はSpring Data Cassandraを使ってデータベースアクセスするアプリケーションを実装していていきます。

|br|


Spring Data Cassandraを使ったアプリケーション実装(2)
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

|br|

アプリケーションコンポーネントの実装に移ります。画面から受け取るリクエストパラメータコンポーネントを以下の様に作成しています。

|br|

リクエストパラメータオブジェクト

.. sourcecode:: java

   package org.debugroom.mynavi.sample.spring.data.cassandra.app.model;

   import lombok.AllArgsConstructor;
   import lombok.Builder;
   import lombok.Data;
   import lombok.NoArgsConstructor;

   import java.io.Serializable;

   @AllArgsConstructor
   @NoArgsConstructor
   @Builder
   @Data
   public class SampleModel implements Serializable {

       private String samplePartitionColumn;
       private String sampleClusterColumn;
       private String sampleText;

   }

|br|

Controllerでは、以下5種類のリクエストを受け取り、ロジックを実行して結果をテンプレートに渡す処理を実装します。

* データを全件取得するServiceを実行し、実行結果をfindAll.htmlのテンプレートへ渡す処理。
* 指定したパーティションキー・ソートキーをもつデータを取得するServiceを実行し、実行結果をfindOne.htmlのテンプレートへ渡す処理
* データ追加するServiceを実行し、実行結果をadd.htmlのテンプレートに渡す処理。
* 指定したパーティションキー・ソートキーのデータを更新するServiceを実行し、実行結果update.htmlのテンプレートへ渡す処理
* 指定したパーティションキー・ソートキーのデータを削除するServiceを実行し、実行結果delete.htmlのテンプレートへ渡す処理

|br|

.. sourcecode:: java

   package org.debugroom.mynavi.sample.spring.data.cassandra.app.web;

   import org.debugroom.mynavi.sample.spring.data.cassandra.app.model.SampleModel;
   import org.debugroom.mynavi.sample.spring.data.cassandra.app.model.SampleModelMapper;
   import org.debugroom.mynavi.sample.spring.data.cassandra.domain.service.SampleService;
   import org.springframework.beans.factory.annotation.Autowired;
   import org.springframework.stereotype.Controller;
   import org.springframework.ui.Model;
   import org.springframework.web.bind.annotation.RequestMapping;
   import org.springframework.web.bind.annotation.RequestMethod;

   @Controller
   public class SampleController {

       @Autowired
       SampleService sampleService;

       @RequestMapping(method = RequestMethod.GET, value="findAll")
       public String findAll(Model model){
           model.addAttribute("mynaviSampleTables", sampleService.getMynaviSampleTables());
           return "findAll";
       }

       @RequestMapping(method = RequestMethod.GET, value="findOne")
       public String findOne(SampleModel sampleModel, Model model){
           model.addAttribute("mynaviSampleTable", sampleService.getMynaviSampleTable(
                SampleModelMapper.mapToMynaviSampleTableKey(sampleModel)));
           return "findOne";
       }

       @RequestMapping(method = RequestMethod.POST,  value="add")
       public String add(SampleModel sampleModel, Model model){
           model.addAttribute("mynaviSampleTable",
                sampleService.addMynaviSampleTable(SampleModelMapper.map(sampleModel)));
           return "add";
       }

       @RequestMapping(method = RequestMethod.POST, value = "update")
       public String update(SampleModel sampleModel, Model model){
           model.addAttribute("mynaviSampleTable",
                sampleService.updateMynaviSampleTable(SampleModelMapper.map(sampleModel)));
           return "update";
       }

       @RequestMapping(method = RequestMethod.POST, value = "delete")
       public String delete(SampleModel sampleModel, Model model){
           model.addAttribute("mynaviSampleTable",
                sampleService.deleteMynaviSampleTable(
                        SampleModelMapper.mapToMynaviSampleTableKey(sampleModel)));
           return "delete";
       }
   }

|br|

また、Serviceを呼び出す際にリクエストパラメータオブジェクトをServiceのインプットオブジェクトとなっているCassandraのテーブルクラスやプライマリキークラスへ変換するマッパーをインターフェースで実装します。

.. sourcecode:: java

   package org.debugroom.mynavi.sample.spring.data.cassandra.app.model;

   import org.debugroom.mynavi.sample.spring.data.cassandra.domain.model.entity.MynaviSampleTable;
   import org.debugroom.mynavi.sample.spring.data.cassandra.domain.model.entity.MynaviSampleTableKey;

   public interface SampleModelMapper {

       public static MynaviSampleTable map(SampleModel sampleModel){
           return MynaviSampleTable.builder()
                .mynaviSampleTableKey(MynaviSampleTableKey.builder()
                    .partitionColumn(sampleModel.getSamplePartitionColumn())
                    .clusterColumn(sampleModel.getSampleClusterColumn()).build())
                .sampleText(sampleModel.getSampleText())
                .build();
       }

       public static MynaviSampleTableKey mapToMynaviSampleTableKey(SampleModel sampleModel){
           return MynaviSampleTableKey.builder()
                .partitionColumn(sampleModel.getSamplePartitionColumn())
                .clusterColumn(sampleModel.getSampleClusterColumn())
                .build();
       }

   }

|br|

.. note:: Java8からstaticメソッドであれば、インターフェースでも実装できる様になっています。

|br|

Serviceの実装は、以下の通り、CRUD処理をRepositoryを通して実行します。

* findAll：作成したDynamoDBのテーブルの全件データをList型で受け取る処理
* findOne：指定したパーティションキー・ソートキーでデータを取得する処理
* add：パーティションキーにランダムなUUID文字列を、ソートキーには"1"を設定し、リクエストから受け取ったテキスト文字列を設定して保存するロジックです。
* update：指定したパーティションキー・ソートキーのテキストデータを更新する処理
* delete：指定したパーティションキー・ソートキーのデータを削除する処理

|br|

.. sourcecode:: java

   package org.debugroom.mynavi.sample.spring.data.dynamodb.domain.service;

   import org.springframework.beans.factory.annotation.Autowired;
   import org.springframework.stereotype.Service;

   @Service
   public class SampleServiceImpl implements SampleService{

       @Autowired
       SampleRepository sampleRepository;

       @Override
       public MynaviSampleTable getMynaviSampleTable(MynaviSampleTableKey mynaviSampleTableKey) {
           return sampleRepository.findById(mynaviSampleTableKey).get();
       }

       @Override
       public List<MynaviSampleTable> getMynaviSampleTables() {
           List<MynaviSampleTable> sampleTables = new ArrayList<>();
           sampleRepository.findAll().iterator().forEachRemaining(sampleTables::add);
           return sampleTables;
       }

       @Override
       public MynaviSampleTable addMynaviSampleTable(MynaviSampleTable mynaviSampleTable) {
           mynaviSampleTable.setSamplePartitionKey(UUID.randomUUID().toString());
           mynaviSampleTable.setSampleSortKey("1");
           return sampleRepository.save(mynaviSampleTable);
       }

       @Override
       public MynaviSampleTable updateMynaviSampleTable(MynaviSampleTable mynaviSampleTable) {
           return sampleRepository.save(mynaviSampleTable);
       }

       @Override
       public MynaviSampleTable deleteMynaviSampleTable(MynaviSampleTableKey mynaviSampleTableKey) {
           MynaviSampleTable mynaviSampleTable = sampleRepository.findById(mynaviSampleTableKey).get();
           sampleRepository.deleteById(mynaviSampleTableKey);
           return mynaviSampleTable;
       }
   }

|br|

ここで、DynamoDBへアクセスするコンポーネントであるSampleRepositoryインターフェースは、以下の様な要領で実装しておく必要があります。

* org.springframework.data.repository.CrudRepositoryを継承する
* テーブルをモデル化したクラスとキーをCrudRepositoryの型パラメータに設定する。

|br|

.. sourcecode:: java

   package org.debugroom.mynavi.sample.spring.data.cassandra.domain.repository;

   import org.springframework.data.repository.CrudRepository;

   import org.debugroom.mynavi.sample.spring.data.cassandra.domain.model.entity.MynaviSampleTable;
   import org.debugroom.mynavi.sample.spring.data.cassandra.domain.model.entity.MynaviSampleTableKey;

   public interface SampleRepository extends CrudRepository<MynaviSampleTable, MynaviSampleTableKey>{
   }

|br|

.. note:: 特に実装クラスも作成せずインターフェースだけで、findAllやsaveメソッドが実行できる理由は、Spring Data Cassandraが、GenericDAOパターンに基づく実装クラスを提供しているためです。
   GenericDAOパターンとはJavaの型パラメータを利用した実装で、CRUD処理をJavaのジェネリクス機構を使って実装しておき、テーブルの種類に応じて、
   型パラメータを設定することで、汎用的なCRUD共通処理を実装したDAO(DatabaseAccessObject)を作成しておくパターンです。
   Spring Data Cassandraに限らず、Spring Data JPAなど類似一連のプロダクトでは、同様にインターフェースを作成するだけで、
   基本的なCRUDはほぼ実装せずにデータベースアクセス処理を行うことが可能です。

|br|

型パラメータとして指定する、Cassandraのテーブルクラスは以下の要領に則って作成します。

* テーブルクラスに@Tableアノテーションを付与し、前回作成したCassandraテーブルのテーブル名を指定します。
* パーティションカラムに限らず、クラスタカラムも定義してプライマリキーとする場合はプライマリキークラスを作成します。
* テーブルクラスにコンストラクタやGetter、Setterメソッドを付与する実装を省略する目的でLombokアノテーションを付与します。
* プライマリキークラスには@PrimaryKeyColumnでPrimaryKeyType.PARTITIONEDを、クラスタカラムには@PrimaryKeyColumnでPrimaryKeyType.CLUSTEREDをを付与します。
* その他、追加したい属性には@Columnアノテーションを付与します。

|br|

.. sourcecode:: java

   package org.debugroom.mynavi.sample.spring.data.cassandra.domain.model.entity;


   import lombok.AllArgsConstructor;
   import lombok.Builder;
   import lombok.Data;
   import lombok.NoArgsConstructor;
   import org.springframework.data.cassandra.core.mapping.Column;
   import org.springframework.data.cassandra.core.mapping.PrimaryKey;
   import org.springframework.data.cassandra.core.mapping.Table;

   import java.io.Serializable;

   @AllArgsConstructor
   @NoArgsConstructor
   @Builder
   @Data
   @Table("mynavi_sample_table")
   public class MynaviSampleTable implements Serializable {

       @PrimaryKey("mynaviSampleTableKey")
       private MynaviSampleTableKey mynaviSampleTableKey;

       @Column("sample_text")
       private String sampleText;

   }

.. sourcecode:: java

   package org.debugroom.mynavi.sample.spring.data.cassandra.domain.model.entity;

   import lombok.AllArgsConstructor;
   import lombok.Builder;
   import lombok.Data;
   import lombok.NoArgsConstructor;

   import org.springframework.data.cassandra.core.cql.PrimaryKeyType;
   import org.springframework.data.cassandra.core.mapping.PrimaryKeyClass;
   import org.springframework.data.cassandra.core.mapping.PrimaryKeyColumn;

   import java.io.Serializable;

   @AllArgsConstructor
   @NoArgsConstructor
   @Builder
   @Data
   @PrimaryKeyClass
   public class MynaviSampleTableKey implements Serializable {

       @PrimaryKeyColumn(name="partition_column", ordinal = 0, type = PrimaryKeyType.PARTITIONED)
       private String partitionColumn;
       @PrimaryKeyColumn(name="cluster_column", ordinal = 1, type = PrimaryKeyType.CLUSTERED)
       private String clusterColumn;

   }

|br|

これでアプリケーションが作成しました。SpringBoot起動クラスを実行し、アプリケーションを実行しましょう。「http://localhost:8080/index.html」へブラウザからアクセスすると以下の様な画面が表示され、以下の５つのサービスが実行できます。

* 任意のテキストデータを入力し、「add Data」ボタンを押下すると、データが追加されます。パーティションキーはUUID、ソートキーは"1"固定です。
* 「find All Data」ボタンを押下すると、全てのデータが取得されます。
* パーティションキーとソートキーを指定して「find One Data」ボタンを押下すると、該当のデータが取得されます。
* パーティションキーとソートキーを指定して、テキストデータを入力し、「Update Data」ボタンを押下すると、該当のデータが更新されます。
* パーティションキーとソートキーを指定して「Delete Data」ボタンを押下すると、該当のデータが削除されます。

|br|

.. figure:: img/aws-nosql/cassandra-app.png
   :scale: 100%

|br|

.. figure:: img/aws-nosql/cassandra-app-add.png
   :scale: 100%

|br|

.. figure:: img/aws-nosql/cassandra-app-findAll.png
   :scale: 100%

|br|

.. figure:: img/aws-nosql/cassandra-app-findOne.png
   :scale: 100%

|br|

.. figure:: img/aws-nosql/cassandra-app-update.png
   :scale: 100%

|br|

.. figure:: img/aws-nosql/cassandra-app-delete.png
   :scale: 100%

|br|

このように、CassandraへCRUD処理するアプリケーションをSpring Data Cassandraを用いて簡単に実装することができます。
実際のアプリケーションでは様々ユースケースに応じて、データモデルを検討しておく必要がありますが、応用編等で追々その辺りを触れたいと思います。

|br|

著者紹介
------------------------------------------------------------------

川畑 光平(KAWABATA Kohei)

.. figure:: img/aws-lambda-and-api-gateway/pic_image01.jpg
   :scale: 100%

某システムインテグレータにて、金融機関システム業務アプリケーション開発・システム基盤担当を経て、現在はソフトウェア開発自動化関連の研究開発・推進に従事。

Red Hat Certified Engineer、Pivotal Certified Spring Professional、AWS Certified Solutions Architect Professional等の資格を持ち、アプリケーション基盤・クラウドなど様々な開発プロジェクト支援にも携わる。

本連載記事の内容に対するご意見・ご質問は `Facebook <https://www.facebook.com/kohei.kawabata.5>`_ まで。
