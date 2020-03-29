.. include:: ../module.txt

.. _section-automation-infra-devops-cloudformation-16-label:

【第36回】AWS CloudFormationを用いた基盤自動化(16)ECSクラスタの構築
-------------------------------------------------------------------------------------------------------------------------------------

|br|

本連載では、以下のイメージの構成にあるAWSリソース基盤自動化環境の構築を実践しています。

|br|

.. figure:: img/automation_infra_devops_cloudformation/cloudformation-scope.png

|br|

前回は、ALB、ElastiCache、S3といったリソースアクセス、SQSへのキュー送信を行うFrontend Webアプリケーションにおいて、Spring Cloud AWSを用いて取得したスタック情報を使ったアプリケーションの設定実装を紹介しました。
今回からは実装したアプリケーションを実行するためのECSクラスタ、タスク定義、サービス実行と順次CloudFormationテンプレートを作成し実行していきます。
実際のソースコードは `GitHub <https://github.com/debugroom/mynavi-sample-cloudformation>`_ 上にコミットしています。
ソースコード中で本質的でない記述を一部省略しているので、実行コードを作成する場合は、必要に応じて適宜GitHub上のソースコードも参照してください。

|br|

.. _section-cloudformation-ecs-task-preparation-label:

事前準備
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

事前準備として、前回実装したアプリケーションのコンテナイメージを作成し、DockerHubへプッシュしておきましょう。
作業の方法は `クラウドネイティブ基本第7回 <https://news.mynavi.jp/itsearch/article/devsoft/4390>`_ と同様です。Backend Serviceアプリケーションおよび、Frontend Webアプリケーションのコンテナイメージを作成するDockerfileのサンプルは以下の通りです。

|br|

.. sourcecode:: none

   # Dockerfile for sample service using embedded tomcat server

   FROM centos:centos7
   MAINTAINER debugroom

   RUN yum install -y \
       java-1.8.0-openjdk \
       java-1.8.0-openjdk-devel \
       wget tar iproute git

   RUN rm -f /etc/rpm/macros.image-language-conf && \
       sed -i '/^override_install_langs=/d' /etc/yum.conf && \
       yum -y update glibc-common && \
       yum clean all

   ENV LANG="ja_JP.UTF-8" \
       LANGUAGE="ja_JP:ja" \
       LC_ALL="ja_JP.UTF-8"

   RUN wget http://repos.fedorapeople.org/repos/dchen/apache-maven/epel-apache-maven.repo -O /etc/yum.repos.d/epel-apache-maven.repo
   RUN sed -i s/\$releasever/6/g /etc/yum.repos.d/epel-apache-maven.repo
   RUN yum install -y apache-maven
   ENV JAVA_HOME /etc/alternatives/jre
   RUN git clone https://github.com/debugroom/mynavi-sample-aws-cloudformation.git /usr/local/mynavi-sample-aws-cloudformation
   RUN mvn install -f /usr/local/mynavi-sample-aws-cloudformation/common/pom.xml
   RUN mvn package -f /usr/local/mynavi-sample-aws-cloudformation/backend-service/pom.xml
   RUN cp /etc/localtime /etc/localtime.org
   RUN ln -sf  /usr/share/zoneinfo/Asia/Tokyo /etc/localtime

   EXPOSE 8080

   CMD java -jar -Dspring.profiles.active=$ENV_TYPE /usr/local/mynavi-sample-aws-cloudformation/backend-service/target/mynavi-sample-cloudformation-backend-0.0.1-SNAPSHOT.jar

|br|

.. sourcecode:: none

   # Dockerfile for sample service using embedded tomcat server

   FROM centos:centos7
   MAINTAINER debugroom

   RUN yum install -y \
       java-1.8.0-openjdk \
       java-1.8.0-openjdk-devel \
       wget tar iproute git

   RUN rm -f /etc/rpm/macros.image-language-conf && \
       sed -i '/^override_install_langs=/d' /etc/yum.conf && \
       yum -y update glibc-common && \
       yum clean all

   ENV LANG="ja_JP.UTF-8" \
       LANGUAGE="ja_JP:ja" \
       LC_ALL="ja_JP.UTF-8"

   RUN wget http://repos.fedorapeople.org/repos/dchen/apache-maven/epel-apache-maven.repo -O /etc/yum.repos.d/epel-apache-maven.repo
   RUN sed -i s/\$releasever/6/g /etc/yum.repos.d/epel-apache-maven.repo
   RUN yum install -y apache-maven
   ENV JAVA_HOME /etc/alternatives/jre
   RUN git clone https://github.com/debugroom/mynavi-sample-aws-cloudformation.git /usr/local/mynavi-sample-aws-cloudformation
   RUN mvn install -f /usr/local/mynavi-sample-aws-cloudformation/common/pom.xml
   RUN mvn package -f /usr/local/mynavi-sample-aws-cloudformation/frontend-webapp/pom.xml
   RUN cp /etc/localtime /etc/localtime.org
   RUN ln -sf  /usr/share/zoneinfo/Asia/Tokyo /etc/localtime

   EXPOSE 8080

   CMD java -jar -Dspring.profiles.active=$ENV_TYPE /usr/local/mynavi-sample-aws-cloudformation/frontend-webapp/target/mynavi-sample-cloudformation-frontend-0.0.1-SNAPSHOT.jar

