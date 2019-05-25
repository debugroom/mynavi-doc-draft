.. include:: ../module.txt

.. _section-automation-infra-devops-sonarqube-label:

基盤・デプロイ自動化実践
==================================================================

マイクロサービスアーキテクチャの基盤・デプロイ自動化
-------------------------------------------------------------------------------------------------------------------------------------

|br|

本連載では、以下のイメージにならって、CodeBuild、SonarQubeを使った継続的インテグレーション(Continuous integration:CI)環境を実際に構築していきます。

|br|

.. figure:: img/automation_infra_devops_actual_experience/sample-continuous-integration.png
   :scale: 100%

|br|

今回は、Amazon RDS、アプリケーションロードバランサ(ALB)を設定して、ECSクラスタ上にオープンソースの静的解析ツールであるSonarQubeServerをECSコンテナとして構築します。

|br|

.. _section-sonarqube-overview-label:

SonarQubeの概要
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

SonarQubeは `SonarSource <https://www.sonarsource.com>`_ によって提供されているオープンソースの静的解析プラットフォームで、形式的なソースコードのバグや不適切な記法のチェック、カバレッジの計測などを行い、品質を可視化することが可能です。

解析できるプログラミング言語は、Java (Android含む), C#, PHP, JavaScript, TypeScript, C/C++, Ruby, Kotlin, Go, COBOL, PL/SQL, PL/I, ABAP, VB.NET, VB6, Python, RPG, Flex, Objective-C, Swift, CSS, HTML, and XMLなど
25を超える言語をサポートし、コードのバグ、Code Smell(深い問題を発生する可能性があるコード)、セキュリティ脆弱性を検出することができます。

また、SonarQubeはプラグインにより拡張が可能になっており、IntelliJ IDEAやEclipseなどの統合開発環境と連携するSonarlintプラグイン、MavenやGradleなどのライブラリ管理ツール向けのScannerプラグインに加え、
GitHubやGitLab、JIRAやJenkinsなどの各開発ツールとの統合プラグイン、チェックルールを追加する場合のFindBugsプラグインなど様々なものが提供されています。SonarQubeを構成する代表的なコンポーネントや機能、その特徴は以下の通りです。

|br|

.. list-table::
   :widths: 3, 4, 13

   * - コンポーネント
     - 機能
     - 特徴

   * - `SonarQubeServer <https://www.sonarqube.org>`_
     - ダッシュボード
     - SonarQubeの中核となるWebアプリケーション。解析対象ごとにプロジェクトを作成し、解析結果のサマリやIssue、解析のアクティビティが可視化できる。

   * -
     - QualityProfile
     - 解析する言語のチェックルールの定義、適用するプロジェクトへの設定を行う。ソースコードを実装する開発端末で、QualityProfileが設定されたSonarQubeServerプロジェクトを指定することで、複数の端末に同一のルールを適用できる。

   * -
     - QualityGate
     - 開発端末でソースコードをコミット・プッシュする場合やSonarScannnerでビルドする際に、プロジェクトに設定した静的チェックルール・メトリクスをパスしたか検証し、またその検証ルールを定義する機能。

   * -
     - MarketPlace
     - FindBugsなどの追加の解析ルールや他のサードパーティ製ツールとの統合プラグインなどのマーケットプレイスで検索・インストールを行う機能。プラグインの一覧は `こちら <https://docs.sonarqube.org/display/PLUG>`_ 。

   * - `Sonarlint <https://www.sonarlint.org>`_
     - ソースコード検証・サジェスト
     - IntelliJ IDEAやEclipseなどの統合開発環境にプラグインをインストールし、コーディングする際に検証やサジェスト、チェックルールの詳細表示などを行う機能

   * - `SonarScanner <https://docs.sonarqube.org/display/SCAN>`_
     - ソースコード解析・結果送信
     - MavenやGladle、Jenkinsなどを使ってアプリケーションをビルドする際に、プラグインを組み込み、ソースコードの解析・検証、カバレッジ測定などを実行して、SonarQubeServerへ結果を連携する機能。

|br|

SonarQubeには上記の機能を無料で使用可能なCommunity Editionや、ソースコードのブランチやプルリクエスト解析などの機能を追加したDeveloper Edtion、ポートフェリオ管理機能や高度なテクニカルサポートがあるEnterprize Edtion、
複数のSonarQubeServerをクラスターとして構築し、高可用性な運用ができるDataCenter Edtionがあります。今回はCommunity Editionを使って、以降環境構築を進めていきます。

