.. include:: ../module.txt

.. _section-automation-infra-devops-cloudformation-5-label:

基盤・デプロイ自動化実践
==================================================================

マイクロサービスアーキテクチャの基盤・デプロイ自動化
-------------------------------------------------------------------------------------------------------------------------------------

|br|

本連載では、以下のイメージの構成にあるAWSリソース基盤自動化環境の構築を実践していきます。

|br|

.. figure:: img/automation_infra_devops_cloudformation/cloudformation-scope.png

|br|

前回は、CloudFormationの簡単なサンプルを作成し、テンプレート記述の基本を解説した上で、実行するためのヘルパースクリプトや実行エラーの確認方法等を説明しました。
今回は、VPCおよびパブリック、プライベートサブネット、ルートテーブルおよび、インターネットゲートウェイを作成します。
なお、このスタック構成は基本AWS無料枠で構成されるサービスです。スタック構築後も特に費用は発生しません。
実際のソースコードは `GitHub <https://github.com/debugroom/mynavi-sample-cloudformation>`_ 上にコミットしています。
ソースコード中で本質的でない記述を一部省略しているので、実行コードを作成する場合は、必要に応じて適宜GitHub上のソースコードも参照してください。


|br|

.. _section-cloudformation-vpc-sample-label:

VPC/Subnet/RouteTable/InternetGatewayスタック構築テンプレート
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

構築するVPC関連設定の内容は連載 `「クラウドネイティブアプリケーションの基本 第4回」 <https://news.mynavi.jp/itsearch/article/devsoft/4354>`_ で構築したVPC構成と基本的にほぼ同様です。
違いとしては、構築するIPアドレス体系や、構築すると費用が発生するNATGatewayを除外している点、コンソール上で作成された時には自動作成される暗黙のメインルートテーブルとサブネットへの関連づけがCloudFormationでは実行されないので、定義を明示的に行っている点です(メインルートテーブルの作成と関連付けは次回NATGatewayの設定で行います)。
サンプルとして作成するsample-vpc-cfn.ymlは以下の通りです。

|br|

.. sourcecode:: none

   AWSTemplateFormatVersion: '2010-09-09'

   // omit

   Parameters:
     VPCName:                                                                   #(A)
       Description: Target VPC Stack Name
       Type: String
       MinLength: 1
       MaxLength: 255
       AllowedPattern: ^[a-zA-Z][-a-zA-Z0-9]*$
       Default: mynavi-sample-cloudformation-vpc
     VPCCiderBlock:                                                             #(B)
       Description: CiderBlock paramater for VPC
       Type: String
       MinLength: 9
       MaxLength: 18
       AllowedPattern: (\d{1,3})\.(\d{1,3})\.(\d{1,3})\.(\d{1,3})/(\d{1,2})
       Default: 172.200.0.0/16
     PublicSubnet1CiderBlock:                                                   #(C)
       Description: CiderBlock paramater for VPC
       Type: String
       MinLength: 9
       MaxLength: 18
       AllowedPattern: (\d{1,3})\.(\d{1,3})\.(\d{1,3})\.(\d{1,3})/(\d{1,2})
       Default: 172.200.1.0/24

    // omit

   Resources:
     VPC:                                                                       #(D)
       Type: AWS::EC2::VPC
       Properties:
         CidrBlock: !Sub ${VPCCiderBlock}
         InstanceTenancy: default
         EnableDnsSupport: true
         EnableDnsHostnames: true
         Tags:
           - Key: Name
             Value: !Sub ${VPCName}

     PublicSubnet1:                                                             #(E)
        Type: AWS::EC2::Subnet
        Properties:
          CidrBlock: !Sub ${PublicSubnet1CiderBlock}
          VpcId: !Ref VPC
          AvailabilityZone: !Select [ 0, !GetAZs '' ]                           #(F)
          Tags:
            - Key: Name
              Value: !Sub ${VPCName}-PublicSubnet1

   // omit

     IGW:                                                                       #(G)
       Type: AWS::EC2::InternetGateway
       Properties:
         Tags:
           - Key: Name
             Value: !Sub ${VPCName}-IGW

     IGWAttach:                                                                 #(H)
       Type: AWS::EC2::VPCGatewayAttachment
       Properties:
         InternetGatewayId: !Ref IGW
         VpcId: !Ref VPC

     CustomRouteTable:                                                          #(I)
       Type: AWS::EC2::RouteTable
       Properties:
         VpcId: !Ref VPC
         Tags:
           - Key: Name
             Value: !Sub ${VPCName}-PublicRoute

     CustomRoute:                                                               #(J)
       Type: AWS::EC2::Route
       Properties:
         RouteTableId: !Ref CustomRouteTable
         DestinationCidrBlock: 0.0.0.0/0
         GatewayId: !Ref IGW

     PublicSubnet1Association:                                                  #(K)
       Type: AWS::EC2::SubnetRouteTableAssociation
       Properties:
         SubnetId: !Ref PublicSubnet1
         RouteTableId: !Ref CustomRouteTable

   // omit

   Outputs:
     VPC:                                                                       #(L)
       Description: VPC ID
       Value: !Ref VPC
       Export:
         Name: !Sub ${AWS::StackName}-VPCID

     PublicSubnet1:                                                             #(M)
       Description: PublicSubnet1
       Value: !Ref PublicSubnet1
       Export:
         Name: !Sub ${AWS::StackName}-PublicSubnet1

     PublicSubnet1Arn:                                                          #(N)
       Description: PublicSubnet1Arn
       Value: !Sub                                                              #(O)
         - arn:aws:ec2:${AWS::Region}:${AWS::AccountId}:subnet/${PublicSubnet1}
         - PublicSubnet1: !Ref PublicSubnet1
       Export:
         Name: !Sub ${AWS::StackName}-PublicSubnet1Arn

    // omit

