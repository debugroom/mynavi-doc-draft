.. include:: ../module.txt

.. _section-cloud-native-nosql-label-1-1:

AWSで作るクラウドネイティブアプリケーションの基本
========================================================================================

.. _section-cloud-native-rds-1st-label:

Amazon RDSにアクセスするSpringアプリケーション(1)
----------------------------------------------------------------------------------------

|br|

クラウド上のリレーショナルデータベースとして、AWSで利用可能なAmazon RDS。今回はPostgreSQLをベースとするAmazon RDSへアクセスする
Springアプリケーションの実装方法についてわかりやすく解説します。なお、データベースアクセスの実装にはSpring Data JPAおよび、
Spring Cloud AWSを用いて実装します。

本連載では、以下のステップで解説を進めていきます。

|br|

#. Amazon RDSの概要とデータベース構築
#. Spring Data JPAおよびSpring Cloud AWSを用いたRDSアクセスアプリケーションの実装

|br|

なお、本連載は以下の前提知識がある開発者を想定しています。

|br|

.. list-table::
   :widths: 3, 7

   * - 対象読者
     - 前提知識

   * - エンタープライズ開発者
     - Java言語及びSpringFrameworkを使った開発に従事したことがある経験者。経験がなければ、|br|
       `こちらのチュートリアル <http://terasolunaorg.github.io/guideline/5.4.1.RELEASE/ja/Tutorial/index.html>`_ を実施することを推奨します。TERASOLUNAはSpringのベストプラクティスをまとめた開発方法論で、このチュートリアルでは、JavaやSpring Frameworkを使った開発に必要な最低限の知識を得ることができます。

   * -
     - GitHubなどのバージョン管理ツールやApache Mavenなどのライブラリ管理ツールを使った開発に従事したことがある経験者。

   * - AWS開発経験者
     - AWSアカウントをもち、コンソール上で各サービスを実行したことがある経験者

|br|

また、動作環境は以下のバージョンで実施しています。

|br|


.. list-table::
   :widths: 5, 5

   * - 動作対象
     - バージョン

   * - Java
     - 1.8

   * - Spring Boot
     - 2.1.2.RELEASE

   * - Spring Data JPA
     - 5.0.1

   * - Spring Cloud AWS
     - 2.1.2.RELEASE

|br|

将来的には、AWSコンソール上の画面イメージや操作、バージョンアップによりJavaのソースコード内で使用するクラスが異なる可能性があります。

|br|


.. _section-cloud-native-rds-overview-label:

Amazon RDSの概要
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

Amazon RDS(Relational Database Service)は冗長化構成によるフェイルオーバや自動バックアップ機能など、
高い可用性を保ちつつも、簡易なセットアップで運用が容易な、時間単位の従量課金制フルマネージドリレーショナルデータベースです。
RDSでは以下のデータベースエンジンを選択可能です。

* Amazon Aurora
* PostgreSQL
* MySQL
* MariaDB
* Oracle
* Microsoft SQL Server

RDS構築時に、複数のアベイラビリティゾーンをまたぎ、リードレプリカを構築できるオプションがあり、
マスターに障害が発生した場合は、スレーブであるリードレプリカがマスターに昇格します。リードレプリカ作成時には、
マスター・スレーブ各々IPが割り振られますが、RDSは単一のエンドポイントを有しているため、
データベース参照側は特にフェイルオーバを意識せずアクセスすることが可能です。データの書き込み時はリードレプリカの
書き込みも含まれるので、読み取りの一貫性は保証されます。また、リードレプリカはクロスリージョンサポートがあるため、
ディザスターリカバリ(災害によるリカバリ)としても活用できます。

RDSの自動バックアップ機能ではDBスナップショットとトランザクションログを保存しますが、スナップショットは
1日1回、指定した世代数を指定した時間帯に取得します。リストアを行う際は任意の世代のDBスナップショットを指定して
復元を行いますが、トランザクションログは随時取得されているので、任意に指定した状態へのリカバリが可能です。
時刻は直近5分から、最新のスナップショットの時点の間で指定することができます。

セキュリティ面のポイントとして、RDSはセキュリティグループによってアクセス制御されます。また、データの暗号化機能も
サポートされており、暗号化キー管理サービス「Key Management Service(KMS)」に格納されたキーを使って暗号化できます。

|br|

.. _section-cloud-native-rds-setup-label:

Amazon RDSの構築
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

それでは、実際にRDSを構築してみましょう。AWSコンソールのサービスから「RDS」を選択し、「データベースの作成」ボタンを押下します。

|br|

.. figure:: img/aws-rds/management-console-rds-create-database-1.png
   :scale: 100%

|br|

今回はデータベースエンジンをPostgreSQLを選択します。

|br|


.. figure:: img/aws-rds/management-console-rds-create-database-2.png
   :scale: 100%

|br|

ユースケースでは、「開発／テスト」を選択します。

|br|


.. figure:: img/aws-rds/management-console-rds-create-database-3.png
   :scale: 100%

|br|

以下の表を参考に、データベースの詳細を指定して、「次へボタン」を押下します。

|br|

