.. include:: ../module.txt

.. _section-automation-infra-devops-cloudformation-20-label:

【第40回】AWS CloudFormationを用いた基盤自動化(20)継続的インテグレーション環境の構築(1)
-------------------------------------------------------------------------------------------------------------------------------------

|br|

本連載では、AWSリソース基盤構築の自動化を実践しています。今回以降は `第2回 <https://news.mynavi.jp/itsearch/article/devsoft/4463>`_ から複数回解説してきた継続的インテグレーション環境をCloudFormationを使って自動構築していきます。

|br|

.. figure:: img/automation_infra_devops_cloudformation/cloudformation-ci-environment.png

|br|

上記の図はリンク先のCI環境構成のイメージを基盤自動化するにあたり新たに書き起こしたものです。ただし、手動で構築してきたときに比べて幾つか差分があります。静的解析を行うSonarQubeServerの構築では、データベースを構築した後のユーザ作成といった初期化処理を
わざわざRDSに接続して手動で行っていましたが、これをCloudFormationのCustom Resource(AWS Lambda-backedカスタムリソース)を使って、AWS Lambdaから実行するものとします。
また、RDSへのパスワード等はCloudFormationのテンプレートコードにハードコーティングするのは望ましくないので、SystemsManager Parameter Store経由で取得するようにします。

|br|

本連載で実際に作成するアプリケーションでは `GitHub <https://github.com/debugroom/mynavi-sample-sonarqube-aws>`_ 上にコミットしています。
以降に記載するソースコードでは、import文など本質的でない記述を省略している部分があるので、実行コードを作成する際は、必要に応じて適宜GitHubにあるソースコードも参照してください。


|br|

.. _section-cloudformation-sonarqube-server-base-label:

SonarQubeServerのベース環境構築テンプレート
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

まず、SonarQubeServerが実行されるベースとなるVPCやセキュリティグループ、ALB、ECSクラスタといったリソースを構築します。
作成したCloudFormationテンプレートは `こちら <https://github.com/debugroom/mynavi-sample-sonarqube-aws/tree/master/base>`_ にあります。
テンプレートは、`第26回 VPC/Subnet/RouteTable/InternetGateway構築自動化テンプレート <https://news.mynavi.jp/itsearch/article/devsoft/4810>`_ 、 `第27回 SecurityGroup構築自動化テンプレート <https://news.mynavi.jp/itsearch/article/devsoft/4829>`_ 、
`第28回 ApplicationLoadBalancer構築自動化テンプレート <https://news.mynavi.jp/itsearch/article/devsoft/4842>`_ 、`第36回 ECSクラスタの構築 <https://news.mynavi.jp/itsearch/article/devsoft/4995>`_ で解説したものとほぼ同等です。
必要に応じて参照してください。

|br|

差分がある箇所として、ECSクラスタの接続がSystems Manager Session Managerを使って可能なように、割り当てるロールにAmazonSSMManagedInstanceCoreポリシーを付与して、
また、ECSのクラスタの起動スクリプトでSSMエージェントをインストールしています。

|br|

