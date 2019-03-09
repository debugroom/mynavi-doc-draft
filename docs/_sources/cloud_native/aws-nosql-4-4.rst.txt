.. include:: ../module.txt

.. _section-cloud-native-nosql-label-4-4:

AWSで作るクラウドネイティブアプリケーションの基本
========================================================================================

.. _section-cloud-native-nosql-4th-4-label:

第3回 AWS上に構築するNoSQLアプリケーション(4)-4
----------------------------------------------------------------------------------------

|br|

.. _section-cloud-native-nosql-spring-applicaiton-4-4-label:

Amazon ElastiCacheへアクセスするSpringアプリケーション
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

クラウド時代が到来し、ビッグデータやキーバリュー型データなどで、ますます活用の機会が広がりつつあるNoSQLデータベース。第3回は代表的なNoSQLプロダクトであるAmazon DynamoDBやApache Cassandra、
Amazon ElastiCacheへアクセスするSpringアプリケーションを構築する方法を説明します。本連載では、以下の様なステップで進めていきます。

|br|

#. NoSQLデータベースの特徴とデータ特性

   * CAP定理を元にしたデータベースの分類とデータ特性
   * AP型データベースAmazon DynamoDBとApache Cassandraの特徴

#. Amazon DynamoDBへアクセスするSpringアプリケーション

   * Amazon DynamoDBの概要及び構築と認証情報の設定
   * Spring Data DynamoDBを用いたアプリケーション(1)
   * Spring Data DynamoDBを用いたアプリケーション(2)

#. Apache CassandraへアクセスするSpringアプリケーション

   * ローカル環境におけるApache Cassandraの構築
   * Spring Data Cassandraを用いたアプリケーション(1)
   * Spring Data Cassandraを用いたアプリケーション(2)

#. Amazon ElastiCacheへアクセスするSpringアプリケーション

   * ローカル環境におけるRedisの構築
   * Spring SessionとSpring Data Redisを用いたアプリケーション(1)
   * Spring SessionとSpring Data Redisを用いたアプリケーション(2)
   * Amazon ElastiCacheの設定                                   …◯
   * セッション共有するECSアプリケーションの構築(1)
   * セッション共有するECSアプリケーションの構築(2)

|br|

前回 :ref:`section-cloud-native-spring-session-data-redis-implementation-2-label` では、Spring SessionとSpring Data Redisを使ってセッション情報を共有するアプリケーションを実装し、
ローカル環境に構築したRedisへのアクセスまで確認しました。今回はAmazon ElastiCache(Redis)を実際に構築してみましょう。

|br|

.. _section-cloud-native-create-elasticache-redis-label:

Amazon ElastiCache(Redis)の構築
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

|br|

ElastiCacheを作成するにあたり、下記のイメージ図のように、事前にElastiCacheノードを配置するためのVPCを構築しておく必要があります。

|br|

.. figure:: img/aws-nosql/architecture-elasticache.png
   :scale: 100%

|br|

VPCの作成方法は、第２回 :ref:`section-cloud-native-ecs-overview-label` に詳細な構築手順・要領をまとめていますので、必要に応じてリンク先を確認し、VPCを作成しておいてください。
ただし、上記のイメージに従えば、NATゲートウェイやインターネットゲートウェイ、プライベートサブネットは今回必ずしも必要ではありません。
パブリックサブネットだけの構築でも入力内容は変わりませんので、適宜リンク先の要領を参考にしてVPCを作成してください。

また、ElastiCacheはセキュリティグループによりアクセス制御されます。セキュリティ上、VPCネットワーク内からのアクセスのみ受け付ける設定が望ましいため、VPC作成後に、以下の手順通り、ElastiCacheに割り当てるセキュリティグループを作成してください。

AWSコンソールの「EC2」サービスから、セキュリティグループメニューを選択肢、「セキュリティグループの作成」ボタンを押下し、以下の要領に則って入力します。

