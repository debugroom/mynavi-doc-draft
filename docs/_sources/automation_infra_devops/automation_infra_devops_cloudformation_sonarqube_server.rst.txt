.. include:: ../module.txt

.. _section-automation-infra-devops-cloudformation-23-label:

【第43回】AWS CloudFormationを用いた基盤自動化(23)継続的インテグレーション環境の構築(4)
-------------------------------------------------------------------------------------------------------------------------------------

|br|

本連載では、AWSリソース基盤構築の自動化を実践しています。これまで解説してきた継続的インテグレーション環境をCloudFormationを使って自動構築しています。

|br|

.. figure:: img/automation_infra_devops_cloudformation/cloudformation-ci-environment.png

|br|

前回は、LambdaをデプロイするCloudFormationやLambdaファンクションを実行するカスタムリソーステンプレートを作成・実行しました。
続く今回はSonarQubeServerの起動スクリプトやDockerfile、ECSタスク、サービステンプレートを作成し、サーバを起動してみます。

本連載で実際に作成するアプリケーションでは `GitHub <https://github.com/debugroom/mynavi-sample-sonarqube-aws>`_ 上にコミットしています。
以降に記載するソースコードでは、import文など本質的でない記述を省略している部分があるので、実行コードを作成する際は、必要に応じて適宜GitHubにあるソースコードも参照してください。

.. _section-cloudformation-sonarqube-server-docker-and-script-label:

SonarQubeServerを起動するスクリプトとDockerfile
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

`第2回 <https://news.mynavi.jp/itsearch/article/devsoft/4463>`_ で既に実装済みのDockerfileや起動スクリプトはありますが、
RDSへ接続するためのエンドポイントやパスワードはCloudFormationのスタックやSystems Manager Parameter Storeから取得する方法に変えています。
そのため、以下のように起動スクリプトを変更します。

|br|

.. sourcecode:: bash

   #!/usr/bin/env bash

   # omit
   # エンドポイントとユーザ名をECSの環境変数として受け取り、SONARQUBE_JDBC_URLとして定義
   export SONARQUBE_JDBC_URL="jdbc:postgresql://$SONARQUBE_RDS_ENDPOINT/$SONARQUBE_RDS_USERNAME"

   # omit

   exec java -jar lib/sonar-application-$SONAR_VERSION.jar \
     -Dsonar.log.console=true \
     -Dsonar.jdbc.username="$SONARQUBE_RDS_USERNAME" \
     -Dsonar.jdbc.password="$SONARQUBE_RDS_PASSWORD" \
     -Dsonar.jdbc.url="$SONARQUBE_JDBC_URL" \
     -Dsonar.web.javaAdditionalOpts="$SONARQUBE_WEB_JVM_OPTS -Djava.security.egd=file:/dev/./urandom" \
       "${sq_opts[@]}" \
       "$@"

|br|

また、Dockerfileではエンドポイントやパスワード等、ECSタスク定義から一部は環境変数を取得するため、従来適していた変数の設定をやめてユーザ名とホームディレクトリ以外を削除します。

|br|

.. sourcecode:: none

   # omit

   ENV SONAR_VERSION=7.7 \
       SONARQUBE_HOME=/usr/local/sonarqube \
       SONARQUBE_RDS_USERNAME=sonar

   # omit

|br|

第2回同様、Docker buildコマンドを実行して新しくコンテナイメージをビルドし、DockerHubへプッシュしておきます。

|br|

.. _section-cloudformation-sonarqube-ecs-label:

SobarQubeECSタスク定義とターゲットグループ、ECSサービスのテンプレート
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

次に、ECSタスク定義とターゲットグループのテンプレートを作成します。実装したものは `こちら <https://github.com/debugroom/mynavi-sample-sonarqube-aws/tree/master/sonarqube-server>`_ ですが、
このテンプレートは、`第37回 ECSタスク定義の構築自動化テンプレート <https://news.mynavi.jp/itsearch/article/devsoft/5002>`_ 、 `第38回 ECSタスクロール定義の構築テンプレート <https://news.mynavi.jp/itsearch/article/devsoft/5007>`_ 、 `第39回 ECSサービス構築テンプレート <https://news.mynavi.jp/itsearch/article/devsoft/5007>`_ で解説したものとほぼ同等です。
ただし、ECSタスク定義やECSサービスは以下の通り、コンテナイメージやポート、環境変数やシークレットパラメータの設定を変更して実装しています。