.. list-table:: RDS DB詳細の指定
   :widths: 3, 7

   * - 入力項目
     - 説明

   * - ライセンスモデル
     - データベースのライセンスを指定します。PostgreSQLではposgresql-licenseの選択になります。

   * - DBエンジンのバージョン
     - データベースのバージョンを指定します。

   * - DBインスタンスのクラス
     - データベースを実行するインスタンスのCPUやメモリなどを指定します。

   * - マルチAZ配置
     - データベースを複数のアベイラビリティゾーンで配置する場合チェックします。

   * - ストレージタイプ
     - データベースストレージを選択します。オプションは `DBインスタンスストレージ <https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/UserGuide/CHAP_Storage.html#Concepts.Storage.GeneralSSD>`_ を参考にしてください。

   * - ストレージ割り当て
     - データベースストレージに割り当てるデータ容量を指定します。

   * - DBインスタンス識別子
     - データベースを作成するリージョン内でユニークな名称を指定します。

   * - マスターユーザの名前
     - データベースにアクセスする場合の管理者ユーザ名を指定します。

   * - マスターパスワード
     - データベースにアクセスする場合の管理者ユーザのパスワードを指定します。


|br|

.. figure:: img/aws-rds/management-console-rds-create-database-4.png
   :scale: 100%

|br|

.. figure:: img/aws-rds/management-console-rds-create-database-5.png
   :scale: 100%

|br|

以下の表を参考に、データベースの詳細設定を設定して「データベースの作成」を押下します。

|br|

.. list-table:: RDS DB詳細の指定
   :widths: 3, 7

   * - 入力項目
     - 説明

   * - Virtual Private Cloud(VPC)
     - RDSを配置するVPCを指定します。

   * - サブネットグループ
     - DBを配置するサブネットを複数まとめたものをサブネットグループとして定義します。Amazon RDSは、VPCにDBインスタンスを作成すると、DBサブネットグループから選択したIPアドレスを使用して、DBインスタンスにネットワークインターフェイスを割り当てます

   * - パブリックアクセシビリティ
     - 指定したVPCの外からアクセスする際にチェックします。「はい」を設定するとDBインスタンスにパブリックIPアドレスが割り当てられます。

   * - アベイラビリティゾーン
     - DBインスタンスを作成するアベイラビリティゾーンを指定します。

   * - VPCセキュリティグループ
     - DBインスタンスへのアクセス制御に関するセキュリティグループを指定します。新規を選択する場合、パブリックアクセシビリティをを有効にしているとコンソールにアクセスしている端末のIPアドレスが許可されたセキュリティグループが作成されます。

   * - データベースの名前
     - データベース名を指定します。

   * - ポート
     - データベースで使用するポートを指定します。

   * - DBパラメータグループ
     - データベースに適用するパラメータをまとめて規定したパラメータグループを指定します。

   * - オプショングループ
     - OracleまたはSQL Server、MySQLで、オプションの機能を有効にするDBオプショングループを選択します。

   * - IAM DB認証
     - IAMユーザやロールでデータベースのユーザ認証を実行する場合に有効化します。

   * - 暗号化
     - データベースインスタンスのストレージを暗号化する場合有効化します。

   * - バックアップの保存期間
     - DBインスタンスの自動バックアップの保存日数を指定します。最大35日まで指定可能です。

   * - バックアップウィンドウ
     - DBインスタンスの自動バックアップの実行時間帯を任意に設定します。

   * - 拡張モニタリング
     - プロセスの仮想メモリサイズやCPU使用率などモニタリングオプションを有効化します。具体的には `拡張モニタリング <https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/UserGuide/USER_Monitoring.OS.html>`_ を参照してください。

   * - モニタリングルール
     - 拡張モニタリングを実行するIAMロールを指定します。新規作成の場合rds-monitoring-roleが作成されます。

   * - パフォーマンスインサイト
     - RDSインスタンスの負荷をモニタリングし、データベースパフォーマンスの分析とトラブルシューティングを行うことができるパフォーマンスインスタンスを有効化します。
       パフォーマンスインサイトに関する詳細は `Amazon RDS Performance Insightsの使用 <https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/UserGuide/USER_PerfInsights.html>`_ を参考にしてください。

   * - 保持期間
     - パフォーマンスインサイトのデータ保持期間を指定します。

   * - マスターキー
     - パフォーマンスインサイトデータを保持するボリュームの暗号化キーを指定します。

   * - ログのエクスポート
     - Amazon CloudWatch Logsへ送信するログを選択します。

   * - マイナーバージョン自動アップグレード
     - マイナーバージョンの自動アップグレードを有効化します。なお、アップグレード時には数分ほどのダウンタイムが発生します。

   * - 削除保護の有効化
     - データベースの削除オペレーションを無効化するか指定します。


|br|

.. figure:: img/aws-rds/management-console-rds-create-database-6.png
   :scale: 100%

|br|

.. figure:: img/aws-rds/management-console-rds-create-database-7.png
   :scale: 100%

|br|

.. figure:: img/aws-rds/management-console-rds-create-database-8.png
   :scale: 100%

|br|

.. figure:: img/aws-rds/management-console-rds-create-database-9.png
   :scale: 100%

|br|

.. figure:: img/aws-rds/management-console-rds-create-database-10.png
   :scale: 100%

|br|

これでRDS DBインスタンスが作成できました。次回はこのデータベースにアクセスするSpringアプリケーションを実装してみます。

|br|


著者紹介
------------------------------------------------------------------

川畑 光平(KAWABATA Kohei)

.. figure:: img/aws-lambda-and-api-gateway/pic_image01.jpg
   :scale: 100%

某システムインテグレータにて、金融機関システム業務アプリケーション開発・システム基盤担当を経て、現在はソフトウェア開発自動化関連の研究開発・推進に従事。

Red Hat Certified Engineer、Pivotal Certified Spring Professional、AWS Certified Solutions Architect Professional等の資格を持ち、アプリケーション基盤・クラウドなど様々な開発プロジェクト支援にも携わる。

本連載記事の内容に対するご意見・ご質問は `Facebook <https://www.facebook.com/kohei.kawabata.5>`_ まで。