|br|

.. _section-create-rds-for-sonarqube-label:

Amazon RDSを使ったSonarQubeServer用データベースの構築
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

SonarQubeServerは、サーバーの設定やプロジェクトの解析結果、組み込んだプラグイン、セキュリティ情報などを保存しておくリレーショナルデータベースが必要になります。データベースはOracle、PostgreSQL、MySQL、MS SQLServerなど
主要なデータベースエンジンをサポートしていますが、ここでは自動バックアップなど高可用性で運用が可能なAmazon RDS(PostgreSQL)を使ってデータベースを構築します。

RDS構築の要領は、筆者が執筆している別連載「AWSで作るクラウドネイティブアプリケーションの基本」の 「 `Amazon RDSにアクセスするSpringアプリケーション(1) <https://news.mynavi.jp/itsearch/article/devsoft/4422>`_  」を参照してください。
設定内容はリンク先の手順とほぼ同一でかまいませんが、「データベースの名前」だけ、「sonar」を設定するようにしてください。

|br|

.. figure:: img/automation_infra_devops_sonarqube/management-console-rds-create-database-pickup.png
   :scale: 100%

|br|

構築後は、sonarデータベースにアクセスするユーザを作成しましょう。

セキュリティグループでRDSのインバウンド接続が許可されたIPアドレスをもつEC2インスタンスなどで、PSQLクライアントなどをインストールし、
作成したRDSのエンドポイント、データベース、ユーザ名を指定してRDSに接続後、ユーザ名「sonar」、パスワード「sonar」とするユーザを作成します。

|br|

.. sourcecode:: bash

   [centos@ip-XXXX-XXX-XXX-XXX ~]$ psql -U username -d sonar -h mynavi-sample-db.xxxxxxx.ap-northeast-1.rds.amazonaws.com
   Password for user username:
   psql (9.4.0, server 10.6)
   sonar=> create role sonar with LOGIN password 'sonar';

|br|

.. note:: CentOSのPSQLインストール方法ですが、EC2でCentOS7を実行した場合は、以下のコマンドでPSQLをインストールできます。

   .. sourcecode:: bash

      [centos@ip-XXXX-XXX-XXX-XXX ~]$ sudo yum update -y
      // omit
      [centos@ip-XXX-XXX-XXX-XXX ~]$ sudo yum install -y postgresql

|br|

.. _section-create-alb-for-sonarqube-label:

SonarQubeServer向けのアプリケーションロードバランサーの構築
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

続いて、SonarQubeServerへ接続するためにアプリケーションロードバランサー(ALB)を経由して接続するように構築します。
ALBと接続するターゲットグループにECSコンテナを登録することにより、ヘルスチェック機能が有効になり、サーバが何らかの理由で落ちた場合も再起動が実行され、SonarQubeServerの可用性を向上させることができます。

ALBの構築は、筆者が執筆している別連載「AWSで作るクラウドネイティブアプリケーションの基本」の 「 `AWS ECS上に構築するSpringアプリケーション(2) <https://news.mynavi.jp/itsearch/article/devsoft/4359>`_ 」を適宜参考にしてください、今回、設定内容でポイントとなる項目は以下の通りです。

|br|

.. list-table::
   :widths: 3, 3, 4

   * - 設定箇所
     - 設定内容
     - 説明

   * - 基本設定：スキーム
     - インターネット向け(internet-facing)
     - 開発端末からSonarQubeServerへインターネット経由でアクセスするため、パブリックサブネット向けのALBを作成します。

   * - セキュリティグループの設定：セキュリティグループの割り当て
     - 新しいセキュリティグループを作成する
     - ポート80番の任意のIPからのアクセスを許可するセキュリティグループを新しく作成します。

   * - ルーティングの設定：ターゲットグループ
     - 新しいターゲットグループ
     - ターゲットグループを新規作成します。

   * - ルーティングの設定：ターゲットグループ
     - [ターゲットの種類]インスタンス　[プロトコル]HTTP [ポート]80
     - HTTPプロトコルでターゲットグループにアクセスします

   * - ルーティングの設定：ヘルスチェック
     - [プロトコル]HTTP　[パス]/ [間隔]300秒
     - SonarQubeServerの初回起動はデータベースの構築など時間を要するため、300[s]ほどにチェック間隔を設定してください。

   * - ターゲットの登録
     - [登録済みターゲット]特に設定しない
     - 後述のECSサービス作成時にターゲットの登録を行うので、ここでは何も設定せずに登録します。

