.. include:: ../module.txt

.. _section-automation-infra-devops-cloudformation-9-label:

基盤・デプロイ自動化実践
==================================================================

マイクロサービスアーキテクチャの基盤・デプロイ自動化
-------------------------------------------------------------------------------------------------------------------------------------

|br|

本連載では、以下のイメージの構成にあるAWSリソース基盤自動化環境の構築を実践しています。

|br|

.. figure:: img/automation_infra_devops_cloudformation/cloudformation-scope.png

|br|

前回は、Frontend、Backendサブネットに配置するアプリケーションロードバランサー(ALB)を構築するCloudFormationテンレプートを実装しました。
続く今回はバックエンドサブネットからのみアクセス可能なRDS(RelationalDatabaseService)を構築するCLoudFormationテンプレートを作成します。
実際のソースコードは `GitHub <https://github.com/debugroom/mynavi-sample-cloudformation>`_ 上にコミットしています。
ソースコード中で本質的でない記述を一部省略しているので、実行コードを作成する場合は、必要に応じて適宜GitHub上のソースコードも参照してください。

|br|

.. _section-cloudformation-rds-sample-label:

RDSスタック構築テンプレート
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

RDSは `クラウドネイティブ基本第11回 <https://news.mynavi.jp/itsearch/article/devsoft/4422>`_ で実施した要領と同等のものを構築します。
CloudFormationで構築する場合、リソースタイプが、 `AWS::RDS::DBInstance <https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-properties-rds-database-instance.html>`_
となるDBインスタンスの定義と、DBを配置するサブネットをグループ化したサブネットグループ定義 `AWS::RDS::DBSubnetGroup <https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-rds-dbsubnet-group.html>`_
また、監視のためのモニター権限をIAMロール `AWS::IAM::Role <https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-elasticloadbalancingv2-listener.html>`_ として作成しておかなければなりません。
プロパティとして設定可能な属性は、各リンク先の通りですが、加えて、Conditions要素を使って、RDSを商用環境、ステージング環境、開発環境という3つのパターンに分けて作成するようにします。また、
RDSにはユーザとパスワードの設定が必要になりますが、秘匿性の高いパスワードについては、AWS SystemsManager ParameterStoreから取得可能なDynamicReferences機能を使って、テンプレートを構成します。テンプレートのサンプルは以下の通りです。


|br|

.. sourcecode:: none

   AWSTemplateFormatVersion: '2010-09-09'

   // omit

   Parameters:

     // omit

     RdsUser:                                                                                      #(A)
       Description: Database Master User Name
       Type: String
       Default: postgresql
     EnvType:                                                                                      #(B)
       Description: Which environments to deploy your service.
       Type: String
       AllowedValues: ["Dev", "Staging", "Production"]
       Default: Dev

   Conditions:                                                                                     #(C)
     ProductionResources: {"Fn::Equals" : [{"Ref":"EnvType"}, "Production"]}
     StagingResources: !Equals [ !Ref EnvType, "Staging"]
     DevResources: {"Fn::Equals" : [{"Ref":"EnvType"}, "Dev"]}

   Resources:
     RDSProductionInstance:                                                                        #(D)
       Type: AWS::RDS::DBInstance
       DeletionPolicy: Snapshot
       Condition: "ProductionResources"                                                            #(E)
       Properties:
         DBInstanceIdentifier: mynavi-sample-cloudformation-production-postgresql
         DBName: PostgreSQL
         Engine: postgres
         MultiAZ: false
         MasterUsername: !Ref RdsUser
         MasterUserPassword: '{{resolve:ssm-secure:mynavi-sample-cloudformation-rds-password:1}}'  #(F)
         DBInstanceClass: db.t2.micro
         AllocatedStorage: '20'
         DBSubnetGroupName: !Ref DBSubnetGroup
         MonitoringInterval: 10
         MonitoringRoleArn: !GetAtt DBMonitorRole.Arn
         VPCSecurityGroups:
           - Fn::ImportValue: !Sub ${VPCName}-SecurityGroupRdsPostgres

     RDSStagingInstance:                                                                           #(G)
       Type: AWS::RDS::DBInstance
       DeletionPolicy: Snapshot
       Condition: "StagingResources"
       Properties:
         DBInstanceIdentifier: mynavi-sample-cloudformation-staging-postgresql
         DBName: PostgreSQL
         Engine: postgres
         MultiAZ: false
         MasterUsername: !Ref RdsUser
         MasterUserPassword: '{{resolve:ssm-secure:mynavi-sample-cloudformation-rds-password:1}}'
         DBInstanceClass: db.t2.micro
         AllocatedStorage: '20'
         DBSubnetGroupName: !Ref DBSubnetGroup
         MonitoringInterval: 10
         MonitoringRoleArn: !GetAtt DBMonitorRole.Arn
         VPCSecurityGroups:
           - Fn::ImportValue: !Sub ${VPCName}-SecurityGroupRdsPostgres

     RDSDevInstance:                                                                               #(H)
       Type: AWS::RDS::DBInstance
       DeletionPolicy: Snapshot
       Condition: "DevResources"
       Properties:
         DBInstanceIdentifier: mynavi-sample-cloudformation-dev-postgresql
         DBName: PostgreSQL
         Engine: postgres
         MultiAZ: false
         MasterUsername: !Ref RdsUser
         MasterUserPassword: '{{resolve:ssm-secure:mynavi-sample-cloudformation-rds-password:1}}'
         DBInstanceClass: db.t2.micro
         AllocatedStorage: '20'
         DBSubnetGroupName: !Ref DBSubnetGroup
         MonitoringInterval: 10
         MonitoringRoleArn: !GetAtt DBMonitorRole.Arn
         VPCSecurityGroups:
           - Fn::ImportValue: !Sub ${VPCName}-SecurityGroupRdsPostgres

     DBSubnetGroup:                                                                                #(I)
       Type: AWS::RDS::DBSubnetGroup
       Properties:
         DBSubnetGroupDescription: DB Subnet Group for Private Subnet
         SubnetIds:
           - Fn::ImportValue: !Sub ${VPCName}-PrivateSubnet1
           - Fn::ImportValue: !Sub ${VPCName}-PrivateSubnet2

     DBMonitorRole:                                                                                #(J)
       Type: AWS::IAM::Role
       Properties:
         Path: "/"
         ManagedPolicyArns:
           - arn:aws:iam::aws:policy/service-role/AmazonRDSEnhancedMonitoringRole
         AssumeRolePolicyDocument:
           Version: 2012-10-17
           Statement:
             - Effect: Allow
               Principal:
                 Service:
                   - monitoring.rds.amazonaws.com
               Action:
                 - sts:AssumeRole

   Outputs:

     //omit

     RDSProductionInstanceEndPoint:                                                                #(K)
       Condition: "ProductionResources"                                                            #(L)
       Description: RDS
       Value: !GetAtt RDSProductionInstance.Endpoint.Address
       Export:
         Name: !Sub ${VPCName}-RDSEndpoint-Production

    // omit

