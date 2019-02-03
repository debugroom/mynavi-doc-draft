.. include:: ../module.txt

.. _section-cloud-native-ecs-label-4:

AWSで作るクラウドネイティブアプリケーションの基本
========================================================================================

.. _section-cloud-native-ecs-4th-label:

第2回 AWS ECS上に構築するSpringアプリケーション(4)
----------------------------------------------------------------------------------------

|br|

クラウド時代が到来し、ますます広がりを見せつつあるコンテナ技術。第2回は、AWS ECS上でSpringアプリケーションを構築する方法を説明します。本稿は以下のステップに沿って、解説しています。

#. VPC(Virtual Private Cloud)環境の構築
#. アプリケーションロードバランサ(ALB)の作成
#. Springを使用したコンテナアプリケーションの実装方法
#. Dockerコンテナの作成・DockerHubへのプッシュ
#. ECSクラスタの作成
#. ECSタスクの定義
#. ECSサービスの定義

前回の記事「 :ref:`section-cloud-native-ecs-3rd-label` 」までに、以下イメージの構成に沿って、VPC環境・ALBを構築し、ECSコンテナ上で動くパブリック・プライベートサブネット用２種類のアプリケーションを作成しました。
今回は「4. Dockerコンテナの作成・DockerHubへのプッシュ」です。

|br|

.. figure:: img/aws-ecs/ecs-architecture.png
   :scale: 100%

|br|

(4)Dockerコンテナの作成・DockerHubへのプッシュ
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

ECSの実態はDockerコンテナなので、前回実装したSpringアプリケーションのコンテナイメージを作成し、DockerHubやAWS Elastic Container Registry(ECR)などのレジストリにプッシュしておくと、
次回以降後述する、ECSクラスタ・タスク・サービスの定義で、レジストリからコンテナイメージを取得して、ECSコンテナとして実行することができるようになります。

それでは、前回実装したSpringアプリケーションのプロジェクトを元にDockerイメージを作成していきます。Dockerイメージを作成する場合は、Dockerfileと呼ばれる定義ファイルを作成します。

前回作成したSpringBootアプリケーションのMavenプロジェクト内に、各々以下のようなDockerファイルを作成してコミットし、GitHubへプッシュします。

.. sourcecode:: bash
   :caption: Backendアプリケーションのプロジェクトに作成するDockerファイル

   # Dockerfile for sample service using embedded tomcat server

   # (A)
   FROM centos:centos7
   # (B)
   MAINTAINER debugroom

   # (C)
   RUN yum install -y \
       java-1.8.0-openjdk \
       java-1.8.0-openjdk-devel \
       wget tar iproute git

   # (D)
   RUN wget http://repos.fedorapeople.org/repos/dchen/apache-maven/epel-apache-maven.repo -O /etc/yum.repos.d/epel-apache-maven.repo
   # (E)
   RUN sed -i s/\$releasever/6/g /etc/yum.repos.d/epel-apache-maven.repo
   # (F)
   RUN yum install -y apache-maven
   # (G)
   ENV JAVA_HOME /etc/alternatives/jre
   # (H)
   RUN git clone https://github.com/debugroom/mynavi-sample-aws-ecs.git /var/local/mynavi-sample-aws-ecs
   # (I)
   RUN mvn install -f /var/local/mynavi-sample-aws-ecs/pom.xml

   # (J)
   RUN cp /etc/localtime /etc/localtime.org
   # (K)
   RUN ln -sf  /usr/share/zoneinfo/Asia/Tokyo /etc/localtime

   # (L)
   EXPOSE 8080

   # (M)
   CMD java -jar -Dspring.profiles.active=production /var/local/mynavi-sample-aws-ecs/backend/target/mynavi-sample-aws-ecs-backend-0.0.1-SNAPSHOT.jar

|br|

.. sourcecode:: bash
   :caption: BFFアプリケーションのプロジェクトに作成するDockerファイル

   # Dockerfile for sample service using embedded tomcat server

   # (A)
   FROM centos:centos7
   # (B)
   MAINTAINER debugroom

   # (C)
   RUN yum install -y \
          java-1.8.0-openjdk \
          java-1.8.0-openjdk-devel \
          wget tar iproute git

   # (D)
   RUN wget http://repos.fedorapeople.org/repos/dchen/apache-maven/epel-apache-maven.repo -O /etc/yum.repos.d/epel-apache-maven.repo
   # (E)
   RUN sed -i s/\$releasever/6/g /etc/yum.repos.d/epel-apache-maven.repo
   # (F)
   RUN yum install -y apache-maven
   # (G)
   ENV JAVA_HOME /etc/alternatives/jre
   # (H)
   RUN git clone https://github.com/debugroom/mynavi-sample-aws-ecs.git /var/local/mynavi-sample-aws-ecs
   # (I)
   RUN mvn install -f /var/local/mynavi-sample-aws-ecs/pom.xml

   # (J)
   RUN cp /etc/localtime /etc/localtime.org
   # (K)
   RUN ln -sf  /usr/share/zoneinfo/Asia/Tokyo /etc/localtime

   # (L)
   EXPOSE 8080

   # (M)
   CMD java -jar -Dspring.profiles.active=production /var/local/mynavi-sample-aws-ecs/backend-for-frontend/target/mynavi-sample-aws-ecs-backend-for-frontend-0.0.1-SNAPSHOT.jar