|br|

.. sourcecode:: none

   #omit

   Mappings:
     SonarQubeTaskDefinitionMap:
       Task:
         "Memory" : 2048
         "Cpu" : 1024
         "ContainerName" : "debugroom-mynavi-sonarqube-cfn-server"
         "ContainerImage" : "debugroom/sonarqube-server-cfn:latest"
         "ContainerPort" : 9000
         "HostPort" : 0
         "ContainerMemory" : 2048
         "Profile" : "dev"

   # omit

   SonarQubeECSTaskDefinition:
     Type: AWS::ECS::TaskDefinition
     Properties:
       Family: sonarqube-server
       RequiresCompatibilities:
         - EC2
       Memory: !FindInMap [SonarQubeTaskDefinitionMap, Task, Memory]
       Cpu: !FindInMap [SonarQubeTaskDefinitionMap, Task, Cpu]
       NetworkMode: bridge
       ExecutionRoleArn: !GetAtt ECSTaskExecutionRole.Arn
       TaskRoleArn: !GetAtt SonarQubeECSTaskRole.Arn
       ContainerDefinitions:
         - Name: !FindInMap [SonarQubeTaskDefinitionMap, Task, ContainerName]
           Image: !FindInMap [SonarQubeTaskDefinitionMap, Task, ContainerImage]
           PortMappings:
             - ContainerPort: !FindInMap [SonarQubeTaskDefinitionMap, Task, ContainerPort]
               HostPort: !FindInMap [SonarQubeTaskDefinitionMap, Task, HostPort]
               Memory: !FindInMap [SonarQubeTaskDefinitionMap, Task, ContainerMemory]
           Environment:
             - Name: SONARQUBE_RDS_ENDPOINT
               Value:
                 Fn::ImportValue: !Sub ${VPCName}-RDS-Endpoint
           Secrets:
             - Name: SONARQUBE_RDS_PASSWORD
               ValueFrom: !Sub "arn:aws:ssm:${AWS::Region}:${AWS::AccountId}:parameter/mynavi-sonarqube-rds-sonar-password"

    # omit

|br|

.. sourcecode:: none

   # omit

   Resources:
     SonarQubeService:
       Type: AWS::ECS::Service
       Properties:
         Cluster:
           Fn::ImportValue: !Sub ${VPCName}-PublicEcsCluster
         DesiredCount: 1
         HealthCheckGracePeriodSeconds: 60
         TaskDefinition:
           Fn::ImportValue: !Sub ${VPCName}-SonarQubeEcsTaskDefinition
         LaunchType: EC2
         LoadBalancers:
           - ContainerName: "debugroom-mynavi-sonarqube-cfn-server"
             ContainerPort: 9000
             TargetGroupArn:
               Fn::ImportValue: !Sub ${VPCName}-SonarQube-TargetGroup

|br|

このテンプレートを実行すると、SonarQubeServerが起動します。

|br|

.. note:: 実行時にエラーが発生した場合は、Systems Manager Session ManagerでECSクラスタへログインし、Docker logsコマンドなどで起動エラーになる原因をトラブルシューティングすることが可能です。

|br|

今回はSonarQubeServerを起動するCloudFormationテンプレートを実装し、起動しました。次回以降はAWS CodeBuildを使ったアプリケーションのビルドをCloudFormationを使って自動化してみます。

|br|

著者紹介
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

川畑 光平(KAWABATA Kohei) - NTTデータ 課長代理

.. figure:: img/automation_infra_devops_overview/pic_image01.jpg


金融機関システム業務アプリケーション開発・システム基盤担当を経て、現在はソフトウェア開発自動化関連の研究開発・推進に従事。

Red Hat Certified Engineer、Pivotal Certified Spring Professional、AWS Certified Solutions Architect Professional等の資格を持ち、アプリケーション基盤・クラウドなど様々な開発プロジェクト支援にも携わる。

`2019 APN AWS Top Engineers & Ambassadors <https://aws.amazon.com/jp/blogs/psa/japan-apn-ambassador-2019/>`_ 選出。