.. sourcecode:: none

   AWSTemplateFormatVersion: '2010-09-09'

   # omit

   Resources:
     ECSRole:
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
           - arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

   # omit

   PublicECSLaunchConfiguration:
     Type: AWS::AutoScaling::LaunchConfiguration
     Properties:
       ImageId: !Ref ECSAMI
       InstanceType: !FindInMap [PublicClusterDefinitionMap, Instance, InstanceType]
       IamInstanceProfile: !Ref IamInstanceProfile
       SecurityGroups:
         - Fn::ImportValue: !Sub ${VPCName}-SecurityGroupPublicEcsCluster
       AssociatePublicIpAddress: true
       UserData:
         Fn::Base64: !Sub |
           #!/bin/bash -xe
           echo ECS_CLUSTER=${PublicECSCluster} >> /etc/ecs/ecs.config
           yum install -y aws-cfn-bootstrap jq
           # Get the current region from the instance metadata
           region=$(curl -s http://169.254.169.254/latest/dynamic/instance-identity/document | jq -r .region)
           # Install the SSM agent RPM
           yum install -y https://amazon-ssm-${AWS::Region}.s3.amazonaws.com/latest/linux_amd64/amazon-ssm-agent.rpm
           /opt/aws/bin/cfn-signal -e $? --stack ${AWS::StackName} --resource PublicECSAutoScalingGroup --region ${AWS::Region}

    # omit

|br|

.. note:: Systems Manager Session ManagerはEC2インスタンスやオンプレミスにあるサーバと安全に接続するためのSystems Managerの機能です。通常EC2やサーバにはSSHなどを使用して接続するのが一般的ですが、
   Session Managerは以下の図のように、接続対象となるサーバにエージェントとなるソフトウェア(SSM Agent)をインストールし、Session Managerへ常にコンソールプロセス情報のポーリングを行うことで、
   インバウンド接続設定を行わずとも、対象のサーバにアクセスすることができようになります。セキュリティグループのデフォルトの設定通りアウトバウンド接続が常に可能な状態になっていれば、プライベートサブネットに配置されたEC2インスタンスでも踏み台なしに、Session Manager経由で直接接続することが可能です。

   |br|

   .. figure:: img/automation_infra_devops_cloudformation/session-manager.png

   |br|

   なお、オンプレミスのサーバでもSSM Agentをインストールすれば、Session Managerからアクセスが可能です。Session Managerには、さらにアクセスログをCloudWatchやS3へ保存する機能やSNSなどの通知サービスと連携させて禁止された操作を検出・通知することもできます。
   AWSでは、アクセスキーやシークレットキーの漏洩リスクがある通常のSSH接続ではなく、Session Managerによるアクセスを推奨しています。Session ManagerはAWSマネジメントコンソール上経由でブラウザを使ってシェルコンソールを開くか、AWS CLI(コマンドライン)のプラグインを使って、
   通常のSSHアクセスと同じようにローカルのターミナルコンソール上から実行することが可能です。

|br|


.. _section-cloudformation-sonarqube-server-rds-label:

SonarQubeServer向けRDS環境構築と事前準備
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

続いて、SonarQubeServer向けRDS環境構築や、次回解説するLambdaファンクション実装の事前準備として以下を行っておく必要があります。

* CloudFormationを使ってLambdaをデプロイするために、事前にS3バケットを作成するテンプレートを実装・実行。
* Lambdaファンクション内でCloudFormationカスタムリソースにおけるResponseURLへリクエスト送信するためにNAT Gatewayを作成するテンプレートを実装・実行
* Systems Manager Parameter StoreにRDSマスターユーザおよびSonarユーザのパスワードを作成(AWSコンソールから手動で実施)
* SonarQuberServer用のRDS(PostgreSQL)テンプレートを実装・実行し、エンドポイント、データベース名、マスターユーザ名をスタックでエクスポート出力

実際に作成したCloudFormationテンプレートは `ここ <https://github.com/debugroom/mynavi-sample-sonarqube-aws/tree/master/rds>`_ にあります。
S3バケットを作成するテンプレートの解説は、`第32回 SQS/S3の構築自動化テンプレート <https://news.mynavi.jp/itsearch/article/devsoft/4926>`_ 、
NAT Gatewayを作成するテンプレートの解説は `第27回 NATGateway構築自動化テンプレート <https://news.mynavi.jp/itsearch/article/devsoft/4829>`_　、
Systems Manager Parameter Storeを使ったパスワード設定やRDSを作成するCloudFormationテンプレートの解説は、`第29回 <https://news.mynavi.jp/itsearch/article/devsoft/4869>`_ を適宜参照してください。

|br|

今回はSonarQubeServerのベースとなる環境のCloudFormationテンプレートの実装とRDS環境を構築しました。次回は、CloudFormationカスタムリソース(AWS Lambda-backed カスタムリソース)を使ってRDSの初期化処理を行うLambdaファンクションを作成します。

|br|

著者紹介
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

川畑 光平(KAWABATA Kohei) - NTTデータ 課長代理

.. figure:: img/automation_infra_devops_overview/pic_image01.jpg


金融機関システム業務アプリケーション開発・システム基盤担当を経て、現在はソフトウェア開発自動化関連の研究開発・推進に従事。

Red Hat Certified Engineer、Pivotal Certified Spring Professional、AWS Certified Solutions Architect Professional等の資格を持ち、アプリケーション基盤・クラウドなど様々な開発プロジェクト支援にも携わる。

`2019 APN AWS Top Engineers & Ambassadors <https://aws.amazon.com/jp/blogs/psa/japan-apn-ambassador-2019/>`_ 選出。
