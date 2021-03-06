.. include:: ../module.txt

.. _section-cloud-native-ecs-7th-label:

【第10回】ECSコンテナAP開発(7)ECSサービス設定
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
Dockerコンテナイメージを作成してレジストリにプッシュしました。そして、コンテナを実行するためのECSクラスタを作成し、ECSコンテナの定義まで完了しました。第二回の最後となる今回は「7. ECSサービスの実行」です。

|br|

.. figure:: img/aws-ecs/ecs-architecture.png


|br|

.. _section-cloud-native-ecs-create-service-label:

(7)ECSサービスの実行
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

ECSタスク定義と同様、ネットワークインターフェースをコンテナにアタッチするなどのサービス実行に必要な権限を持つロールが必要になります。　
「 :ref:`section-cloud-native-ecs-define-task-label` 」に記載するIAMロール作成と同様の手順で、AmazonECSServiceRolePolicyをアタッチしたIAMロールを作成しておいてください。
なお、ECSを初回起動する場合には、以降の手順を実施していく中で、「AWSServiceRoleForECS」という名前で自動作成してくれますので、そのやり方で実行する場合は、スキップしてもかまいません。

ECSサービス実行では、ECSクラスタ上にコンテナを起動します。「ECS」サービスから「ECSクラスタ」メニューを選択し、コンテナを実行するクラスタを選択します。

|br|

.. figure:: img/aws-ecs/management-console-ecs-create-service-1.png


|br|

「サービス」タブにある「作成」ボタンを押下します。

|br|

.. figure:: img/aws-ecs/management-console-ecs-create-service-2.png


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
     - コンテナの最小実行数の割合を設定します。

   * - 最大率
     - コンテナの最大実行数の割合を設定します。

   * - デプロイメントタイプ
     - コンテナをアップデートする場合の戦略を選択します。ローリングアップデートするかAWS CodeDeployを使用したブルーグリーンデプロイメントを選択できます。詳細は「 `Amazon ECS デプロイタイプ <https://docs.aws.amazon.com/ja_jp/AmazonECS/latest/developerguide/deployment-types.html>`_ 」を参照してください。

   * - 配置テンプレート
     - コンテナを配置する戦略を選択します。詳細は「 `Amazon ECS タスク配置戦略 <https://docs.aws.amazon.com/ja_jp/AmazonECS/latest/developerguide/task-placement-strategies.html>`_ 」を参照してください。

   * - ヘルスチェックの猶予時間
     - ECSコンテナを起動してからのヘルスチェックの猶予時間を設定します。Springアプリケーションを起動するのに数秒〜数十秒を要するので、その辺を考慮した値を設定しましょう。

   * - ELBタイプ
     - ECSコンテナに処理を分散させるロードバランサーを指定します。ここではALBを指定してください。

   * - サービス用IAMロールの選択
     - ECSサービス実行用のロールを設定してください。前述した通り、初回ECSを起動する場合は、新しくロールを作成するを選択すると、「AWSServiceRoleForECS」が作成されます。なお、詳細は「 `Amazon ECS のサービスにリンクされたロールの使用 <https://docs.aws.amazon.com/ja_jp/AmazonECS/latest/developerguide/using-service-linked-roles.html>`_ 」を参考にしてください。

   * - ELB名
     - 「 :ref:`section-cloud-native-ecs-create-alb-label` 」で作成したALBを選択します。コンテナアプリケーションに合わせて、パブリック・プライベート各々サブネットと対応したALBを選択してください。

   * - リスナーポート
     - ALBを作成時に設定していたポートを選択されます。

   * - リスナープロトコル
     - ALBを作成時に設定していたプロトコルを選択されます。

   * - ターゲットグループ名
     - ALBに設定されているターゲットグループを選択します。複数サービスがある場合はここでターゲットグループをコンテナに関連づけることになります。

   * - ターゲットグループのプロトコル
     - ALBを作成時に設定していたプロトコルがされます。

   * - パスパターン
     - 選択したターゲットグループにより自動指定されます。

   * - ヘルスチェックパス
     - 選択したターゲットグループにより自動設定されます。

   * - サービス検出統合の有効化
     - 2018年9月に東京リージョンで追加されたRoute53を使ったサービス検出・ディスカバリー機能を有効化するオプションです。詳細は「 `オプションサービス検出を使用するようにサービスを設定する <https://docs.aws.amazon.com/ja_jp/AmazonECS/latest/userguide/service-configure-servicediscovery.html>`_ 」に記述がありますが、今回はALBのパスルーティングを使用して、コンテナを識別・負荷分散するので、設定をオフにしてください。

   * - ServiceAutoScaling
     - AutoScaling機能を有効化する場合、オプションを設定します。

