.. include:: ../module.txt

.. _section-automation-infra-devops-cloudformation-11-label:

基盤・デプロイ自動化実践
==================================================================

マイクロサービスアーキテクチャの基盤・デプロイ自動化
-------------------------------------------------------------------------------------------------------------------------------------

|br|

本連載では、以下のイメージの構成にあるAWSリソース基盤自動化環境の構築を実践しています。

|br|

.. figure:: img/automation_infra_devops_cloudformation/cloudformation-scope.png

|br|

前回は、DynamoDBを構築するテンプレートを実装しました。続く今回はElastiCacheを構築するテンプレートを作成します。
実際のソースコードは `GitHub <https://github.com/debugroom/mynavi-sample-cloudformation>`_ 上にコミットしています。
ソースコード中で本質的でない記述を一部省略しているので、実行コードを作成する場合は、必要に応じて適宜GitHub上のソースコードも参照してください。

|br|

.. _section-cloudformation-elasticache-sample-label:

ElastiCacheスタック構築テンプレート
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

ElastiCacheは `クラウドネイティブ基本第22回 <https://news.mynavi.jp/itsearch/article/devsoft/4543>`_ で実施した要領と同等のものを構築します。
ElastiCache(Redis)をCloudFormationで構築する場合、リソースタイプが、 `AWS::ElastiCache::ReplicationGroup <https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-elasticache-replicationgroup.html>`_
であるElastiCache定義および、 `AWS::ElastiCache::SubnetGroup <https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-properties-elasticache-subnetgroup.html>`_ となるサブネットグループ定義、
`AWS::ElastiCache::ParameterGroup <https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-properties-elasticache-parameter-group.html>`_ となるパラメータグループ定義が必要です。
プロパティとして設定可能な属性は、各リンク先の通りですが、ElastiCacheを商用環境、ステージング環境、開発環境の3つのパターンに分けて作成するようにします。

.. note::  `クラウドネイティブ基本第19回 <https://news.mynavi.jp/itsearch/article/devsoft/4523>`_ では、開発環境はローカル環境にRedisServerを立てるかたちで解説しました。ElastiCacheはRDSやDynamoDBとは異なり、2020年2月時点でもパブリックアクセスが許可されていません(パブリックサブネットに配置してVPC外からエンドポイントを指定してもアクセスできません)。
   そのため、VPCの外にある開発環境ではローカルにRedisServerを立てる必要があります。開発環境でElastiCacheを使用したい場合はAmazon WorkspacesなどでVPC内に仮想マシン環境を用意する方法が簡易です。今回はVPC内に開発環境があることを想定して、開発環境用のElastiCacheをCloudFormationテンプレートで構築してみます。

|br|

.. warning:: ElastiCacheには `AWS::ElastiCache::CacheCluster <https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-properties-elasticache-cache-cluster.html>`_ リソースタイプもありますが、Redisを使用する場合はノード数を2つ以上にするとエラーとなります。エンジンがRedisの場合は、AWS::ElastiCache::ReplicationGroupを使用しましょう。

|br|

テンプレートのサンプルは以下の通りです。


|br|