|br|

.. _section-create-dockerfile-for-sonarqube-label:

SonarQubeを構築するDockerfile
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

次にSonarQubeServerをECSで構築するためのDockerfileを作成します。SonarQubeServerでデフォルトで提供されている起動スクリプトであるsonar.shは、
SonarQubeアプリケーションをスクリプト中で別プロセスでデーモンとして起動する形式になっているため、Dockerfileイメージでそのまま実行するとDockerプロセスとしてアプリケーションを起動することができません。
そこで、`SonarSourceから提供されているDockerfile <https://github.com/SonarSource/docker-sonarqube/blob/52981f8b2efdbfe18ffacaae687af7a4f05c15a5/7.7-community/Dockerfile>`_ を参考に、
CentOSでSonarQubeServerを構築するDockerfileを作成します。


|br|

.. sourcecode:: bash
   :caption: Backendアプリケーションのプロジェクトに作成するDockerfile

   # Dockerfile for sonarqube server
   # (A)
   FROM            docker.io/centos:latest
   # (B)
   MAINTAINER      debugroom

   # (C)
   RUN yum install -y java-1.8.0-openjdk zip unzip wget

   # (D)
   ENV SONAR_VERSION=7.7 \
       SONARQUBE_HOME=/usr/local/sonarqube \
       SONARQUBE_JDBC_USERNAME=sonar \
       SONARQUBE_JDBC_PASSWORD=sonar \
       SONARQUBE_JDBC_URL=jdbc:postgresql://mynavi-sample-sonarqube-db.xxxxxxxx.ap-northeast-1.rds.amazonaws.com/sonar

   # (E)
   EXPOSE 9000

   # (F)
   RUN groupadd -r sonarqube && useradd -r -g sonarqube sonarqube

   # (G)
   RUN set -x \
    && (gpg --batch --keyserver ha.pool.sks-keyservers.net --recv-keys F1182E81C792928921DBCAB4CFCA4A29D26468DE \
	    || gpg --batch --keyserver ipv4.pool.sks-keyservers.net --recv-keys F1182E81C792928921DBCAB4CFCA4A29D26468DE) \
    && curl -o sonarqube.zip -fSL https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-$SONAR_VERSION.zip \
    && curl -o sonarqube.zip.asc -fSL https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-$SONAR_VERSION.zip.asc \
    && gpg --batch --verify sonarqube.zip.asc sonarqube.zip \
    && unzip sonarqube.zip \
    && mv sonarqube-$SONAR_VERSION $SONARQUBE_HOME \
    && rm sonarqube.zip* \
    && rm -rf $SONARQUBE_HOME/bin/*

   # (H)
   WORKDIR $SONARQUBE_HOME
   # (I)
   COPY run.sh $SONARQUBE_HOME/bin/
   # (J)
   RUN chmod 744 $SONARQUBE_HOME/bin/run.sh
   # (K)
   RUN chown -R sonarqube:sonarqube $SONARQUBE_HOME
   # (L)
   USER sonarqube
   # (M)
   CMD ["./bin/run.sh"]

   |br|

.. list-table:: Dockerfileの実行コマンド
   :widths: 1, 19

   * - 項番
     - コマンド

   * - (A)
     - FROM centos:centos7

   * -
     - コンテナイメージのベースとなるOSイメージを指定します。

   * - (B)
     - MAINTAINER debugroom

   * -
     - メンテナンスしている組織や個人の名称を指定します。省略可能です。

   * - (C)
     - RUN yum install -y java-1.8.0-openjdk zip unzip wget

   * -
     - OSにJDK、zipコマンド、unzipコマンド、wgetコマンドをインストールしてます。

   * - (D)
     - ENV SONAR_VERSION=7.7 omit...

   * -
     - コンテナの環境変数にSonarQubeのバージョンやインストールするホームディレクトリ、ユーザ名、パスワード、RDSのエンドポイントを設定します。

   * - (E)
     - EXPOSE 9000

   * -
     - コンテナのポート9000をオープンします。

   * - (F)
     - RUN groupadd -r sonarqube && useradd -r -g sonarqube sonarqube

   * -
     - sonarqubeグループとsonarqubeユーザを作成します。

   * - (G)
     - RUN set -x omit...

   * -
     - SonarQubeServerをダウンロードして、$SONARQUBE_HOME配下へ展開します。

   * - (H)
     - WORKDIR $SONARQUBE_HOME

   * -
     - コマンド実行する起点となるディレクトリを$SONARQUBE_HOMEへ移します。

   * - (I)
     - COPY run.sh $SONARQUBE_HOME/bin/

   * -
     - SonarQubeServerアプリケーションを実行するスクリプトrun.shを$SONARQUBE_HOME/binへコピーします。

   * - (J)
     - RUN chmod 744 $SONARQUBE_HOME/bin/run.sh

   * -
     - run.shに実行権限を付与します。

   * - (K)
     - RUN chown -R sonarqube:sonarqube $SONARQUBE_HOME

   * -
     - $SONARQUBE_HOME配下にある全てのディレクトリ・ファイルの所有者権限をグループsonarqube、ユーザsonarqubeへ変更します。

   * - (L)
     - USER sonarqube

   * -
     - sonarqubeユーザへスイッチします。

   * - (L)
     - CMD ["./bin/run.sh"]

   * -
     - SonarQubeServer実行スクリプトを起動します。

|br|

また、SonarQubeServerを実行するrun.shスクリプトは以下の通りです。起動スクリプトはSonarSourceから提供されているものをほぼそのまま利用しますが、
javaコマンドでアプリケーションを起動するプロセスをフォアグラウンドで実行しているのがポイントです。

|br|

.. sourcecode:: bash

   #!/usr/bin/env bash

   set -e

   if [ "${1:0:1}" != '-' ]; then
     exec "$@"
   fi

   declare -a sq_opts

   while IFS='=' read -r envvar_key envvar_value
     do
       if [[ "$envvar_key" =~ sonar.* ]]; then
          sq_opts+=("-D${envvar_key}=${envvar_value}")
       fi
     done < <(env)

   exec java -jar lib/sonar-application-$SONAR_VERSION.jar \
     -Dsonar.log.console=true \
     -Dsonar.jdbc.username="$SONARQUBE_JDBC_USERNAME" \
     -Dsonar.jdbc.password="$SONARQUBE_JDBC_PASSWORD" \
     -Dsonar.jdbc.url="$SONARQUBE_JDBC_URL" \
     -Dsonar.web.javaAdditionalOpts="$SONARQUBE_WEB_JVM_OPTS -Djava.security.egd=file:/dev/./urandom" \
    "${sq_opts[@]}" \
    "$@"

|br|

こちらも `AWS ECS上に構築するSpringアプリケーション(4) <https://news.mynavi.jp/itsearch/article/devsoft/4390>`_ と同様の手順で、Dockerfile作成後にDockerがインストールされた適当な端末で、イメージを作成します。

|br|

.. sourcecode:: bash

   [centos@ip-XXX-XXX-XXX-XXX]$ docker build . -t debugroom/sonarqube-server:latest
      // omit
   [centos@ip-XXX-XXX-XXX-XXX ~]$ docker images
   REPOSITORY                                TAG                 IMAGE ID            CREATED             SIZE
   debugroom/sonarqube-server                latest              97e608f45e20        2 minites ago       973MB

|br|

Dockerイメージが作成されたら、Docker Hubへプッシュします。

.. sourcecode:: bash

   [centos@ip-XXX-XXX-XXX-XXX ~]$ docker login
   // omit  - enter USERID and password
   [centos@ip-XXX-XXX-XXX-XXX ~]$ docker push debugroom/sonarqube-server:latest


|br|

.. _section-create-ecs-for-sonarqube-label:

ECSクラスタ・タスク定義、サービスの実行
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

SonarQubeServerのDockerfileイメージをDockerHubへプッシュした後は、EC2起動型のECSクラスタおよびECSタスク定義を行い、コンテナをサービスとして実行します。
ECSクラスタの構築手順は `AWS ECS上に構築するSpringアプリケーション(5) <https://news.mynavi.jp/itsearch/article/devsoft/4405>`_ を、ECSタスク定義は `AWS ECS上に構築するSpringアプリケーション(6) <https://news.mynavi.jp/itsearch/article/devsoft/4408>`_ 、
ECSサービスの実行は `AWS ECS上に構築するSpringアプリケーション(7) <https://news.mynavi.jp/itsearch/article/devsoft/4416>`_ に記載してある要領を参考に進めてください。ただし、今回ポイントになる設定項目は以下の通りです。

|br|

.. list-table::
   :widths: 3, 3, 4

   * - 設定箇所
     - 設定内容
     - 説明

   * - ECSクラスタの作成：インスタンスの設定
     - 1
     - 今回はターゲットグループに１つのECSコンテナだけ起動するように設定します。複数のECSクラスタを実行場合はDataCenterEditionが必要になるので複数には設定しないようにしましょう。

   * - ECSクラスタの作成：ネットワーキング：セキュリティグループ
     - 新規作成：[プロトコル]HTTP、SSH [ポート]80、22 [ソース]ALBのセキュリティグループ、0.0.0.0/0
     - ECSクラスタにはALBからの通信をポート80番で静的マッピングするため、ソースを上記で作成したALBのセキュリティグループに設定します。またトラブルシューティング用にSSHのポートも開けておきます。

   * - ECSタスクの定義：タスクのサイズ：タスクメモリ
     - 2048MiB
     - SonarQubeには `ハードウェア要求スペック <https://docs.sonarqube.org/latest/requirements/requirements/>`_ として2GB程度のメモリを推奨しているため、相応のメモリを確保しましょう。

   * - ECSタスクの定義：タスクのサイズ：タスクCPU
     - 1 vcpu
     - 0.5 vcpu以下にすると、サービス起動時に時間がかかりすぎてヘルスチェックに引っかかる場合があるので、1vcpu程度取っておいた方がベターです。

   * - ECSタスクの定義：コンテナの追加：ポートマッピング
     - [ホストポート]80 [コンテナポート]9000
     - SonarQubeは9000番ポートでアプリケーションを起動するので、ALBからくるリクエストを9000番へ転送するように設定します。

   * - ECSサービスの実行：サービスの設定：サービスタイプ
     - REPLICA
     - 今回はターゲットグループに１つのECSコンテナだけ起動するようにサービスタイプをREPLICAでタスク数を1に設定しますが、DAEMONでも可です。

   * - ECSサービスの実行：サービスの設定：タスクの数
     - 1
     - 今回はターゲットグループに１つのECSコンテナだけ起動するように設定します。複数のサービスを実行する場合はDataCenterEditionが必要になるので複数には設定しないようにしましょう。

   * - ECSサービスの実行：ネットワーク構成：ヘルスチェックの猶予期間
     - 300
     - 上記のタスクサイズの設定で、アプリケーションの起動に60秒ほど要するため、余裕を持った値を設定しておきます。

   * - ECSサービスの実行：サービスの検出：サービスの検出の統合の有効化
     - チェックしない
     - 今回はDNSでALBのエンドポイントを利用するので、設定をオフにしておきます。

|br|

ECSサービスを実行した後、ALBのDNS名を使ってブラウザからアクセスできれば起動は成功です。

|br|

.. figure:: img/automation_infra_devops_sonarqube/sonarqube-top.png
   :scale: 100%

|br|

.. note:: 正しくコンテナが実行されない場合、ECSクラスタにSSHログインし、docker ps -aすることでコンテナの実行を確認できます。またコンテナ実行時のログは/var/log/ecs/ecs-agent.log_XXXXで、
          SonarQubeServerの起動のログはコンテナの/usr/local/sonarqube/logsに各種アプリケーション起動ログが出力されるよう設定されています。トラブルシューティングはこうしたログを参照してください。

|br|


次回はSonarQubeServerでプロジェクトを作成し、QualityProfileを設定、開発端末のIDEで環境構築してみます。


|br|

著者紹介
------------------------------------------------------------------

川畑 光平(KAWABATA Kohei) - NTTデータ 課長代理

.. figure:: img/automation_infra_devops_overview/pic_image01.jpg
   :scale: 100%

金融機関システム業務アプリケーション開発・システム基盤担当を経て、現在はソフトウェア開発自動化関連の研究開発・推進に従事。

Red Hat Certified Engineer、Pivotal Certified Spring Professional、AWS Certified Solutions Architect Professional等の資格を持ち、アプリケーション基盤・クラウドなど様々な開発プロジェクト支援にも携わる。
