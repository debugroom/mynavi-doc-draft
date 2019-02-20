.. include:: ../module.txt

.. _section-cloud-native-nosql-label-3-1:

AWSで作るクラウドネイティブアプリケーションの基本
========================================================================================

.. _section-cloud-native-nosql-3rd-1-label:

第3回 AWS上に構築するNoSQLアプリケーション(3)-1
----------------------------------------------------------------------------------------

|br|

.. _section-cloud-native-nosql-spring-applicaiton-3-1-label:

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

前回は、AP型データベースである、Amazon DynamoDBへデータベースアクセスするSpringアプリケーションを構築しました。
今回はApache Cassandraの概要を説明し、実際に構築して見ます。

|br|

Apache Cassandraの概要
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

|br|

Apache CassandraはAmazon Dynamoの分散デザインと、Google Bigtableのデータモデルを併せ持つ非中央集権型で伸縮自在なスケール、高可用性、耐障害性、設定可能な一貫性を備えたカラム指向型データベースです。
基本的にAP型データベースのアーキテクチャを踏襲しており、結果整合性をもつデータベースとよく呼ばれますが、正しくは、設定可能な一貫性を持つデータベースであり、Cassndaraに対しては正確ではありません。
一貫性はACIDの定義に従えば、読み込むクライアントがお互いに矛盾したことなる値を参照することなく、データの状態がある状態へと正しく遷移することですが、
一貫性にはレベルがあり、Cassandraでは、テーブルごとにクライアントが全ての更新において参照するレプリカ数をレプリケーションファクタによって制御できるよう設定できます。

Cassandraのデータ構造はバージョンによって異なります。 Cassandraはキーバリュー型のデータ(カラム)を一連にまとめたRowのまとまりであるカラムファミリ及び、
それらをノード単位でまとめるキースペースををもつ多次元ハッシュテーブルのデータ構造を持っています。v0.7より前では、DynamoDBと同じく、どのRowも１つかそれ以上のカラムを持ちながら、
他のRowと全く同じカラムを持つ必要がないスキーマフリーな構造をとっていましたが、v0.7以降オプションでスキーマ定義が可能となり、v1.1以降は、スキーマ定義が必須となりました。
いくつかの古いドキュメントでは、このv0.7以降を前提とした記述になっているので、参照する際には注意が必要です。

また、CassandraはSQLに似たCQL(Cassandra Query Language)という、データベース操作言語を持っています。
「 :ref:`section-cloud-native-nosql-feature-label` 」でも記載した通り、NoSQL型の特性を理解して
SQLとの差異を踏まえて実装する必要があるのですが、ユーザはこの言語を用いて、SQLライクにデータアクセスのための処理を記述することができます。

では、Cassandra環境を構築してみましょう。

|br|

Apache Cassandraの構築
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

相応のメモリが必要になるのですが、ローカル環境でシングルノードのCassandraを構築すると開発時は便利なので、ここでは、
Dockerを用いてゲストOSとして、CentOS7上にCassandraをインストールします。ホストOSとなる端末はMacOS、Windowsどちらでもかまいませんが、
Dockerを事前にインストールしてください。以降はDockerがインストールされた端末を前提に進めていきます。

|br|

適当な任意のフォルダにDockerfileと言う名前で以下のファイルを作成してください。

