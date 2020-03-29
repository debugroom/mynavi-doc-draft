.. include:: ../module.txt

.. _section-automation-infra-devops-cloudformation-12-label:

【第32回】AWS CloudFormationを用いた基盤自動化(12)SQS/S3の構築
-------------------------------------------------------------------------------------------------------------------------------------

|br|

本連載では、以下のイメージの構成にあるAWSリソース基盤自動化環境の構築を実践しています。

|br|

.. figure:: img/automation_infra_devops_cloudformation/cloudformation-scope.png

|br|

前回は、ElastiCacheを構築するテンプレートを実装しました。続く今回はS3およびSQSを構築するテンプレートを作成します。
実際のソースコードは `GitHub <https://github.com/debugroom/mynavi-sample-cloudformation>`_ 上にコミットしています。
ソースコード中で本質的でない記述を一部省略しているので、実行コードを作成する場合は、必要に応じて適宜GitHub上のソースコードも参照してください。

|br|

.. _section-cloudformation-s3-sample-label:

S3スタック構築テンプレート
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

S3は `クラウドネイティブ基本第25回 <https://news.mynavi.jp/itsearch/article/devsoft/4597>`_ で実施した要領と同等のものを構築します。
S3をCloudFormationで構築する場合、リソースタイプが、 `AWS::S3::Bucket <https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-properties-s3-bucket.html>`_
であるリソース定義が必要です。プロパティとして設定可能な属性は、上記リンク先の通りですが、加えて、S3を商用環境、ステージング環境、開発環境という3つのパターンに分けて作成するようにします。テンプレートのサンプルは以下の通りです。

|br|

.. sourcecode:: none

   AWSTemplateFormatVersion: '2010-09-09'

   // omit

   Mappings:
     BucketDefinitionMap:                                                                               #(A)
       Production:
         "BucketName" : debugroom-mynavi-sample-cloudformation-bucket-production
         "AccessControl" : "Private"
         "BlockPublicAcls" : True
         "BlockPublicPolicy" : True
         "IgnorePublicAcls" : True
         "RestrictPublicBuckets" : True

    // omit

   Parameters:
     EnvType:                                                                                           #(B)
       Description: Which environments to deploy your service.
       Type: String
       AllowedValues: ["Dev", "Staging", "Production"]
       Default: Dev

   Resources:
     S3Bucket:                                                                                          #(C)
       Type: AWS::S3::Bucket
       Properties:
         BucketName: !FindInMap [BucketDefinitionMap, !Ref EnvType, BucketName]                         #(D)
         AccessControl: !FindInMap [BucketDefinitionMap, !Ref EnvType, AccessControl]
         PublicAccessBlockConfiguration:
           BlockPublicAcls: !FindInMap [BucketDefinitionMap, !Ref EnvType, BlockPublicAcls]
           BlockPublicPolicy: !FindInMap [BucketDefinitionMap, !Ref EnvType, BlockPublicPolicy]
           IgnorePublicAcls: !FindInMap [BucketDefinitionMap, !Ref EnvType, IgnorePublicAcls]
           RestrictPublicBuckets: !FindInMap [BucketDefinitionMap, !Ref EnvType, RestrictPublicBuckets]

   Outputs:
     S3Bucket:                                                                                          #(E)
       Description: Mynavi S3 bucket name
       Value: !Ref S3Bucket
       Export:
         Name: !Sub MynaviSampleS3Bucket-${EnvType}

|br|

S3のテンプレートの記述の基本となるポイントは(A)〜(G)の通りです。

|br|

.. list-table:: S3のCloudFormationテンプレート記述のポイント
   :widths: 1, 9

   * - 記述
     - 説明

   * - (A)
     - Mappingsとして、EnvTypeパラメータの値に応じて、3つの環境ごとの論理名とパラメータを定義します。Mappings定義方法の詳細は :ref:`section-cloudformation-alb-sample-label` と同様なので適宜参照してください。

   * - (B)
     - S3を商用環境、ステージング環境、開発環境用に分けるよう、EnvTypeパラメータとして指定可能にします。このパラメータに応じて、Mappingsに定義したパラメータを切り替え、リソース定義を分けて作成できるようにします。

   * - (C)
     - S3のリソース定義を行います。定義するプロパティの詳細は  `AWS::S3::Bucket <https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-properties-s3-bucket.html>`_ を参照してください。

   * - (D)
     - 環境によって異なる値を定義したい箇所は、EnvTypeパラメータを引数としたFindInMap関数でデータ取得します。

   * - (E)
     - S3のバケット名をOutputs出力します。

|br|

作成したテンプレートに対して、ヘルパースクリプトを以下のように、スタック名とテンプレートパスを変更して実行します。

|br|

.. sourcecode:: bash

   #!/usr/bin/env bash

   stack_name="mynavi-sample-s3"
   template_path="sample-s3-cfn.yml"

   parameters="EnvType=Dev"

   aws cloudformation deploy --stack-name ${stack_name} --template-file ${template_path} --parameter-overrides ${parameters} --capabilities CAPABILITY_IAM

|br|

実行が正常に終了すると、S3のバケットが作成されます。

|br|

.. figure:: img/automation_infra_devops_cloudformation/management_console_cloudformation_stack_s3.png

|br|

続いて、SQSのスタックテンプレートを実装します。

|br|

.. _section-cloudformation-sqs-sample-label:

SQSスタック構築テンプレート
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

