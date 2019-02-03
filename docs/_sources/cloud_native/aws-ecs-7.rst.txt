.. include:: ../module.txt

.. _section-cloud-native-ecs-label-7:

AWSで作るクラウドネイティブアプリケーションの基本
========================================================================================

.. _section-cloud-native-ecs-7th-label:

第2回 AWS ECS上に構築するSpringアプリケーション(7)
----------------------------------------------------------------------------------------

|br|

クラウド時代が到来し、ますます広がりを見せつつあるコンテナ技術。第2回は、AWS ECS上でSpringアプリケーションを構築する方法を説明します。本稿は以下のステップに沿って、解説しています。

#. VPC(Virtual Private Cloud)環境の構築
#. アプリケーションロードバランサ(ALB)の作成
#. Springを使用したコンテナアプリケーションの実装方法
#. Dockerコンテナの作成・DockerHubへのプッシュ
#. ECSクラスタの作成
#. ECSタスクの定義
#. ECSサービスの実行

前回の記事「 :ref:`section-cloud-native-ecs-6th-label` 」までに、以下イメージの構成に沿って、VPC環境・ALBを構築した後、ECSコンテナ上で動くパブリック・プライベートサブネット用２種類のアプリケーションを作成し、
Dockerコンテナイメージにしてレジストリにプッシュしました。そして、コンテナを実行するためのECSクラスタを作成し、ECSコンテナの定義まで完了しました。第二回の最後となる今回は「7. ECSサービスの実行」です。

|br|

.. figure:: img/aws-ecs/ecs-architecture.png
   :scale: 100%

|br|

(7)ECSサービスの実行
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

ECSサービス実行では、実際にECSクラスタ上にコンテナをLaunchします。「ECS」サービスから「ECSクラスタ」メニューを選択し、コンテナを実行するクラスタを選択します。

|br|

.. figure:: img/aws-ecs/management-console-ecs-create-service-1.png
   :scale: 100%

|br|

「サービス」タブにある「作成」ボタンを押下します。

|br|

.. figure:: img/aws-ecs/management-console-ecs-create-service-2.png
   :scale: 100%

|br|


以下の要領に従って、ECSサービス定義の設定を行います。

.. list-table:: ECSサービス定義の入力項目
   :widths: 2, 8

   * - 入力項目
     - 説明

   * - 起動タイプ
     - FargateかEC2起動型か選択します。ここでは、EC2起動型を選択します。

   * - タスク定義
     - サービスとして実行するタスク定義を設定します。前回設定したタスク定義を選択します。

   * - クラスタ
     - サービスをLaunchするクラスタを選択します。パブリック・プライベートに対応するクラスタを選択します。

   * - サービス名
     - サービスの名前として設定する任意の名称を入力します。

   * - サービスタイプ
     - サービスタイプを選択します。REPLICAは、クラスタ全体でタスクの数として指定した数のコンテナを実行するオプションで、DAEMONはECSクラスタのインスタンスの増減に合わせて実行コンテナを増減させるオプションです。クラスタインスタンス1台ごとに1つの実行コンテナを維持します。詳細は、 `サービススケジューラの概念 <https://docs.aws.amazon.com/ja_jp/AmazonECS/latest/developerguide/ecs_services.html#service_scheduler>`_ を参照してください。

   * - タスクの数
     - クラスタで実行するコンテナ数を設定します。

   * - 最小ヘルス率
     -

   * - 最大率
     -

   * - デプロイメントタイプ
     -

   * - 配置テンプレート
     -

   * - ヘルスチェックの猶予時間
     -

   * - ELBタイプ
     -

   * - サービス用IAMロールの選択
     -

   * - ELB名
     -

   * - リスナーポート
     -

   * - リスナープロトコル
     -

   * - ターゲットグループ名
     -

   * - ターゲットグループのプロトコル
     -

   * - パスパターン
     -

   * - ヘルスチェックパス
     -

   * - サービス検出統合の有効化
     -

   * - ServiceAutoScaling
     -

|br|

.. figure:: img/aws-ecs/management-console-ecs-create-service-3.png
   :scale: 100%

.. figure:: img/aws-ecs/management-console-ecs-create-service-4.png
   :scale: 100%

.. figure:: img/aws-ecs/management-console-ecs-create-service-5.png
   :scale: 100%

.. figure:: img/aws-ecs/management-console-ecs-create-service-6.png
   :scale: 100%

.. figure:: img/aws-ecs/management-console-ecs-create-service-7.png
   :scale: 100%

|br|

入力項目を確認し、サービスの作成ボタンを押下します。

|br|

.. figure:: img/aws-ecs/management-console-ecs-create-service-8.png
   :scale: 100%

|br|

同様にプライベートサブネットのアプリケーションコンテナもサービスを実行します。パブリックサブネットのALBのDNSにBFFアプリケーションのパスを加え、アプリケーションが正しく実行されるか確認します。

BFFアプリケーションのURL：　http://mynavi-sample-alb-public-722388091.ap-northeast-1.elb.amazonaws.com/backend-for-frontend/index.html

|br|

.. figure:: img/aws-ecs/ecs-bff-application.png
   :scale: 100%

|br|

.. note:: 正しくコンテナが実行されない場合、ECSクラスタにSSHログインし、docker ps -aすることでコンテナの実行を確認できます。また、コンテナ実行時のログは/var/log/ecs/ecs-agent.log_XXXXで確認できます。トラブルシューティングはこちらを参照してください。

|br|

まとめ
------------------------------------------------------------------

|br|

AWS ECSでは、ネットワーク構成やロードバランサ、コンテナの定義など様々なことを考慮する必要がありますが、クラウド環境で可用性・セキュリティを保ちつつ、堅牢なSpringアプリケーションを構築することができます。

|br|

著者紹介
------------------------------------------------------------------

川畑 光平(KAWABATA Kohei)

.. figure:: img/aws-lambda-and-api-gateway/pic_image01.jpg
   :scale: 100%

某システムインテグレータにて、金融機関システム業務アプリケーション開発・システム基盤担当を経て、現在はソフトウェア開発自動化関連の研究開発・推進に従事。

Red Hat Certified Engineer、Pivotal Certified Spring Professional、AWS Certified Solutions Architect Professional等の資格を持ち、アプリケーション基盤・クラウドなど様々な開発プロジェクト支援にも携わる。

本連載記事の内容に対するご意見・ご質問は `Facebook <https://www.facebook.com/kohei.kawabata.5>`_ まで。