|br|

ALBのテンプレートの記述の基本となるポイントは(A)〜(L)の通りです。

|br|

.. list-table:: ALBのCloudFormationテンプレート記述のポイント
   :widths: 1, 9

   * - 記述
     - 説明

   * - (A)
     - パラメータ要素として、RDSのユーザを作成します。

   * - (B)
     - RDSを構築する環境をEnvTypeパラメータとして指定可能にします。このパラメータに応じて、Conditionsを設定し、作成するリソースを切り替えます。

   * - (C)
     - Condtionsとして、EnvTypeパラメータの値に応じて、３つの論理名を定義します※。

   * - (D)
     - 商用環境向けのDBインスタンスのリソース定義を行います。詳細は `AWS::RDS::DBInstance <https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-properties-rds-database-instance.html>`_ を参照してください。なお、サブネットグループやモニタリングロールは(I)、(J)の定義を、適用するセキュリティグループは :ref:`section-cloudformation-security-group-sample-label` で作成したものを、クロススタックリファレンスを使って参照します。

   * - (E)
     - (C)で定義したConditionsの論理名が"ProductionResources"だった場合に、リソース定義が有効化するよう、Condition要素を定義します。

   * - (F)
     -  DynamicReferences機能※※を使って、AWS SystemsManager ParameterStoreからRDSユーザのパスワードを取得します。ここでは"mynavi-sample-cloudformation-rds-password"という定義名のセキュアパラメータを受け取る設定とします。

   * - (G)
     - ステージング環境向けのDBインスタンスのリソース定義を行います。詳細は `AWS::RDS::DBInstance <https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-properties-rds-database-instance.html>`_ を参照してください。なお、サブネットグループやモニタリングロールは(I)、(J)の定義を、適用するセキュリティグループは :ref:`section-cloudformation-security-group-sample-label` で作成したものを、クロススタックリファレンスを使って参照します。

   * - (H)
     - 開発環境向けのDBインスタンスのリソース定義を行います。詳細は `AWS::RDS::DBInstance <https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-properties-rds-database-instance.html>`_ を参照してください。なお、サブネットグループやモニタリングロールは(I)、(J)の定義を、適用するセキュリティグループは :ref:`section-cloudformation-security-group-sample-label` で作成したものを、クロススタックリファレンスを使って参照します。

   * - (I)
     - DBサブネットグループを定義します。詳細は、 `AWS::RDS::DBSubnetGroup <https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-rds-dbsubnet-group.html>`_ を参照してください。なお、サブネットは :ref:`section-cloudformation-vpc-sample-label` で作成したものを、クロススタックリファレンスを使って参照します。

   * - (J)
     - DBを監視するためのIAMロールを作成します。ポリシーはAmazonRDSEnhancedMonitoringRoleをアタッチします。詳細は、 `AWS::IAM::Role <https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-elasticloadbalancingv2-listener.html>`_ を参照してください。

   * - (K)
     - RDSのエンドポイントをOutputs出力します。

   * - (L)
     - (E)と同様、Conditionsの論理名が"ProductionResources"だった場合に、リソース定義が有効化するよう、Condition要素を定義します。

