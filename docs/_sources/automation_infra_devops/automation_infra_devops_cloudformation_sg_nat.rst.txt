.. include:: ../module.txt

.. _section-automation-infra-devops-cloudformation-7-label:

基盤・デプロイ自動化実践
==================================================================

マイクロサービスアーキテクチャの基盤・デプロイ自動化
-------------------------------------------------------------------------------------------------------------------------------------

|br|

本連載では、以下のイメージの構成にあるAWSリソース基盤自動化環境の構築を実践しています。

|br|

.. figure:: img/automation_infra_devops_cloudformation/cloudformation-scope.png

|br|

前回は、VPCおよびパブリック、プライベートサブネット、ルートテーブルおよび、インターネットゲートウェイを作成するCloudFormationテンプレートを作成しました。
今回は、作成した各パブリック・プライベートサブネットをFrontend、Backendサブネットとして位置付け、各AWSリソースへ設定するセキュリティグループ、FrontendサブネットにアタッチするNATGatewayを構築するテンプレートを実装します。
なお、セキュリティグループを作成するスタック構成は基本AWS無料枠で構成されるサービスですが、NATGatewayはスタック構築後に料金が発生するようになります。
NATGatewayはECSを実行する際に必要になってくる(NATGatewayがないと、外からコンテナイメージを取得できなくなり起動が失敗する)ので、注意が必要ですが、
不必要に費用が発生しないよう、NATGatewayの構築は別のスタックに切り出しておきましょう。
実際のソースコードは `GitHub <https://github.com/debugroom/mynavi-sample-cloudformation>`_ 上にコミットしています。
ソースコード中で本質的でない記述を一部省略しているので、実行コードを作成する場合は、必要に応じて適宜GitHub上のソースコードも参照してください。

|br|

.. _section-cloudformation-security-group-sample-label:

セキュリティグループスタック構築テンプレート
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

クラウドネイティブ連載や本連載ではこれまで、以下のようなリソースに対するセキュリティグループを作成してきました。

|br|

.. list-table:: これまで作成したセキュリティグループ
   :widths: 3, 3, 2, 5

   * - 連載
     - 対象
     - タイプ
     - 設定

   * - `クラウドネイティブ第5回 <https://news.mynavi.jp/itsearch/article/devsoft/4359>`_
     - FrontendALB
     - インバウンド接続
     - 任意のアドレスからの80番ポートの接続許可

   * - `クラウドネイティブ第5回 <https://news.mynavi.jp/itsearch/article/devsoft/4359>`_
     - BackendALB
     - インバウンド接続
     - VPC内からの80番ポートの接続許可

   * - `クラウドネイティブ第8回 <https://news.mynavi.jp/itsearch/article/devsoft/4405>`_
     - Frontendサブネットに配置するECSクラスタ
     - インバウンド接続
     - FrontendALBから32768-61000ポートの接続許可

   * - `クラウドネイティブ第8回 <https://news.mynavi.jp/itsearch/article/devsoft/4405>`_
     - Frontendサブネットに配置するECSクラスタ
     - インバウンド接続
     - 任意のアドレスからのSSH接続

   * - `クラウドネイティブ第8回 <https://news.mynavi.jp/itsearch/article/devsoft/4405>`_
     - Backendサブネットに配置するECSクラスタ
     - インバウンド接続
     - BackendALBから32768-61000ポートの接続許可

   * - `クラウドネイティブ第11回 <https://news.mynavi.jp/itsearch/article/devsoft/4422>`_
     - BackendサブネットからアクセスされるRDS
     - インバウンド接続
     - Backendサブネットからの5432番ポートへの接続許可

   * - `クラウドネイティブ第22回 <https://news.mynavi.jp/itsearch/article/devsoft/4543>`_
     - FrontendサブネットからアクセスされるElastiCache
     - インバウンド接続
     - Frontendサブネットに配置されたECSクラスタからの6379番ポートへの接続許可

   * - `基盤構築・デプロイ自動化第11回 <https://news.mynavi.jp/itsearch/article/devsoft/4595>`_
     - CodeBuildでビルド実行するコンテナ
     - アウトバウンド接続
     - 任意のアドレスへのアウトバウンド許可