|br|

.. _section-cloudformation-ecs-cluster-sample-label:

ECSクラスタスタック構築テンプレート
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

ECSクラスタは `クラウドネイティブ基本第8回 <https://news.mynavi.jp/itsearch/article/devsoft/4405>`_ で実施した要領と同等のものを構築します。
ECSクラスタをCloudFormationで構築する場合、リソースタイプが、 `AWS::ECS::Cluster <https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-ecs-cluster.html>`_
、クラスタの実行に必要なIAMロール `AWS::IAM::Role <https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-iam-role.html>`_ および、
そのロールを設定したインスタンスプロファイル `AWS::IAM::InstanceProfile <https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-iam-instanceprofile.html>`_ 、
オートスケールルールを設定した `AWS::AutoScaling::AutoScalingGroup <https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-properties-as-group.html>`_ に加えて、
起動設定を定義した `AWS::AutoScaling::LaunchConfiguration <https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-properties-as-launchconfig.html>`_ が必要です。
プロパティとして設定可能な属性は、上記リンク先の通りですが、加えて、ECSを商用環境、ステージング環境、開発環境という3つのパターンに分けて作成するようにします。

テンプレートのサンプルは以下の通りです。

|br|

.. sourcecode:: xml

   AWSTemplateFormatVersion: '2010-09-09'

   // omit

   Parameters:
     ECSAMI:
       Description: AMI ID
       Type: AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>                                       #(A)
       Default: /aws/service/ecs/optimized-ami/amazon-linux-2/recommended/image_id
     EnvType:
       Description: Which environments to deploy your service.
       Type: String
       AllowedValues:
         - Dev
         - Staging
         - Production
       Default: Dev

   Mappings:
     FrontendClusterDefinitionMap:                                                                 #(B)
       Production:
         "InstanceType" : "r4.large"
         "DesiredCapacity" : 1
         "EC2InstanceMaxSizeOfECS": 3
         "KeyPairName" : "test"
    // omit

   Resources:
     ECSRole:                                                                                      #(C)
       Type: AWS::IAM::Role
       Properties:
         Path: /
         AssumeRolePolicyDocument:
           Statement:
             - Action: sts:AssumeRole
               Effect: Allow
               Principal:
                 Service: ec2.amazonaws.com
         ManagedPolicyArns:
           - arn:aws:iam::aws:policy/service-role/AmazonEC2ContainerServiceforEC2Role

     IamInstanceProfile:                                                                           #(D)
       Type: AWS::IAM::InstanceProfile
       Properties:
         Path: /
         Roles:
           - !Ref ECSRole

     FrontendECSCluster:                                                                           #(E)
       Type: AWS::ECS::Cluster
       Properties:
         ClusterName: !Sub sample-frontend-cluster-${EnvType}
         Tags:
           - Key: Name
             Value: !Sub FrontendECSCluster-${EnvType}

     // omit

     FrontendECSAutoScalingGroup:                                                                  #(F)
       Type: AWS::AutoScaling::AutoScalingGroup
       Properties:
         VPCZoneIdentifier:
           - Fn::ImportValue: !Sub ${VPCName}-PublicSubnet1
           - Fn::ImportValue: !Sub ${VPCName}-PublicSubnet2
         LaunchConfigurationName: !Ref FrontendECSLaunchConfiguration
         MinSize: '0'
         MaxSize: !FindInMap [FrontendClusterDefinitionMap, !Ref EnvType, EC2InstanceMaxSizeOfECS] #(G)
         DesiredCapacity: !FindInMap [FrontendClusterDefinitionMap, !Ref EnvType, DesiredCapacity]
         Tags:
           - Key: Name
             Value: !Sub FrontendECSCluster-${EnvType}
             PropagateAtLaunch: true
       CreationPolicy:
         ResourceSignal:
           Timeout: PT5M
       UpdatePolicy:
         AutoScalingReplacingUpdate:
           WillReplace: true

     FrontendECSLaunchConfiguration:                                                                #(H)
       Type: AWS::AutoScaling::LaunchConfiguration
       Properties:
         ImageId: !Ref ECSAMI
         InstanceType: !FindInMap [FrontendClusterDefinitionMap, !Ref EnvType, InstanceType]
         IamInstanceProfile: !Ref IamInstanceProfile
         KeyName: !FindInMap [FrontendClusterDefinitionMap, !Ref EnvType, KeyPairName]
         SecurityGroups:
           - Fn::ImportValue: !Sub ${VPCName}-SecurityGroupFrontendEcsCluster
         AssociatePublicIpAddress: true
         UserData:                                                                                  #(I)
           Fn::Base64: !Sub |
             #!/bin/bash -xe
             echo ECS_CLUSTER=${FrontendECSCluster} >> /etc/ecs/ecs.config
             yum install -y aws-cfn-bootstrap
             /opt/aws/bin/cfn-signal -e $? --stack ${AWS::StackName} --resource FrontendECSAutoScalingGroup --region ${AWS::Region}

   // omit