.. sourcecode:: bash

   # Dockerfile for cassandra

   # (A)
   FROM            docker.io/centos:latest
   # (B)
   MAINTAINER      debugroom

   # (C)
   RUN yum install -y \
      java-1.8.0-openjdk \
      java-1.8.0-openjdk-devel \
      wget tar iproute

   # (D)
   ENV JAVA_HOME /etc/alternatives/java_sdk
   # (E)
   ADD datastax.repo /etc/yum.repos.d/
   # (F)
   RUN yum -y install dsc30
   # (G)
   RUN yum -y install cassandra30-tools
   # (H)
   RUN sed -i s/\#-Xms4G/-Xms1G/g /etc/cassandra/conf/jvm.options
   # (I)
   RUN sed -i s/\#-Xmx4G/-Xmx2G/g /etc/cassandra/conf/jvm.options
   # (J)
   RUN sed -i s/seeds\:\ \"127\.0\.0\.1\"/seeds\:\ \"172\.17\.0\.2\"/g /etc/cassandra/conf/cassandra.yaml
   # (K)
   RUN sed -i s/listen\_address\:\ localhost/listen\_address\:\ 172\.17\.0\.2/g /etc/cassandra/conf/cassandra.yaml
   # (L)
   RUN sed -i s/rpc\_address\:\ localhost/\rpc\_address\:\ 172\.17\.0\.2/g /etc/cassandra/conf/cassandra.yaml
   # (M)
   RUN systemctl enable cassandra
   # (N)
   ADD create-keyspace.cql /root/
   # (O)
   EXPOSE 7199 7000 7001 9160 9042 22 8012 61621

|br|

.. list-table:: ローカル環境にあるCentOS7Dockerコンテナ上で、シングルノードCassandraを構築するDockerfileの実行コマンド
   :widths: 1, 19

   * - 項番
     - コマンド

   * -
     - 説明

   * - (A)
     - FROM docker.io/centos:latest

   * -
     - コンテナイメージのベースとなるOSイメージを指定します。

   * - (B)
     - MAINTAINER debugroom

   * -
     - メンテナンスしている組織や個人の名称を指定します。省略可能です。

   * - (C)
     - RUN yum install -y java-1.8.0-openjdk java-1.8.0-openjdk-devel wget tar iproute

   * -
     - OSにJDK、wgetコマンド、tarコマンド、iprouteコマンドをインストールしてます。

   * - (D)
     - ENV JAVA_HOME /etc/alternatives/jre

   * -
     - JAVA_HOME環境変数をインストールされたJDKのディレクトリに設定しています。

   * - (E)
     - ADD datastax.repo /etc/yum.repos.d/

   * -
     - /etc/yum.repos.d ディレクトリに、Datastax社のレポジトリファイルを作成し、yumレポジトリを追加します。

   * - (F)
     - RUN yum -y install dsc30

   * -
     - Apache Cassandra3.0系をインストールしています。

   * - (G)
     - RUN yum -y install cassandra30-tools

   * -
     - nodetoolなど周辺ツールをインストールしています。

   * - (H)
     - RUN sed -i s/\#-Xms4G/-Xms1G/g /etc/cassandra/conf/jvm.options

   * -
     - Cassandraが使用する最小Javaメモリサイズのプロパティをsedコマンドで置換しています。ここでは1GBに設定しています。ローカル端末の環境に合わせて適宜設定してください。

   * - (I)
     - RUN sed -i s/\#-Xmx4G/-Xmx2G/g /etc/cassandra/conf/jvm.options

   * -
     - Cassandraが使用する最大Javaメモリサイズのプロパティをsedコマンドで置換しています。ここでは2GBに設定しています。ローカル端末の環境に合わせて適宜設定してください。

   * - (J)
     - RUN sed -i s/seeds\:\ \"127\.0\.0\.1\"/seeds\:\ \"172\.17\.0\.2\"/g /etc/cassandra/conf/cassandra.yaml

   * -
     - CassandraのGossipネットワークのハブとして機能するSeedサーバのIPアドレスを指定します。CentOSのDockerイメージのデフォルトIPアドレスである172.17.0.2に設定してください。

   * - (K)
     - RUN sed -i s/listen\_address\:\ localhost/listen\_address\:\ 172\.17\.0\.2/g /etc/cassandra/conf/cassandra.yaml

   * -
     - CassandraのリスナーとなるサーバのIPアドレスを指定します。CentOSのDockerイメージのデフォルトIPアドレスである172.17.0.2に設定してください。

   * - (L)
     - RUN sed -i s/rpc\_address\:\ localhost/\rpc\_address\:\ 172\.17\.0\.2/g /etc/cassandra/conf/cassandra.yaml

   * -
     - CassandraのRPCとなるサーバのIPアドレスを指定します。CentOSのDockerイメージのデフォルトIPアドレスである172.17.0.2に設定してください。

   * - (M)
     - RUN systemctl enable cassandra

   * -
     - Dockerコンテナ実行時にCassandraのサービスが起動するよう、systemctlコマンドでcassandra.serviceを登録しておきます。

   * - (N)
     - ADD create-keyspace.cql /root/

   * -
     - Dockerコンテナを実行してCassandraのサービスが上がった後に、キースペースを作成するCQLファイルをコンテナ内にコピーします。

   * - (O)
     - EXPOSE 7199 7000 7001 9160 9042 22 8012 61621

   * -
     - CassandraでアプリケーションやCQLクライアントおよびノード間通信するポートを解放しておきます。