|br|

上記のセキュリティグループをCloudFormationで構築する場合、リソースタイプが、`AWS::EC2::SecurityGroup <https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-properties-ec2-security-group.html>`_
となるセキュリティグループ自体の定義と、接続可能な条件を設定する `AWS::EC2::SecurityGroupIngress(インバウンド) <https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-properties-ec2-security-group-ingress.html>`_
もしくは `AWS::EC2::SecurityGroupEgress(アウトバウンド接続) <https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-security-group-egress.html>`_ が必要になります。
プロパティとして設定可能な属性は、各リンク先の通りですが、上記の表のセキュリティグループを作成するsample-sg-cfn.ymlは以下の通りです。

|br|

.. sourcecode:: none

   AWSTemplateFormatVersion: '2010-09-09'

   // omit

   Resources:
     SecurityGroupFrontendALB:                                       #(A)
       Type: AWS::EC2::SecurityGroup
       Properties:
         GroupName: SecurityGroupFrontendALB
         GroupDescription: http access
         VpcId:
           Fn::ImportValue: !Sub ${VPCName}-VPCID                    #(B)
         Tags:
           - Key : Name
             Value: !Sub ${VPCName}-SecurityGroupFrontendALB

     SecurityGroupInggressFrontendALB:                               #(C)
       Type: AWS::EC2::SecurityGroupIngress
       Properties:
         GroupId: !Ref SecurityGroupFrontendALB                      #(D)
         IpProtocol: tcp
         FromPort: 80
         ToPort: 80
         CidrIp: 0.0.0.0/0                                           #(E)

     SecurityGroupBackendALB:                                        #(F)
       Type: AWS::EC2::SecurityGroup
       Properties:
         GroupName: SecurityGroupBackendALB
         GroupDescription: http access
         VpcId:
           Fn::ImportValue: !Sub ${VPCName}-VPCID
         Tags:
           - Key : Name
             Value: !Sub ${VPCName}-SecurityGroupBackendALB

     SecurityGroupIngressBackendALB:                                 #(G)
       Type: AWS::EC2::SecurityGroupIngress
       Properties:
         GroupId: !Ref SecurityGroupBackendALB
         IpProtocol: tcp
         FromPort: 80
         ToPort: 80
         CidrIp: !Ref VPCCiderBlock                                  #(H)

     SecurityGroupFrontendEcsCluster:                                #(I)
       Type: AWS::EC2::SecurityGroup
       Properties:
         GroupName: SecurityGroupFrontendEcsCluster
         GroupDescription: http access only alb
         VpcId:
           Fn::ImportValue: !Sub ${VPCName}-VPCID
         Tags:
           - Key : Name
             Value: !Sub ${VPCName}-SecurityGroupFrontendEcsCluster

     SecurityGroupIngressFrontendEcsCluster:                         #(J)
       Type: AWS::EC2::SecurityGroupIngress
       Properties:
         GroupId: !Ref SecurityGroupFrontendEcsCluster
         IpProtocol: tcp
         FromPort: 32768
         ToPort: 61000
         SourceSecurityGroupId: !Ref SecurityGroupFrontendALB        #(K)

     SecurityGroupIngressForSSHFrontendEcsCluster:                   #(L)
       Type: AWS::EC2::SecurityGroupIngress
       Properties:
         GroupId: !Ref SecurityGroupFrontendEcsCluster
         IpProtocol: tcp
         FromPort: 22
         ToPort: 22
         CidrIp: 0.0.0.0/0

     SecurityGroupBackendEcsCluster:                                 #(M)
       Type: AWS::EC2::SecurityGroup
       Properties:
         GroupName: SecurityGroupBackendEcsCluster
         GroupDescription: http access only alb
         VpcId:
           Fn::ImportValue: !Sub ${VPCName}-VPCID
         Tags:
           - Key : Name
             Value: !Sub ${VPCName}-SecurityGroupBackendEcsCluster

     SecurityGroupIngressBackendEcsCluster:                          #(N)
       Type: AWS::EC2::SecurityGroupIngress
       Properties:
         GroupId: !Ref SecurityGroupBackendEcsCluster
         IpProtocol: tcp
         FromPort: 32768
         ToPort: 61000
         SourceSecurityGroupId: !Ref SecurityGroupBackendALB         #(O)

     SecurityGroupRdsPostgres:                                       #(P)
       Type: AWS::EC2::SecurityGroup
       Properties:
         GroupName: SecurityGroupRdsPostgres
         GroupDescription: db access only backend subnet
         VpcId:
           Fn::ImportValue: !Sub ${VPCName}-VPCID
         Tags:
           - Key: Name
             Value: !Sub ${VPCName}-SecurityGroupRdsPostgres

     SecurityGroupIngressRdsPostgres:                                #(Q)
       Type: AWS::EC2::SecurityGroupIngress
       Properties:
         GroupId: !Ref SecurityGroupRdsPostgres
         IpProtocol: tcp
         FromPort: 5432
         ToPort: 5432
         SourceSecurityGroupId: !Ref SecurityGroupBackendEcsCluster  #(R)

     SecurityGroupElastiCacheRedis:                                  #(S)
       Type: AWS::EC2::SecurityGroup
       Properties:
         GroupName: SecurityGroupElastiCacheRedis
         GroupDescription: redis access only frontend ecs cluster
         VpcId:
           Fn::ImportValue: !Sub ${VPCName}-VPCID
         Tags:
           - Key: Name
             Value: !Sub ${VPCName}-SecurityGroupElastiCacheRedis

     SecurityGroupIngressElastiCacheRedis:                           #(T)
       Type: AWS::EC2::SecurityGroupIngress
       Properties:
         GroupId: !Ref SecurityGroupElastiCacheRedis
         IpProtocol: tcp
         FromPort: 6379
         ToPort: 6379
         SourceSecurityGroupId: !Ref SecurityGroupFrontendEcsCluster #(U)

     SecurityGroupCodeBuild:                                         #(V)
       Type: AWS::EC2::SecurityGroup
       Properties:
         GroupName: SecurityGroupCodeBuild
         GroupDescription: CodeBuild environments
         VpcId:
           Fn::ImportValue: !Sub ${VPCName}-VPCID
         Tags:
           - Key: Name
             Value: !Sub ${VPCName}-SecurityGroupCodeBuild

    // omit

