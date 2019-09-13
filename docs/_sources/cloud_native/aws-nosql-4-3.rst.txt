.. include:: ../module.txt

.. _section-cloud-native-nosql-label-4-3:

AWSで作るクラウドネイティブアプリケーションの基本
========================================================================================

.. _section-cloud-native-nosql-4th-3-label:

第3回 AWS上に構築するNoSQLアプリケーション(4)-3
----------------------------------------------------------------------------------------

|br|

.. _section-cloud-native-nosql-spring-applicaiton-4-3-label:

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

   * Amazon DynamoDBの概要及び構築と認証情報の作成
   * Spring Data DynamoDBを用いたアプリケーション(1)
   * Spring Data DynamoDBを用いたアプリケーション(2)

#. Apache CassandraへアクセスするSpringアプリケーション

   * Apache Cassandraの概要及びローカル環境構築
   * Spring Data Cassandraを用いたアプリケーション(1)
   * Spring Data Cassandraを用いたアプリケーション(2)

#. Amazon ElastiCacheへアクセスするSpringアプリケーション

   * AmazonElasiCacheの概要及びローカル環境でのRedisServer構築
   * Spring SessionとSpring Data Redisを用いたアプリケーション(1)
   * **Spring SessionとSpring Data Redisを用いたアプリケーション(2)**
   * Amazon ElastiCacheの設定
   * セッション共有するECSアプリケーションの構築(1)
   * セッション共有するECSアプリケーションの構築(2)

|br|

前回 :ref:`section-cloud-native-spring-session-data-redis-implementation-1-label` に引き続き、 今回もSpring SessionとSpring Data Redisを使ってセッション情報を共有するアプリケーションを実装していきます。

|br|

.. _section-cloud-native-spring-session-data-redis-implementation-2-label:

Spring SessionとSpring Data Redisを使ったアプリケーション実装(2)
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

|br|

アプリケーションコンポーネントの実装に移ります。@SessionScopeアノテーションを付与し、セッションに格納するJavaオブジェクトを以下の通り実装します。
また、コンポーネントスキャンの対象となるよう、@Componentを付与し、前回設定クラスで実装した@ComponentScanで指定したパッケージ配下に作成してください。
今回セッションが共有することが分かるよう、変数としてホスト名と最終更新日時をもつJavaオブジェクトとします。

|br|

セッションに格納するオブジェクト

.. sourcecode:: java

   package org.debugroom.mynavi.sample.aws.elasticache.app.web.model;

   import java.io.Serializable;
   import java.util.Date;

   import org.springframework.stereotype.Component;
   import org.springframework.web.context.annotation.SessionScope;

   import lombok.AllArgsConstructor;
   import lombok.Builder;
   import lombok.Data;
   import lombok.NoArgsConstructor;

   @SessionScope
   @Component
   @AllArgsConstructor
   @NoArgsConstructor
   @Builder
   @Data
   public class SampleSession implements Serializable {

       private String host;
       private Date lastUpdatedAt;

   }

|br|

Controllerでは、リクエストを受け取り、上記のオブジェクトの変数である、ホスト名と最新日時を更新した後、セッションに保存して、結果をテンプレートに渡す処理を実装します。
オブジェクトには@SessionScopeアノテーションが付与されているので、Controllerクラスで対象オブジェクトを@Autowiredすれば、固有のセッションを取得できます。
オブジェクトのパラメータを更新するだけで自動的にセッション内のデータも更新されるようになります。

また、次回以降、AWS ECSを使って複数のコンテナ上で、当アプリケーションを実行しますが、異なるコンテナ上で実行されている複数のアプリケーションが共通して、
同じセッションデータを保存しているElastiCacheにアクセスしていることが分かるように、リクエストのパスによって異なるコンテナへアクセスするよう、ALBのパスベースルーティングを利用します。
画面から指定するcontainerGroupというパラメータでアプリケーションロードバランサーが実行するコンテナを振り分けて指定できるように、セッションを保存するURLにはパスパターンにパラメータを含めておきましょう。

その他、セッションを無効化する処理をもつメソッドも合わせて実装しておきます。

|br|

