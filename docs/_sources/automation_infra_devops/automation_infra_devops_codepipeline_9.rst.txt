.. include:: ../module.txt

.. _section-automation-infra-devops-codepipeline-9-label:

基盤・デプロイ自動化実践
==================================================================

マイクロサービスアーキテクチャの基盤・デプロイ自動化
-------------------------------------------------------------------------------------------------------------------------------------

.. _section-codepipeline-setting-codepipeline-deploy-label:

(8)プロダクション環境へデプロイするパイプラインの構築
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

本連載では、マイクロサービスアーキテクチャでの継続的デリバリ(Continuous Delivery:CD)を以下のようなパイプラインで実現していきます。

|br|

.. figure:: img/automation_infra_devops_codepipeline/pipeline.png


|br|

前回は(7)プロダクション環境向けのアプリケーションのコンテナイメージをビルドしてDockerHubへプッシュするパイプラインを構築しました。
続く今回は、そのコンテナをプロダクション環境へデプロイするパイプラインを構築します。

|br|

8. プロダクション環境へのリリース

* (8-1) ECSクラスタ上にアプリケーションコンテナを実行させる命令を発出します(BackendおよびBFFは並列実行)。
* (8-2) ECSエージェントが(7-2)でプッシュしたコンテナを、各々ECSクラスタ上にプルします。
* (8-3) 各々ECSクラスタ上でアプリケーションコンテナが実行されます。

|br|

.. figure:: img/automation_infra_devops_codepipeline/pipeline-8.png


|br|

.. _section-codepipeline-setting-codepipeline-production-environment-label:

事前準備：アプリケーションのプロダクション環境の構築
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

パイプラインの設定を行う前に、 :ref:`section-codepipeline-setting-codepipeline-backend-staging-environment-label` および、 :ref:`section-codepipeline-setting-bff-staging-environment-label` と同様、
プロダクション環境のアプリケーションロードバランサ、ECSクラスタ、タスク定義、サービスの構築をBackendとBFF両方で行います。

|br|

再掲になりますが、ステージング環境の構築と同様、プロダクション環境でも、:ref:`section-codepipeline-setting-codepipeline-backend-staging-environment-label` の表の記載してある、
タスクのメモリ、CPU、コンテナ名、イメージ設定、アプリケーションロードバランサのDNSの環境変数への設定で同様の留意が必要です。

|br|

.. list-table::
   :widths: 2, 3, 5

   * - 設定箇所
     - 設定内容
     - 説明

   * - ECSタスク定義：コンテナの追加
     - 環境：環境変数
     - バックエンドマイクロサービスへパスルーティングするアプリケーションロードバランサのDNSをSERVICE_DNS環境変数として設定します。プロダクション環境での具体的な設定値は、SystemsManagerParameterStoreで定義した"SERVICE_DNS_PRODUCTION"から取得するよう、ValueFromフィールドを設定します。

|br|

.. figure:: img/automation_infra_devops_codepipeline/management_console_ecs_task_definition_bff_production_1.png


|br|

|br|

.. figure:: img/automation_infra_devops_codepipeline/management_console_ecs_task_definition_bff_production_2.png


|br|

|br|

.. figure:: img/automation_infra_devops_codepipeline/management_console_ecs_task_definition_bff_production_3.png


|br|

.. _section-codepipeline-setting-ssm-definition-deploy-production-label:


AWS Sysmtems Managers Parameter Storeでの環境変数の定義
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

CodePipelineの設定を行う前に、前節のECSで使用する環境変数を定義しておきます。設定の要領は、 :ref:`section-codebuild-setting-sms-label` と同様です。以下のパラメータを定義します。

* "SERVICE_DNS_PRODUCTION"：前節で作成したアプリケーションロードバランサーのDNS

|br|

.. _section-codebuild-setting-pipeline-deploy-production-label:

プロダクション環境デプロイのパイプラインの作成
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

これまで作成してきたCodePipelineを編集して、プロダクション環境へのデプロイパイプラインを設定します。
AWSコンソールの「CodePipeline」サービスを選択し、パイプラインを選択して、「編集する」ボタンを押下します。

|br|

.. figure:: img/automation_infra_devops_codepipeline/management_console_codepipeline_edit_project_31_build_bff_staging.png


|br|

前回作成した、プロダクション環境のコンテナを作成したステージにデプロイアクションを追加します。「ステージを編集する」ボタンを押下し、新たな「アクショングループ」を追加します。

|br|

.. figure:: img/automation_infra_devops_codepipeline/management_console_codepipeline_edit_project_64_deploy_backend_production.png


|br|
|br|

.. figure:: img/automation_infra_devops_codepipeline/management_console_codepipeline_edit_project_65_deploy_backend_production.png


|br|