|br|

内容は２つともほぼ同一のため、BackendアプリケーションプロジェクトのDockerfileを元に解説します。

|br|

.. list-table:: Dockerfileの実行コマンド
   :widths: 1, 19

   * - 項番
     - コマンド

   * -
     - 説明

   * - (A)
     - FROM centos:centos7

   * -
     - コンテナイメージのベースとなるOSイメージを指定します。

   * - (B)
     - MAINTAINER debugroom

   * -
     - メンテナンスしている組織や個人の名称を指定します。省略可能です。

   * - (C)
     - RUN yum install -y java-1.8.0-openjdk java-1.8.0-openjdk-devel wget tar iproute git

   * -
     - OSにJDK、wgetコマンド、tarコマンド、iprouteコマンド、gitをインストールしてます。

   * - (D)
     - RUN wget http://repos.fedorapeople.org/repos/dchen/apache-maven/epel-apache-maven.repo -O /etc/yum.repos.d/epel-apache-maven.repo

   * -
     - Apache Mavenのインストールに必要な資材をwgetコマンドで取得しています。

   * - (E)
     - RUN sed -i s/\$releasever/6/g /etc/yum.repos.d/epel-apache-maven.repo

   * -
     - /etc/yum.repos.d/epel-apache-maven.repoのリリースバージョンを置換しています。

   * - (F)
     - RUN yum install -y apache-maven

   * -
     - Apache Mavenをインストールしています。

   * - (G)
     - ENV JAVA_HOME /etc/alternatives/jre

   * -
     - JAVA_HOME環境変数をインストールされたJDKのディレクトリに設定しています。

   * - (H)
     - RUN git clone https://github.com/debugroom/mynavi-sample-aws-ecs.git /var/local/mynavi-sample-aws-ecs

   * -
     - コミットしたSpringアプリケーションを/var/local/mynavi-sample-aws-ecsへgit cloneしています。ディレクトリは適宜変更しても可能です。

   * - (I)
     - RUN mvn install -f /var/local/mynavi-sample-aws-ecs/pom.xml

   * -
     - アプリケーションをmvn installコマンドでビルドしています。

   * - (J)
     - RUN cp /etc/localtime /etc/localtime.org

   * -
     - アプリケーションの実行環境のタイムゾーンを変更するため、タイムゾーンファイルを一度バックアップします。

   * - (K)
     - RUN ln -sf  /usr/share/zoneinfo/Asia/Tokyo /etc/localtime

   * -
     - シンボリックリンクを上書き作成して、アプリケーション実行環境のタイムゾーンを東京に変更しています。

   * - (L)
     - EXPOSE 8080

   * -
     - コンテナのポート8080をオープンします。

   * - (M)
     - CMD java -jar -Dspring.profiles.active=production /var/local/mynavi-sample-aws-ecs/backend-for-frontend/target/mynavi-sample-aws-ecs-backend-for-frontend-0.0.1-SNAPSHOT.jar

   * -
     - (I)でビルドしたSpringBootアプリケーションをプロファイルproductionで実行しています。

|br|

上記のDockerfileを、DockerがインストールされたLinuxマシン上で、docker buildコマンドを実行することによりDockerイメージを作成することができます。

|br|

.. note:: CentOSのDockerインストール方法について

   以下の通り、EC2上にLaunchされたCentOS7では、以下のコマンドでDockerをインストールできます。

   .. sourcecode:: bash

      [centos@ip-XXXX-XXX-XXX-XXX ~]$ sudo yum update -y
      // omit
      [centos@ip-XXX-XXX-XXX-XXX ~]$ sudo yum install -y docker
      // omit
      [centos@ip-XXX-XXX-XXX-XXX ~]$ sudo systemctl enable docker.service
      Created symlink from /etc/systemd/system/multi-user.target.wants/docker.service to /usr/lib/systemd/system/docker.service.
      [centos@ip-XXX-XXX-XXX-XXX ~]$ sudo systemctl start docker.service