.. sourcecode:: none

   AWSTemplateFormatVersion: '2010-09-09'

   // omit

   Parameters:
     // omit
     EnvType:                                                                                                #(A)
       Description: Which environments to deploy your service.
       Type: String
       AllowedValues: ["Dev", "Staging", "Production"]
       Default: Dev
     SecurityGroupDev:                                                                                       #(B)
       Description: The local machine permitted SecurityGroup is needed as paramater for Dev.
       Type: String
       Default: ""
   Conditions:                                                                                               #(C)
     DevResources: {"Fn::Equals" : [{"Ref":"EnvType"}, "Dev"]}

   Mappings:
     CacheDefinitionMap:                                                                                     #(D)
       Production:
         "CacheInstanceType": "cache.t2.micro"
         "CacheParameterGroupFamily": "redis5.0"
         "EngineVersion": "5.0.3"
         "Port": 6379
         "ClusterEnabled" : "yes"
         "AutomaticFailoverEnabled": true
         "ReplicasPerNodeGroup": 2
       Staging:
         "CacheInstanceType": "cache.t2.micro"
         "CacheParameterGroupFamily": "redis5.0"
         "EngineVersion": "5.0.3"
         "Port": 6379
         "ClusterEnabled" : "no"
         "AutomaticFailoverEnabled": false
         "ReplicasPerNodeGroup": 1
       Dev:
         "CacheInstanceType": "cache.t2.micro"
         "CacheParameterGroupFamily": "redis5.0"
         "EngineVersion": "5.0.3"
         "Port": 6379
         "ClusterEnabled" : "no"
         "AutomaticFailoverEnabled": false
         "ReplicasPerNodeGroup": 1

   Resources:
     ElastiCacheSubnetGroup:                                                                                 #(E)
       Type: AWS::ElastiCache::SubnetGroup
       Properties:
         CacheSubnetGroupName: ElastiCacheSubnetGroup
         Description: SampleCloudFormation ElastiCacheSubnetGroup
         SubnetIds:
           - Fn::ImportValue: !Sub ${VPCName}-PublicSubnet1
           - Fn::ImportValue: !Sub ${VPCName}-PublicSubnet2

     ElastiCacheParameterGroup:                                                                              #(F)
       Type: AWS::ElastiCache::ParameterGroup
       Properties:
         CacheParameterGroupFamily: !FindInMap [CacheDefinitionMap, !Ref EnvType, CacheParameterGroupFamily] #(G)
         Description: SampleCloudFormation ElastiCacheParameterGroup
         Properties:
           cluster-enabled: !FindInMap [CacheDefinitionMap, !Ref EnvType, ClusterEnabled]

     ElastiCacheRedis:                                                                                       #(H)
       Type: AWS::ElastiCache::ReplicationGroup
       Properties:
         ReplicationGroupId: !Sub elasticache-${EnvType}
         Engine: redis
         ReplicationGroupDescription: SampleCloudFormation RedisCluster
         EngineVersion: !FindInMap [CacheDefinitionMap, !Ref EnvType, EngineVersion]
         Port: !FindInMap [CacheDefinitionMap, !Ref EnvType, Port]
         CacheParameterGroupName: !Ref ElastiCacheParameterGroup
         CacheNodeType: !FindInMap [CacheDefinitionMap, !Ref EnvType, CacheInstanceType]
         ReplicasPerNodeGroup: !FindInMap [CacheDefinitionMap, !Ref EnvType, ReplicasPerNodeGroup]
         AutomaticFailoverEnabled: !FindInMap [CacheDefinitionMap, !Ref EnvType, AutomaticFailoverEnabled]
         CacheSubnetGroupName: !Ref ElastiCacheSubnetGroup
         SecurityGroupIds:
           - Fn::ImportValue: !Sub ${VPCName}-SecurityGroupElastiCacheRedis
           - !If ["DevResources", !Ref SecurityGroupDev, !Ref "AWS::NoValue"]                                 #(I)

   Outputs:
    // omit
     ElastiCacheRedisEndPoint:                                                                                #(J)
       Description: ElastiCache Redis EndPoint
       Value: !GetAtt ElastiCacheRedis.PrimaryEndPoint.Address
       Export:
         Name: !Sub ${VPCName}-ElastiCacheRedisEndPoint

     ElastiCacheRedisPort:                                                                                    #(K)
       Description: ElastiCache Redis Port
       Value:  !FindInMap [CacheDefinitionMap, !Ref EnvType, Port]
       Export:
         Name: !Sub ${VPCName}-ElastiCacheRedisPort-${EnvType}

|br|

ElastiCacheのテンプレートの記述の基本となるポイントは(A)〜(K)の通りです。

|br|