|br|

テンプレートの記述の基本となるポイントは(A)〜(O)の通りです。

|br|

.. list-table:: VPC関連CloudFormationテンプレート記述のポイント
   :widths: 1, 9

   * - 記述
     - 説明

   * - (A)
     - ステージング環境・プロダクション環境など別のVPCを作る場合の再利用に備えて、VPC名をパラメータ化します。

   * - (B)
     - 異なるVPCでIPアドレス帯を変更できるようにパラメータ化しておきます。

   * - (C)
     - Bと同様、VPC内に構築するサブネットアドレス帯を変更できるようにパラメータ化します。なおここでは、１つのパブリックサブネットしか記載していませんが、パブリック・サブネット各2つずつ定義します。

   * - (D)
     - 構築するVPCリソースを定義します。設定可能なパラメータや必須項目は `公式ページ AWS::EC2::VPC <https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-vpc.html>`_ を参照してください。

   * - (E)
     - VPC内に構築するパブリック・プライベートサブネットリソースを定義します。設定可能なパラメータや必須項目は `公式ページ AWS::EC2::Subnet <https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-subnet.html>`_ を参照してください。なおここでは、１つのパブリックサブネットしか記載していませんが、パブリック・サブネット各2つずつ定義します。

   * - (F)
     - サブネットのAvailabilityZoneプロパティでは、組み込みファンクション「!GetAZs」や「!Select」を組み合わせて、リージョンで使用可能なアベイラビリティゾーンを取得します。

   * - (G)
     - VPCに設定するインターネットゲートウェイリソースを定義します。設定可能なパラメータや必須項目は `公式ページ AWS::EC2::InternetGateway <https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-internetgateway.html>`_ を参照してください。

   * - (H)
     - Gで作成したインターネットゲートウェイリソースをアタッチするVPCを定義するゲートウェイアタッチメントリソースを定義します。設定可能なパラメータや必須項目は `公式ページ AWS::EC2::VPCGatewayAttachment <https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-vpc-gateway-attachment.html>`_ を参照してください。

   * - (I)
     - インターネットゲートウェイへ到達するルートテーブルを作成します。設定可能なパラメータや必須項目は `公式ページ AWS::EC2::RouteTable <https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-route-table.html>`_ を参照してください。

   * - (J)
     - インターネットゲートウェイへ到達するルートを作成します。設定可能なパラメータや必須項目は `公式ページ AWS::EC2::Route <https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-route.html>`_ を参照してください。

   * - (K)
     - Jで作成したルートとパブリックサブネットを関連づけます。設定可能なパラメータや必須項目は `公式ページ AWS::EC2::SubnetRouteTableAssociation <https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-subnet-route-table-assoc.html>`_ を参照してください。

   * - (L)
     - VPCの物理IDをOutputsとして出力します。この値は次回以降別のテンプレートで使用します(クロススタックリファレンス)。

   * - (M)
     - VPC内に構築したパブリックサブネットの物理IDをOutputsとして出力します。なおここでは、１つのパブリックサブネットしか記載していませんが、パブリック・サブネット各2つずつ定義します。

   * - (N)
     - VPC内に構築したパブリックサブネットのARN(AmazonResourceName)をOutputsとして出力します。なおここでは、１つのパブリックサブネットしか記載していませんが、パブリック・サブネット各2つずつ定義します。この値は次回以降別のテンプレートで使用します(クロススタックリファレンス)。

   * - (O)
     - 組み込み関数「!Sub」やAWSデフォルトの擬似パラメータを用いて、サブネットのARNを組み立てます。!Subを配列で構成し、配列の第一要素で変換文字列、第二要素で変換パラメータを指定してARNを出力しています。この記法の詳細については、 `AWS公式ページ Fn::Sub <https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-sub.html>`_ を参照してください。

|br|

作成したテンプレートに対して、ヘルパースクリプトを以下のように、スタック名とテンプレートパスを変更して実行します。パラメータはデフォルト値を利用するので省略します。

|br|

.. sourcecode:: bash

   #!/usr/bin/env bash

   stack_name="mynavi-sample-vpc"
   template_path="sample-vpc-cfn.yml"
   #parameters="VPCCiderBlock=172.200.0.0/16"

   if [ "$parameters" == "" ]; then
       aws cloudformation deploy --stack-name ${stack_name} --template-file ${template_path} --capabilities CAPABILITY_IAM
   else
       aws cloudformation deploy --stack-name ${stack_name} --template-file ${template_path} --parameter-overrides ${parameters} --capabilities CAPABILITY_IAM
   fi

|br|

実行が正常に終了すると、VPCやサブネット、インターネットゲートウェイが作成されます。

|br|

.. figure:: img/automation_infra_devops_cloudformation/management_console_cloudformation_stack_vpc.png

|br|

次回は、セキュリティグループおよび、NATゲートウェイ、アプリケーションロードバランサーを構築するスタックテンプレートの解説です。

|br|

著者紹介
------------------------------------------------------------------

川畑 光平(KAWABATA Kohei) - NTTデータ 課長代理

.. figure:: img/automation_infra_devops_overview/pic_image01.jpg


金融機関システム業務アプリケーション開発・システム基盤担当を経て、現在はソフトウェア開発自動化関連の研究開発・推進に従事。

Red Hat Certified Engineer、Pivotal Certified Spring Professional、AWS Certified Solutions Architect Professional等の資格を持ち、アプリケーション基盤・クラウドなど様々な開発プロジェクト支援にも携わる。

`2019 APN AWS Top Engineers & Ambassadors <https://aws.amazon.com/jp/blogs/psa/japan-apn-ambassador-2019/>`_ 選出。