|br|

.. note:: dockerコマンドの実行について

   dockerコマンドを利用する際は sudoが必要です。 sudoなしでdockerコマンドを実行するには、dockerグループを作成し、実行ユーザを追加する必要があります。

   .. sourcecode:: bash

      [centos@ip-XXX-XXX-XXX-XXX ~]$ sudo groupadd docker
      [centos@ip-XXX-XXX-XXX-XXX ~]$ sudo gpasswd -a $USER docker
       Adding user centos to group docker
      [centos@ip-XXX-XXX-XXX-XXX ~]$ sudo systemctl restart docker
      [centos@ip-XXX-XXX-XXX-XXX ~]$ exit

|br|

.. note:: CentOSのGitインストール方法について

   以下の通り、EC2上にLaunchされたCentOS7では、以下のコマンドでGitをインストールできます。インストールディレクトリは任意に変更可能です。

   .. sourcecode:: bash

      [centos@ip-XXXX-XXX-XXX-XXX ~]$ sudo yum -y install wget make gcc perl-ExtUtils-MakeMaker curl-devel expat-devel gettext-devel openssl-devel zlib-devel autoconf
      // omit
      [centos@ip-XXXX-XXX-XXX-XXX ~]$ cd /var/local/
      [centos@ip-XXXX-XXX-XXX-XXX ~]$ sudo wget https://www.kernel.org/pub/software/scm/git/git-2.9.5.tar.gz
      // omit
      [centos@ip-XXXX-XXX-XXX-XXX ~]$ sudo tar xvzf git-2.9.5.tar.gz
      // omit
      [centos@ip-XXXX-XXX-XXX-XXX ~]$ cd git-2.9.5
      [centos@ip-XXXX-XXX-XXX-XXX ~]$ sudo make configure
      // omit
      [centos@ip-XXXX-XXX-XXX-XXX ~]$ sudo ./configure --prefix=/usr
      // omit
      [centos@ip-XXXX-XXX-XXX-XXX ~]$ sudo make install
      // omit
      [centos@ip-XXXX-XXX-XXX-XXX ~]$ git --version
      git version 2.9.5

|br|

以降は、EC2環境等でゲストOSとしてCentOS7をLaunchさせて、DockerやGitをインストール済みしたマシン上での操作を前提に話を進めていきます。

GitにあるソースコードプロジェクトにあるDockerfileを元に、Dockerイメージを作成します。以下では、ソースコードをGitHubからクローンし、
Backendアプリケーションのプロジェクト直下にあるDockerfileを指定して、docker buildコマンドでDockerイメージを作成しています。
GitHubからチェックアウトするソースコードのプロジェクトやdocker buildコマンドで指定する-tオプションのレポジトリ名は適宜、自身の環境に合わせて変更してください。
下記ではDocker Hubを前提にしたコマンド実行例です。

|br|

.. sourcecode:: bash

   [centos@ip-XXX-XXX-XXX-XXX ~]$ sudo git clone https://github.com/debugroom/mynavi-sample-aws-ecs.git
   [centos@ip-XXX-XXX-XXX-XXX ~]$ cd mynavi-sample-aws-ecs
   [centos@ip-XXX-XXX-XXX-XXX mynavi-sample-aws-ecs]$ docker build backend/ -t debugroom/mynavi-sample-ecs-backend:latest

|br|

Dockerイメージが作成されたら、Docker HubもしくはAmazon ECRへプッシュします。下記は、DockerHubへコンテナイメージをプッシュする例です。

.. sourcecode:: bash

   [centos@ip-XXX-XXX-XXX-XXX ~]$ docker login
   // omit  - enter USERID and password
   [centos@ip-XXX-XXX-XXX-XXX ~]$ docker push debugroom/mynavi-sample-ecs-backend:latest

|br|

これでコンテナイメージをレジストリにプッシュできました。次回は、実際にECSコンテナを実行するためのECSクラスタを構築します。

|br|


著者紹介
------------------------------------------------------------------

川畑 光平(KAWABATA Kohei)

.. figure:: img/aws-lambda-and-api-gateway/pic_image01.jpg
   :scale: 100%

某システムインテグレータにて、金融機関システム業務アプリケーション開発・システム基盤担当を経て、現在はソフトウェア開発自動化関連の研究開発・推進に従事。

Red Hat Certified Engineer、Pivotal Certified Spring Professional、AWS Certified Solutions Architect Professional等の資格を持ち、アプリケーション基盤・クラウドなど様々な開発プロジェクト支援にも携わる。

本連載記事の内容に対するご意見・ご質問は `Facebook <https://www.facebook.com/kohei.kawabata.5>`_ まで。