|br|

.. figure:: img/aws-ecs/management-console-ecs-create-service-3.png


.. figure:: img/aws-ecs/management-console-ecs-create-service-4.png


.. figure:: img/aws-ecs/management-console-ecs-create-service-5.png


.. figure:: img/aws-ecs/management-console-ecs-create-service-6.png


.. figure:: img/aws-ecs/management-console-ecs-create-service-7.png


|br|

入力項目を確認し、サービスの作成ボタンを押下します。

|br|

.. figure:: img/aws-ecs/management-console-ecs-create-service-8.png


|br|

同様にプライベートサブネットのアプリケーションコンテナもサービスを実行します。パブリックサブネットのALBのDNSにBFFアプリケーションのパスを加え、アプリケーションが正しく実行されるか確認します。xxxxxxの部分は自身の環境のALBのDNS名に合わせてアクセスしてください。

BFFアプリケーションのURL：　http://xxxxxxxx.ap-northeast-1.elb.amazonaws.com/backend-for-frontend/index.html

|br|

.. figure:: img/aws-ecs/ecs-bff-application.png


|br|

.. note:: 正しくコンテナが実行されない場合、ECSクラスタにSSHログインし、docker ps -aすることでコンテナの実行を確認できます。また、コンテナ実行時のログは/var/log/ecs/ecs-agent.log_XXXXで確認できます。トラブルシューティングはこうしたログを参照してください。

|br|

まとめ
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

ネットワーク構成やロードバランサ、コンテナの定義など様々なことを考慮する必要がありますが、AWS ECSを使って、クラウド環境でセキュリティを保ちつつ、可用性の高いアプリケーションを構築することができます。
また、「 :ref:`section-cloud-native-ecs-spring-app-implementation-label` 」の様にSpringアプリケーションを実装し、別のECSコンテナアプリケーションを直接呼び出すのではなく、
ALBのパスベースのルーティング機能を使い、サービスを呼び出す構成にしておくことで、以下のようなメリットを享受することができます。

* 呼び出したいアプリケーションのコンテナのIPやポートを意識しなくて済む
* ヘルスチェックによるコンテナアプリケーション監視が行える
* コンテナのオートリカバリを実現できる(ヘルスチェックでNGだとコンテナを再起動)
* 複数のコンテナが実行されていた場合に処理を分散できる
* CloudWatchによるリソース監視ができる。

こうした構成はエンタープライズアプリケーションをクラウドネイティブ化する場合に十分有用なアーキテクチャです。

|br|

著者紹介
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

川畑 光平(KAWABATA Kohei)

.. figure:: img/aws-lambda-and-api-gateway/pic_image01.jpg


某システムインテグレータにて、金融機関システム業務アプリケーション開発・システム基盤担当を経て、現在はソフトウェア開発自動化関連の研究開発・推進に従事。

Red Hat Certified Engineer、Pivotal Certified Spring Professional、AWS Certified Solutions Architect Professional等の資格を持ち、アプリケーション基盤・クラウドなど様々な開発プロジェクト支援にも携わる。

本連載記事の内容に対するご意見・ご質問は `Facebook <https://www.facebook.com/kohei.kawabata.5>`_ まで。