|br|

セキュリティグループのテンプレートの記述の基本となるポイントは(A)〜(V)の通りです。

|br|

.. list-table:: セキュリティグループCloudFormationテンプレート記述のポイント
   :widths: 1, 9

   * - 記述
     - 説明

   * - (A)
     - Frontendサブネットに配置するALBのセキュリティグループを定義します。

   * - (B)
     - セキュリティグループを使用するVPCのIDを設定します。ここでは、Fn::ImportValue関数を使って、前回作成したVPCテンプレートでOutputsとして出力したVPCの物理IDを取得します(クロススタックリファレンス)。

   * - (C)
     - (A)で定義したセキュリティグループに割り当てるインバウンド接続ルールを定義します。

   * - (D)
     - インバウンド接続ルールをどのセキュリティグループにあてるか定義します。複数の接続ルールを一つのセキュリティグループに割り当てることもできます。

   * - (E)
     - 接続許可されるプロトコル、ポート、送信元を定義します。

   * - (F)
     - BackendServiceサブネットに配置するALBのセキュリティグループを定義します。設定内容は(A)と同様です。

   * - (G)
     - (F)で定義したセキュリティグループに割り当てるインバウンド接続ルールを定義します。

   * - (H)
     - 送信元のCIDRを設定します。接続可能な範囲はVPC内からのアクセスですが、前回CIDRはパラメータ要素として設定しているので、今回も同様にParametersから取得するものとします。

   * - (I)
     - Frontendサブネットに配置するECSクラスタのセキュリティグループを定義します。設定内容は(A)と同様です。

   * - (J)
     - (I)で定義したセキュリティグループに割り当てるインバウンド接続ルールを定義します。

   * - (K)
     - 送信元をFrontendサブネットに配置したALBに設定するため、SourceSecurityGroupIdにFrontendALBのセキュリティグループ(A)を設定します。

   * - (L)
     - (I)で定義したセキュリティグループにSSHの接続を２つ目のルールとして作成し割り当てます。

   * - (M)
     - Backendサブネットに配置するECSクラスタのセキュリティグループを定義します。設定内容は(A)と同様です。

   * - (N)
     - (M)で定義したセキュリティグループに割り当てるインバウンド接続ルールを定義します。

   * - (O)
     - 送信元をBackendサブネットに配置したALBに設定するため、SourceSecurityGroupIdにBackendALBのセキュリティグループ(F)を設定します。

   * - (P)
     - RDSに設定するセキュリティグループを定義します。

   * - (Q)
     - (P)で定義したセキュリティグループに割り当てるインバウンド接続ルールを定義します。設定内容は(A)と同様です。

   * - (R)
     - 送信元をBackendサブネットに配置したECSクラスタに設定するため、SourceSecurityGroupIdにBackendECSクラスタのセキュリティグループ(M)を設定します。

   * - (S)
     - ElastiCacheに設定するセキュリティグループを定義します。設定内容は(A)と同様です。

   * - (T)
     - (S)で定義したセキュリティグループに割り当てるインバウンド接続ルールを定義します。

   * - (U)
     - 送信元をFrontendサブネットに配置したECSクラスタに設定するため、SourceSecurityGroupIdにFrontendECSクラスタのセキュリティグループ(I)を設定します。

   * - (V)
     - CodeBuildに設定するセキュリティグループを定義します。アウトバウンド接続はデフォルトで全ての通信が許可されるため、SecurityGroupEgressの設定は必要ありません。