|br|

.. warning:: Seedサーバやリスナー、RPCのIPアドレスでlocalhostやループバックアドレスを指定していると、ホストOSなどからCQLでアクセスする際接続ができなくなるので、必ず実行しているコンテナのIPアドレスを指定してください。

|br|

また、上記のDockerfile内で使用するDatastax社のレポジトリファイルおよび、Cassandra起動後にキースペースを作成するCQLのファイルをDockerfileと同一ディレクトリ内に作成してください。

|br|

.. sourcecode:: bash

   [datastax]
   name = DataStax Repo for Apache Cassandra
   baseurl = http://rpm.datastax.com/community
   enabled = 1
   gpgcheck = 0

|br|

.. sourcecode:: bash

   CREATE KEYSPACE sample WITH replication = {'class': 'SimpleStrategy', 'replication_factor' : 1};

|br|

上記のファイルを作成後、コマンドラインからDockerイメージを作成するコマンドを実行します。docker buildコマンドでイメージ名やタグを指定して、元になるDockerfileがあるディレクトリを指定してください。
ここでは、DockerHub上でのxxxxxxレポジトリにsample-cassandra-localというイメージをlatestタグをつけて実行しています。最後にDockerfileがあるカレントディレクトリを指定しています。

.. sourcecode:: bash

   docker build -t xxxxxx/sample-cassandra-local:latest .

|br|

Dockerイメージが作成されたら、コマンドラインからDockerプロセスを実行してください。なお、実行するコンテナ内でsystemctlサービスを実行するために、--privilegedオプションを付与する必要があります。
実行コンテナ名称(cassandra-server1とします。)と、コンテナイメージ名、コンテナの起動時は/sbin/initを実行し、systemdを起動する形でコンテナを実行します。

.. sourcecode:: bash

   docker run -d --privileged -p 9042:9042 --name cassandra-server1 xxxxxxx/sample-cassandra-local /sbin/init

|br|

起動したコンテナにアクセスし、以下の通り、起動したCassandraでキースペースを作成します。キースペースが作成されているか確認しましょう。

.. sourcecode:: bash

   docker exec -ti cassandra-server1 /bin/bash
   cqlsh 172.17.0.2 9042 -f /root/create-keyspace.cql
   cqlsh 172.17.0.2 9042
   cql> describe sample;

|br|

これで、Cassandraの環境が構築できました。次回はこの構築した環境にアクセスするSpringアプリケーションを実装してみます。

|br|

著者紹介
------------------------------------------------------------------

川畑 光平(KAWABATA Kohei)

.. figure:: img/aws-lambda-and-api-gateway/pic_image01.jpg
   :scale: 100%

某システムインテグレータにて、金融機関システム業務アプリケーション開発・システム基盤担当を経て、現在はソフトウェア開発自動化関連の研究開発・推進に従事。

Red Hat Certified Engineer、Pivotal Certified Spring Professional、AWS Certified Solutions Architect Professional等の資格を持ち、アプリケーション基盤・クラウドなど様々な開発プロジェクト支援にも携わる。

本連載記事の内容に対するご意見・ご質問は `Facebook <https://www.facebook.com/kohei.kawabata.5>`_ まで。