SQSは `クラウドネイティブ基本第28回 <https://news.mynavi.jp/itsearch/article/devsoft/4656>`_ で実施した要領と同等のものを構築します。
S3をCloudFormationで構築する場合、リソースタイプが、 `AWS::SQS::Queue <https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-properties-sqs-queues.html>`_
であるリソース定義が必要です。プロパティとして設定可能な属性は、各リンク先の通りですが、加えて、SQSを商用環境、ステージング環境、開発環境という3つのパターンに分けて作成するようにします。

|br|

.. sourcecode:: none

   AWSTemplateFormatVersion: '2010-09-09'

   // omit

   Mappings:
     SQSDefinitionMap:                                                                                              #(A)
       Production:
         "QueueName" : mynavi-sample-queue-production
         "VisibilityTimeout": 30
         "DelaySeconds" : 5
         "MaximumMessageSize" : 26144
         "MessageRetentionPeriod" : 345600
         "ReceiveMessageWaitTimeSeconds" : 0
       // omit

   Parameters:
     EnvType:                                                                                                       #(B)
       Description: Which environments to deploy your service.
       Type: String
       AllowedValues: ["Dev", "Staging", "Production"]
       Default: Dev

   Resources:
     SQSSampleQueue:                                                                                                #(C)
       Type: AWS::SQS::Queue
       Properties:
         QueueName: !FindInMap [SQSDefinitionMap, !Ref EnvType, QueueName]                                          #(D)
         VisibilityTimeout: !FindInMap [SQSDefinitionMap, !Ref EnvType, VisibilityTimeout]
         DelaySeconds:  !FindInMap [SQSDefinitionMap, !Ref EnvType, DelaySeconds]
         MaximumMessageSize: !FindInMap [SQSDefinitionMap, !Ref EnvType, MaximumMessageSize]
         MessageRetentionPeriod:  !FindInMap [SQSDefinitionMap, !Ref EnvType, MessageRetentionPeriod]
         ReceiveMessageWaitTimeSeconds:  !FindInMap [SQSDefinitionMap, !Ref EnvType, ReceiveMessageWaitTimeSeconds]

   Outputs:
     SQSServiceEndpoint:                                                                                            #(E)
       Description: SQS service endipoint
       Value: !Sub https://sqs.${AWS::Region}.amazonaws.com
       Export:
         Name: !Sub MynaviSampleSQS-${EnvType}-ServiceEndpoint

     SQSServiceRegion:                                                                                              #(F)
       Description: SQS service region
       Value: !Sub ${AWS::Region}
       Export:
         Name: !Sub MynaviSampleSQS-${EnvType}-Region

     SQSSampleQueue:                                                                                                #(G)
       Description: SQS sample queue
       Value: !Ref SQSSampleQueue
       Export:
         Name: !Sub MynaviSampleSQS-${EnvType}


|br|

SQSのテンプレートの記述の基本となるポイントは(A)〜(G)の通りです。

|br|

.. list-table:: SQSのCloudFormationテンプレート記述のポイント
   :widths: 1, 9

   * - 記述
     - 説明

   * - (A)
     - Mappingsとして、EnvTypeパラメータの値に応じて、3つの環境ごとの論理名とパラメータを定義します。Mappings定義方法の詳細は :ref:`section-cloudformation-alb-sample-label` と同様なので適宜参照してください。

   * - (B)
     - SQSを商用環境、ステージング環境、開発環境用に分けるよう、EnvTypeパラメータとして指定可能にします。このパラメータに応じて、Mappingsに定義したパラメータを切り替え、リソース定義を分けて作成できるようにします。

   * - (C)
     - SQSのリソース定義を行います。定義するプロパティの詳細は  `AWS::SQS::Queue <https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-properties-sqs-queues.html>`_ を参照してください。

   * - (D)
     - 環境によって異なる値を定義したい箇所は、EnvTypeパラメータを引数としたFindInMap関数でデータ取得します。

   * - (E)
     - SQSのサービスエンドポイントをOutputs出力します。

   * - (F)
     - SQSを構築したリージョンをOutputs出力します。

   * - (G)
     - SQSのキュー名をOutputs出力します。

|br|

作成したテンプレートに対して、ヘルパースクリプトを以下のように、スタック名とテンプレートパスを変更して実行します。

|br|

.. sourcecode:: bash

   #!/usr/bin/env bash

   stack_name="mynavi-sample-sqs"
   template_path="sample-sqs-cfn.yml"

   parameters="EnvType=Dev"

   aws cloudformation deploy --stack-name ${stack_name} --template-file ${template_path} --parameter-overrides ${parameters} --capabilities CAPABILITY_IAM

|br|

実行が正常に終了すると、SQSのキューが作成されます。

|br|

.. figure:: img/automation_infra_devops_cloudformation/management_console_cloudformation_stack_sqs.png

|br|

今回はS3、SQSを構築するCloudFormationテンプレートを実装しました。次回は、これまで作成してきたテンプレートを一括で実行できるよう親子関係でネスト化する手順を解説します。

|br|

著者紹介
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

川畑 光平(KAWABATA Kohei) - NTTデータ 課長代理

.. figure:: img/automation_infra_devops_overview/pic_image01.jpg


金融機関システム業務アプリケーション開発・システム基盤担当を経て、現在はソフトウェア開発自動化関連の研究開発・推進に従事。

Red Hat Certified Engineer、Pivotal Certified Spring Professional、AWS Certified Solutions Architect Professional等の資格を持ち、アプリケーション基盤・クラウドなど様々な開発プロジェクト支援にも携わる。

`2019 APN AWS Top Engineers & Ambassadors <https://aws.amazon.com/jp/blogs/psa/japan-apn-ambassador-2019/>`_ 選出。