* セキュリティグループ名：任意のセキュリティグループ名を入力します。
* 説明：セキュリティグループで任意の説明文言を入力します。
* VPC：作成したVPCを選択します。
* セキュリティグループのルール

  * タイプ：カスタムTCPを選択します。
  * プロトコル：TCPを入力します。
  * ポート範囲：Redisのデフォルトポートである6379を設定します。
  * ソース：VPCのCIDRを設定します。


|br|

.. figure:: img/aws-nosql/management-console-ec2-create-security-group-for-elasticache-1.png
   :scale: 100%

|br|

セキュリティグループを作成した後、AWSコンソールの「ElastiCache」サービスから、「テーブルの作成」ボタンを押下し、以下の要領に則って入力します。

* クラスターエンジン：Redisを選択します。

  * クラスターモードが有効：シャーディングを有効化するクラスターモードを設定します。特にサンプルではシャーディングは必要ないのでここではチェックしません。

* Redisの設定

  * 名前：20文字内で任意のRedisクラスタ名を入力します。
  * 説明：任意の説明文言を入力します。
  * エンジンバージョンの互換性：最新バージョンを選択します。
  * ポート：デフォルト6279を選択します。
  * パラメータグループ：Redisで使用するパラメータグループを指定します。詳細は `こちら <https://docs.aws.amazon.com/ja_jp/AmazonElastiCache/latest/red-ug/ParameterGroups.Redis.html>`_ を参照してください。
  * ノードのタイプ：キャッシュのスペックを選択します。
  * レプリケーション数：リードレプリカの数を指定します。

* Redisの詳細設定

  * 自動フェイルオーバーを備えたマルチAZ：マルチAZ構成にする場合チェックします。
  * サブネットグループ：新規作成を選択します。

    * 名前：フェイルオーバーのための複数のサブネット構成に対する任意のグループ名を入力します。
    * 説明：任意の説明文言を入力します。
    * VPC ID：作成しているVPCのIDを選択します。
    * サブネット：Redisレプリカを配置するVPCサブネットを複数選択します。

  * 優先アベイラビリティゾーン：フェイルオーバーで優先するAZを選択します。

* セキュリティ

  * セキュリティグループ：上記で作成したセキュリティグループを指定します。
  * 保管時の暗号化：Redis内のデータを暗号化する必要がある場合チェックします。
  * 送信中の暗号化：Redisとの通信時に暗号化を行う場合チェックします。

* クラスタへのデータのインポート

  * 構築時にデータを設定する場合、S3にデータを配置し、オブジェクトキーを指定します。

* バックアップ

  * 自動バックアップの有効化：Redis内のデータをバックアップする場合チェックします。
  * バックアップ保存期間：バックアップの保存期間を設定します。
  * バックアップウィンドウ：バックアップ期間を細かく指定する場合にチェックします。

* メンテナンス

  * メンテナンスウィンドウ：メンテナンス期間を指定する場合にチェックします。
  * SNS通知のトピック：重要なクラスターイベントの通知が送信されるよう設定する場合はトピックを設定します。

|br|

.. figure:: img/aws-nosql/management-console-elasticache-create-1.png
   :scale: 100%

|br|

.. figure:: img/aws-nosql/management-console-elasticache-create-2.png
   :scale: 100%

|br|

.. figure:: img/aws-nosql/management-console-elasticache-create-3.png
   :scale: 100%

|br|

以上で、ElastiCacheの作成は完了です。次回はロードバランサー及びアプリケーションをECSコンテナ化してデプロイし、セッション情報をRedisに共有するよう設定します。


著者紹介
------------------------------------------------------------------

川畑 光平(KAWABATA Kohei)

.. figure:: img/aws-lambda-and-api-gateway/pic_image01.jpg
   :scale: 100%

某システムインテグレータにて、金融機関システム業務アプリケーション開発・システム基盤担当を経て、現在はソフトウェア開発自動化関連の研究開発・推進に従事。

Red Hat Certified Engineer、Pivotal Certified Spring Professional、AWS Certified Solutions Architect Professional等の資格を持ち、アプリケーション基盤・クラウドなど様々な開発プロジェクト支援にも携わる。

本連載記事の内容に対するご意見・ご質問は `Facebook <https://www.facebook.com/kohei.kawabata.5>`_ まで。