まず、Backendのコンテナイメージをデプロイするための設定を追加します。以下の要領でアクションを設定し、「完了」ボタンを押下します。

* アクション名：任意のアクション名を追加します。
* アクションプロバイダー：「Amazon ECS」を選択します。
* リージョン：プロダクション環境があるリージョンを選択します。
* 入力アーティファクト：前回のコンテナをビルドするパイプラインで出力アーティファクトとなっている「BuildArtifact-backend-production」を選択します。なお、この実体はimagedefinition.jsonであり、S3に保存されています。
* クラスタ名： 前節で作成したECSクラスタを選択します。
* サービス名： 前節で作成したECSサービスを選択します。
* イメージ定義ファイル：前回のパイプライン処理でbuildspec.ymlで出力したimagedefition.jsonを設定します。

|br|

.. figure:: img/automation_infra_devops_codepipeline/management_console_codepipeline_edit_project_66_deploy_backend_production.png


|br|

同様に、BFFのコンテナイメージを並列実行してデプロイするための設定を追加します。上記で作成したアクションの横にある「アクションの追加」を選んだのち、同様の要領でアクションを設定します。

|br|

.. figure:: img/automation_infra_devops_codepipeline/management_console_codepipeline_edit_project_67_deploy_bff_production.png


|br|

|br|

.. figure:: img/automation_infra_devops_codepipeline/management_console_codepipeline_edit_project_68_deploy_bff_production.png


|br|

設定完了後、「変更をリリースする」ボタンを押下し、デプロイが問題なく完了するか確認します。

|br|

.. figure:: img/automation_infra_devops_codepipeline/management_console_codepipeline_edit_project_69_deploy_bff_production.png


|br|


.. _section-codepipeline-conclusion-label:

マイクロサービスにおける継続的デリバリーのまとめ
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

以上、ここまでの連載で、E2Eテストを含め、以下のような継続的デリバリーをCodePipelineを使って構築してきました。

#. バックエンドのマイクロサービスのコンテナイメージのビルド、DockerHubへのプッシュ
#. マイクロサービスコンテナイメージをプロダクションとほぼ同等のステージング環境へデプロイ
#. Webアプリケーション(BFF)のE2Eテストとして、Seleniumテストコード実行
#. Webアプリケーション(BFF)のコンテナイメージのビルド、DockerHubへのプッシュ
#. Webアプリケーション(BFF)のステージング環境へデプロイ
#. パフォーマンステストやセキュリティテスト、受入等のその他テスト後の管理者によるリリース承認
#. ステージング環境でテスト済みのコンテナイメージをリリース用にタグ付け、DockerHubへのプッシュ
#. プロダクション環境へリリース

なお、今回はトピックスには盛り込んでいませんが、デプロイのオプションとしてECSブルーグリーンデプロイメントを選択することで、一部のアプリケーションコンテナのみを新しいバージョンで試しながらリリースする手段も実現できます。

クラウドベースのCI/CDサービスであるCodeBuildやCodePipelineを活用することにより、複数のマイクロサービスの並列ビルド・デプロイや
マルチOSやブラウザでのE2Eテストの並列実行などを容易に実現できることがお分りいただけたのではないでしょうか。
マイクロサービスにおいては複数のサービスが連携し、各サービスごとに独立してビルドやテスト、デプロイなどを行うかたちになります。
継続的インテグレーションや継続的デリバリーの仕組みもこうしたスケーラビリティに優れたCI/CDサービスを利用することで、開発におけるアジリティ・耐障害性の向上といった、マイクロサービスアーキテクチャのメリットをより享受することができます。
また、SystemsManagerParameterStoreなどを活用して、ステージング環境でテストしたアプリケーションのコンテナイメージを環境変数だけを差し替えて、そのままプロダクション環境へ持っていくことで、
プロダクション向けのビルドが省略でき、本番環境起因のバグリスクを極力下げながら、リリースも迅速に行うことが可能です。


|br|

次回は、こうした環境の一括構築を行うCodeStarを使ったCI/CD環境の構築について概説していきます。

|br|

著者紹介
------------------------------------------------------------------

川畑 光平(KAWABATA Kohei) - NTTデータ 課長代理

.. figure:: img/automation_infra_devops_overview/pic_image01.jpg


金融機関システム業務アプリケーション開発・システム基盤担当を経て、現在はソフトウェア開発自動化関連の研究開発・推進に従事。

Red Hat Certified Engineer、Pivotal Certified Spring Professional、AWS Certified Solutions Architect Professional等の資格を持ち、アプリケーション基盤・クラウドなど様々な開発プロジェクト支援にも携わる。

`2019 APN AWS Top Engineers & Ambassadors <https://aws.amazon.com/jp/blogs/psa/japan-apn-ambassador-2019/>`_ 選出。