|br|

作成したテンプレートに対して、ヘルパースクリプトを以下のように、スタック名とテンプレートパスを変更して実行します。パラメータはデフォルト値を利用するので省略します。

|br|

.. sourcecode:: bash

   #!/usr/bin/env bash

   stack_name="mynavi-sample-sg"
   template_path="sample-sg-cfn.yml"

   aws cloudformation deploy --stack-name ${stack_name} --template-file ${template_path} --capabilities CAPABILITY_IAM

|br|

実行が正常に終了すると、セキュリティグループが作成されます。

|br|

.. figure:: img/automation_infra_devops_cloudformation/management_console_cloudformation_stack_sg.png


|br|

.. _section-cloudformation-natgateway-sample-label:

NATGatewayスタック構築テンプレート
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

続いて、FrontendサブネットにNATGatewayを配置するCloudFormationテンプレートを作成します。
NATGatewayは `クラウドネイティブ連載第4回 <https://news.mynavi.jp/itsearch/article/devsoft/4354>`_ のVPC構築の中で同時に作成したものですが、
時間単位で費用が発生するのでNATGatwayだけ切り出していつでも配置を外せるようにしておきます。テンプレートではNATGatewayのリソース定義に加えて、ElasticIPアドレスやメインルートテーブル、パブリックサブネットのアタッチ定義なども必要です。

|br|