|br|

ECSクラスタのテンプレートの記述の基本となるポイントは(A)〜(G)の通りです。

|br|

.. list-table:: ECSクラスタのCloudFormationテンプレート記述のポイント
   :widths: 1, 9

   * - 記述
     - 説明

   * - (A)
     - 起動するECSの `Amazon Linux AMIをSystems Manager Parameter Store経由で取得 <https://aws.amazon.com/jp/blogs/news/query-for-the-latest-amazon-linux-ami-ids-using-aws-systems-manager-parameter-store/>`_ し、パラメータとして定義します。

   * - (B)
     - パラメータEnvTypeに応じて、適用するパラメータ値を変更するよう、Mappings要素を定義します。

   * - (C)
     - ECSクラスタの実行に必要なAmazonEC2ContainerServiceforEC2Roleポリシーが付与されたIAMロールを定義します。詳細は `AWS::IAM::Role <https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-iam-role.html>`_ を参照してください。

   * - (D)
     - (C)のIAMロールが付与されたインスタンスプロファイルを定義します。詳細は　 `AWS::IAM::InstanceProfile <https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-iam-instanceprofile.html>`_ を参照してください。

   * - (E)
     - ECSクラスタを定義します。詳細は  `AWS::ECS::Cluster <https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-ecs-cluster.html>`_ を参照してください。

   * - (F)
     - クラスタのオートスケーリンググループを定義します。詳細は  `AWS::AutoScaling::AutoScalingGroup <https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-properties-as-group.html>`_ を参照してください。

   * - (G)
     - FindInMap関数を用いて、作成する環境に応じてパラメータを切り替えます。

   * - (H)
     - ECSクラスタの起動オプション設定を定義します。詳細は `AWS::AutoScaling::LaunchConfiguration <https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-properties-as-launchconfig.html>`_ を参照してください。

   * - (I)
     - クラスタ起動に実行する起動スクリプトを定義します。ecs.configへのクラスタ定義の追加や、aws-cfn-bootstrapのインストール、cfn-signalの実行がECSクラスタの起動に必要になります。

|br|

.. warning:: プライベートサブネットにECSクラスタを構築する場合は、起動スクリプト内のyumコマンドを実行するため、NATGatewayを設置して、ルートテーブルをプライベートサブネットに関連づけておく必要があります。上記でNATGatewayの設定はテンプレート記述に含めていませんが、
   `第27回 <https://news.mynavi.jp/itsearch/article/devsoft/4829>`_ を参考にNATGatewayを事前に設定しておいてください。

|br|

作成したテンプレートに対して、ヘルパースクリプトを以下のように、スタック名とテンプレートパスを変更して実行します。

|br|

.. sourcecode:: bash

   #!/usr/bin/env bash

   stack_name="mynavi-sample-ecs-cluster"
   template_path="sample-ecs-cluster-cfn.yml"

   parameters="EnvType=Dev"

   aws cloudformation deploy --stack-name ${stack_name} --template-file ${template_path} --parameter-overrides ${parameters} --capabilities CAPABILITY_IAM

|br|

実行が正常に終了すると、ECSクラスタが作成されます。

|br|

.. figure:: img/automation_infra_devops_cloudformation/management_console_cloudformation_stack_ecs_cluster.png

|br|

今回はECSクラスタを構築するCloudFormationテンプレートを実装しました。次回は、ECSタスク定義を行うCloudFormationテンプレートを作成する手順を紹介します。

|br|

著者紹介
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

川畑 光平(KAWABATA Kohei) - NTTデータ 課長代理

.. figure:: img/automation_infra_devops_overview/pic_image01.jpg


金融機関システム業務アプリケーション開発・システム基盤担当を経て、現在はソフトウェア開発自動化関連の研究開発・推進に従事。

Red Hat Certified Engineer、Pivotal Certified Spring Professional、AWS Certified Solutions Architect Professional等の資格を持ち、アプリケーション基盤・クラウドなど様々な開発プロジェクト支援にも携わる。

`2019 APN AWS Top Engineers & Ambassadors <https://aws.amazon.com/jp/blogs/psa/japan-apn-ambassador-2019/>`_ 選出。