|br|

.. note:: ※Conditionsについて

   Conditionsは条件によって有効な論理名を決定し、ResourcesやOutputs要素などで、Condition要素で条件に一致する論理名が適用された際に定義が有効化されます。以下では、EnvTypeのデフォルトパラメータ"Dev"の場合、Conditionsに定義した論理名"DevResources"が有効化されます。Resources要素で各リソース定義にCondition要素を定義すると、"DevResources"の定義のみリソースが作成されます。

   |br|

   .. sourcecode:: none

      Parameters:
        EnvType:
          Description: Which environments to deploy your service.
          Type: String
          AllowedValues: ["Dev", "Staging", "Production"]
          Default: Dev

      Conditions:
        ProductionResources: {"Fn::Equals" : [{"Ref":"EnvType"}, "Production"]} #完全修飾形表記
        StagingResources: !Equals [ !Ref EnvType, "Staging"] #簡略化表記
        DevResources: {"Fn::Equals" : [{"Ref":"EnvType"}, "Dev"]}

      Resources:
        RDSDevInstance:
          Type: AWS::RDS::DBInstance
          DeletionPolicy: Snapshot
          Condition: "DevResources"
          Properties:
            //omit

   |br|

   Condtionsでの条件の書き方は完全修飾形と簡略化表記2種類選択できますが、可読性の観点から簡略化表記の記載をおすすめします。

|br|

.. note:: ※※Dynamic Referencesについて

   CLoudFormationからAWS Systems Manager ParametersStoreやSecretsManagerに保存されたデータアクセスが可能で「Dynamic References」と呼ばれます。以下のような書式でデータアクセスが可能です。

   |br|

   .. sourcecode:: none

      '{{resolve:service-name:reference-key}}'

   |br|

   詳細はAWS公式の `Dynamic Referencesを参照してテンプレートを指定する <https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/dynamic-references.html>`_ も参考にしてください。

|br|

続いて、AWSコンソール画面から、「AWS SystemsManager」を選択し、パスワードをセキュアパラメータを使って以下のイメージ通り設定します。

|br|

.. figure:: img/automation_infra_devops_cloudformation/management_console_ssm_create_secure_parameter.png

|br|

作成したテンプレートに対して、ヘルパースクリプトを以下のように、スタック名とテンプレートパスを変更して実行します。IAMロールを作成するので、"--capabilities CAPABILITY_IAM"オプションを忘れずに設定してコマンド実行してください。

|br|

.. sourcecode:: bash

   #!/usr/bin/env bash

   stack_name="mynavi-sample-rds"
   template_path="sample-alb-rds.yml"

   parameters="EnvType=Production"

   aws cloudformation deploy --stack-name ${stack_name} --template-file ${template_path} --parameter-overrides ${parameters} --capabilities CAPABILITY_IAM

|br|

実行が正常に終了すると、RDSが作成されます。

|br|

.. figure:: img/automation_infra_devops_cloudformation/management_console_cloudformation_stack_rds.png

|br|

今回はConditions要素やDynamicReferece機能を使いながら、RDSを構築するCloudFormationテンプレートを実装しました。次回は、DynamoDBを構築するテンプレートを作成します。

|br|

著者紹介
------------------------------------------------------------------

川畑 光平(KAWABATA Kohei) - NTTデータ 課長代理

.. figure:: img/automation_infra_devops_overview/pic_image01.jpg


金融機関システム業務アプリケーション開発・システム基盤担当を経て、現在はソフトウェア開発自動化関連の研究開発・推進に従事。

Red Hat Certified Engineer、Pivotal Certified Spring Professional、AWS Certified Solutions Architect Professional等の資格を持ち、アプリケーション基盤・クラウドなど様々な開発プロジェクト支援にも携わる。

`2019 APN AWS Top Engineers & Ambassadors <https://aws.amazon.com/jp/blogs/psa/japan-apn-ambassador-2019/>`_ 選出。