.. sourcecode:: none

   AWSTemplateFormatVersion: '2010-09-09'

   // omit

   Resources:
     NatGWEIP:                                               #(A)
       Type: AWS::EC2::EIP
       Properties:
         Domain:
           Fn::ImportValue: !Sub ${VPCName}-VPCID

     NatGW:                                                  #(B)
       Type: AWS::EC2::NatGateway
       Properties:
         AllocationId: !GetAtt NatGWEIP.AllocationId         #(C)
         SubnetId:
           Fn::ImportValue: !Sub ${VPCName}-PublicSubnet1    #(D)
         Tags:
           - Key: Name
             Value: !Sub ${VPCName}-NatGW

     MainRouteTable:                                         #(E)
       Type: AWS::EC2::RouteTable
       Properties:
         VpcId:
           Fn::ImportValue: !Sub ${VPCName}-VPCID
         Tags:
           - Key: Name
             Value: !Sub ${VPCName}-PrivateRoute

     MainRoute:                                              #(F)
       Type: AWS::EC2::Route
       Properties:
         RouteTableId: !Ref MainRouteTable
         DestinationCidrBlock: 0.0.0.0/0
         NatGatewayId: !Ref NatGW

     PrivateSubnet1Association:                              #(G)
       Type: AWS::EC2::SubnetRouteTableAssociation
       Properties:
         SubnetId:
           Fn::ImportValue: !Sub ${VPCName}-PrivateSubnet1
         RouteTableId: !Ref MainRouteTable

     // omit


|br|

NatGatewayのテンプレートの記述の基本となるポイントは(A)〜(G)の通りです。

|br|

.. list-table:: NATGateway CloudFormationテンプレート記述のポイント
   :widths: 1, 9

   * - 記述
     - 説明

   * - (A)
     - NatGatewayに割り当てるElasticIPアドレスを定義します。設定するプロパティは `AWS::EC2::EIP <https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-properties-ec2-eip.html>`_ を参照してください。Domainプロパティにはクロススタックリファレンスにより前回定義したVPCを指定します。

   * - (B)
     - NatGatewayリソースを定義します。設定するプロパティは `AWS::EC2::NatGateway <https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-natgateway.html>`_ を参照して下さい。

   * - (C)
     - AllocationIdプロパティには、GetAtt関数を用いて、(A)で定義したElasticIPアドレスのものを設定します。

   * - (D)
     - SubnetIdプロパティには、前回作成したパブリックサブネットをクロススタックリファレンスを使って指定します。

   * - (E)
     - メインとして設定するルートテーブルを定義します。設定するプロパティは `AWS::EC2::RouteTable <https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-route-table.html>`_ を参照してください。

   * - (F)
     - メインとして設定するルートを定義します。設定するプロパティは `AWS::EC2::Route <https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-route.html>`_ を参照してください。

   * - (G)
     - (E)で設定したルートテーブルのプライベートサブネットへの関連づけを定義します。設定するプロパティは `AWS::EC2::SubnetRouteTableAssociation <https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-subnet-route-table-assoc.html>`_ を参照してください。

|br|

作成したテンプレートに対して、ヘルパースクリプトを以下のように、スタック名とテンプレートパスを変更して実行します。

|br|

.. sourcecode:: bash

   #!/usr/bin/env bash

   stack_name="mynavi-sample-ng"
   template_path="sample-ng-cfn.yml"

   aws cloudformation deploy --stack-name ${stack_name} --template-file ${template_path} --capabilities CAPABILITY_IAM

|br|

実行が正常に終了すると、NATGatewayが作成され、アタッチされます。

|br|

.. figure:: img/automation_infra_devops_cloudformation/management_console_cloudformation_stack_ng.png

|br|

今回はセキュリティグループおよびNATGatewayをCloudFormationテンプレートで構築しました。次回は、アプリケーションロードバランサーを構築するスタックテンプレートを作成します。

|br|

著者紹介
------------------------------------------------------------------

川畑 光平(KAWABATA Kohei) - NTTデータ 課長代理

.. figure:: img/automation_infra_devops_overview/pic_image01.jpg


金融機関システム業務アプリケーション開発・システム基盤担当を経て、現在はソフトウェア開発自動化関連の研究開発・推進に従事。

Red Hat Certified Engineer、Pivotal Certified Spring Professional、AWS Certified Solutions Architect Professional等の資格を持ち、アプリケーション基盤・クラウドなど様々な開発プロジェクト支援にも携わる。

`2019 APN AWS Top Engineers & Ambassadors <https://aws.amazon.com/jp/blogs/psa/japan-apn-ambassador-2019/>`_ 選出。