.. sourcecode:: java

   package org.debugroom.mynavi.sample.aws.elasticache.app.web;

   import javax.servlet.http.HttpSession;
   import java.util.Date;

   import org.springframework.beans.factory.annotation.Autowired;
   import org.springframework.beans.factory.annotation.Value;
   import org.springframework.stereotype.Controller;
   import org.springframework.ui.Model;
   import org.springframework.web.bind.annotation.RequestMapping;
   import org.springframework.web.bind.annotation.RequestMethod;

   import org.debugroom.mynavi.sample.aws.elasticache.app.web.model.SampleSession;

   @Controller
   public class SampleController {

       @Value("${app.host}")
       private String host;
       @Autowired
       private SampleSession sampleSession;

       @RequestMapping(method=RequestMethod.GET, value="/{containerGroup:[0-9]+}")
       public String sharedSession(Model model){
           sampleSession.setHost(host);
           sampleSession.setLastUpdatedAt(new Date());
           model.addAttribute("sampleSession", sampleSession);
           return "sharedSession";
       }

       @RequestMapping(method=RequestMethod.GET, value="/invalidateSession")
       public String invalidateSession(HttpSession session){
           session.invalidate();
           return "redirect:/index.html";
       }

   }

|br|

ホスト名は、ECSコンテナに設定された環境変数HOSTNAMEから取得できるように、application.ymlで下記のように設定しておきます。

|br|

application.yml

.. sourcecode:: bash

   app:
     host: ${HOSTNAME:localhost}

|br|


これでアプリケーションが作成しました。SpringBoot起動クラスを実行し、アプリケーションを実行しましょう。「http://localhost:8080/index.html」へブラウザからアクセスすると以下の様な画面が表示されます。

|br|

.. figure:: img/aws-nosql/elasticache-app.png


|br|

実行するコンテナの番号を入力(ローカル環境では1つのAPしかないので意味はない)、「Shared Session」ボタンを押下すると、Redisにセッション情報が追加されます。

|br|

.. figure:: img/aws-nosql/elasticache-app-sharedSession.png


|br|

redis-cliなどを用いて、Redisのキー一覧を参照すると以下のように表示されるはずです。hgetallコマンドを用いてセッションに保存されたオブジェクトを表示させてみます。

|br|

.. sourcecode:: bash

   127.0.0.1:6379> keys *
   1) "spring:session:sessions:4e48e07b-bb62-46ea-8666-8fe8437d6d56"
   2) "spring:session:sessions:expires:4e48e07b-bb62-46ea-8666-8fe8437d6d56"
   3) "spring:session:expirations:1552045800000"

   127.0.0.1:6379> hgetall "spring:session:sessions:4e48e07b-bb62-46ea-8666-8fe8437d6d56"
   1) "lastAccessedTime"
   2) "1552043971610"
   3) "maxInactiveInterval"
   4) "1800"
   5) "creationTime"
   6) "1552043970153"
   7) "sessionAttr:scopedTarget.sampleSession"
   8) "{\"@class\":\"org.debugroom.mynavi.sample.aws.elasticache.app.web.model.SampleSession\",\"host\":\"localhost\",\"lastUpdatedAt\":[\"java.util.Date\",1552043970167]}"

|br|

ローカル環境に構築したRedisにアクセスし、セッション情報を保存するアプリケーションを実装することができました。次回は、AWSでElastiCacheを構築してみます。

|br|

著者紹介
------------------------------------------------------------------

川畑 光平(KAWABATA Kohei)

.. figure:: img/aws-lambda-and-api-gateway/pic_image01.jpg


某システムインテグレータにて、金融機関システム業務アプリケーション開発・システム基盤担当を経て、現在はソフトウェア開発自動化関連の研究開発・推進に従事。

Red Hat Certified Engineer、Pivotal Certified Spring Professional、AWS Certified Solutions Architect Professional等の資格を持ち、アプリケーション基盤・クラウドなど様々な開発プロジェクト支援にも携わる。

`2019 APN AWS Top Engineers & Ambassadors <https://aws.amazon.com/jp/blogs/psa/japan-apn-ambassador-2019/>`_ 選出。

本連載記事の内容に対するご意見・ご質問は `Facebook <https://www.facebook.com/kohei.kawabata.5>`_ まで。