.. list-table:: ElastiCacheのCloudFormationテンプレート記述のポイント
   :widths: 1, 9

   * - 記述
     - 説明

   * - (A)
     - ElastiCacheを商用環境、ステージング環境、開発環境用に分けるよう、EnvTypeパラメータとして指定可能にします。このパラメータに応じて、Mappingsに定義したパラメータを切り替え、リソース定義を分けて作成できるようにします。

   * - (B)
     - 開発環境としてのElastiCacheにアクセス可能な端末をターゲットとしたセキュリティグループをパラメータとして受け取ります。 `第27回 <https://news.mynavi.jp/itsearch/article/devsoft/4829>`_ と同様、事前にセキュリティグループを定義し、IDをパラメータとして渡す前提で実装しておきます。

   * - (C)
     - Conditionsとして、EnvTypeがDevだった場合に、有効になる論理名を定義します。定義方法の詳細は :ref:`section-cloudformation-rds-sample-label` と同様なので適宜参照してください。

   * - (D)
     - Mappingsとして、EnvTypeパラメータの値に応じて、2つの論理名を定義します。Mappings定義方法の詳細は :ref:`section-cloudformation-alb-sample-label` と同様なので適宜参照してください。

   * - (E)
     - ElastiCacheを配置するサブネットグループのリソース定義を行います。定義するプロパティの詳細は `AWS::ElastiCache::SubnetGroup <https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-properties-elasticache-subnetgroup.html>`_  を参照してください。

   * - (F)
     - ElastiCacheにおけるパラメータグループのリソース定義を行います。定義するプロパティの詳細は `AWS::ElastiCache::ParameterGroup <https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-properties-elasticache-parameter-group.html>`_ を参照してください。

   * - (G)
     - 環境によって異なる値を定義したい箇所は、EnvTypeパラメータを引数としたFindInMap関数でデータ取得します。

   * - (H)
     - ElastiCacheのリソース定義を行います。定義するプロパティの詳細は  `AWS::ElastiCache::ReplicationGroup <https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-elasticache-replicationgroup.html>`_ を参照してください。

   * - (I)
     - 開発環境の場合のみ、パラメータとして指定したセキュリティグループが追加されるように定義します。

   * - (J)
     - ElastiCacheのプライマリエンドポイントをOutputs出力します。

   * - (K)
     - ElastiCacheのポートをOutputs出力します。

|br|

作成したテンプレートに対して、ヘルパースクリプトを以下のように、スタック名とテンプレートパスを変更して実行します。

|br|

.. sourcecode:: bash

   #!/usr/bin/env bash

   stack_name="mynavi-sample-elasticache"
   template_path="sample-elasticache-cfn.yml"

   parameters="EnvType=Staging"

   aws cloudformation deploy --stack-name ${stack_name} --template-file ${template_path} --parameter-overrides ${parameters} --capabilities CAPABILITY_IAM

|br|

実行が正常に終了すると、ElastiCacheが作成されます。

|br|

.. figure:: img/automation_infra_devops_cloudformation/management_console_cloudformation_stack_elasticache.png

|br|

今回はElastiCacheを構築するCloudFormationテンプレートを実装しました。次回は、AmazonS3やAmazonSQSを構築するテンプレートを作成します。

|br|

著者紹介
------------------------------------------------------------------

川畑 光平(KAWABATA Kohei) - NTTデータ 課長代理

.. figure:: img/automation_infra_devops_overview/pic_image01.jpg


金融機関システム業務アプリケーション開発・システム基盤担当を経て、現在はソフトウェア開発自動化関連の研究開発・推進に従事。

Red Hat Certified Engineer、Pivotal Certified Spring Professional、AWS Certified Solutions Architect Professional等の資格を持ち、アプリケーション基盤・クラウドなど様々な開発プロジェクト支援にも携わる。

`2019 APN AWS Top Engineers & Ambassadors <https://aws.amazon.com/jp/blogs/psa/japan-apn-ambassador-2019/>`_ 選出。
