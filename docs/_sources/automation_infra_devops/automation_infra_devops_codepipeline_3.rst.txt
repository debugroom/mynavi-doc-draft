.. include:: ../module.txt

.. _section-automation-infra-devops-codepipeline-3-label:

基盤・デプロイ自動化実践
==================================================================

マイクロサービスアーキテクチャの基盤・デプロイ自動化
-------------------------------------------------------------------------------------------------------------------------------------

.. _section-codepipeline-setting-codepipeline-deploy-backend-staging-label:

(2)バックエンドマイクロサービスのコンテナイメージをデプロイするパイプラインの構築
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

本連載では、マイクロサービスアーキテクチャでの継続的デリバリ(Continuous Delivery:CD)を以下のようなパイプラインで実現していきます。

|br|

.. figure:: img/automation_infra_devops_codepipeline/pipeline.png


|br|

前回は(1)バックエンドのマイクロサービスアプリケーションをビルドし、DockerHubへプッシュするパイプライン処理を構築しました。今回はプッシュしたコンテナイメージをECSクラスタ上にデプロイするパイプラインを構築します。

|br|

2. マイクロサービスコンテナイメージをプロダクションとほぼ同等のステージング環境へデプロイ

* (2-1) ECSクラスタ上にアプリケーションコンテナを実行させる命令を発出します。
* (2-2) ECSエージェントが(1-2)でプッシュしたコンテナをECSクラスタ上にプルします。
* (2-3) ECSクラスタ上でアプリケーションコンテナが実行されます。

|br|

.. figure:: img/automation_infra_devops_codepipeline/pipeline-2.png


|br|

.. _section-codepipeline-setting-codepipeline-backend-staging-environment-label:

事前準備：マイクロサービスのステージング環境の構築
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

パイプラインの設定を行う前に、マイクロサービスをデプロイする先であるステージング環境のアプリケーションロードバランサ、ECSクラスタ、タスク定義、サービスの構築を行います。
作成方法の要領は、以下の手順と同様です。

* アプリケーションロードバランサ： `AWS ECS上に構築するSpringアプリケーション(2) <https://news.mynavi.jp/itsearch/article/devsoft/4359>`_
* ECSクラスタ： `AWS ECS上に構築するSpringアプリケーション(5) <https://news.mynavi.jp/itsearch/article/devsoft/4405>`_
* ECSタスク定義： `AWS ECS上に構築するSpringアプリケーション(6) <https://news.mynavi.jp/itsearch/article/devsoft/4408>`_
* ECSサービス実行： `AWS ECS上に構築するSpringアプリケーション(7) <https://news.mynavi.jp/itsearch/article/devsoft/4416>`_

|br|

ただし、この事前準備では、以下の設定に留意が必要です。

|br|

.. list-table::
   :widths: 2, 3, 5

   * - 設定箇所
     - 設定内容
     - 説明

   * - 基本設定：ECSタスク定義
     - タスクサイズ：タスクメモリ
     - SpringBootアプリケーションであれば1024MB以上を目処に設定した方がベターです。起動時間がかかり、ヘルスチェックがNGになる可能性があります。

   * - 基本設定：ECSタスク定義
     - タスクサイズ：タスクCPU
     - 0.25〜0.5 vcpuを目処に設定します。ECSクラスタのスペックにもよりますが、小さすぎると上記のタスクメモリの設定同様、起動に時間がかかりヘルスチェックがNGになる可能性がありますが、逆に大きすぎるとパイプライン処理のコンテナ再起動時にリソースが足りず、エラーとなる可能性があります。

   * - 基本設定：ECSタスク定義
     - コンテナの編集：コンテナ名
     - 前回作成したbuildspec.ymlのアーティファクトで指定したimagedefinitions.jsonのname属性と一致した名前を設定する必要があります。

   * - 基本設定：ECSタスク定義
     - コンテナの編集：イメージ
     - buildspec.yml及びSystemsManagerParameterStoreで環境変数として設定した値のイメージ名とバージョンを指定します。

|br|

.. figure:: img/automation_infra_devops_codepipeline/management_console_ecs_task_definition_backend_staging_1.png


|br|

.. figure:: img/automation_infra_devops_codepipeline/management_console_ecs_task_definition_backend_staging_2.png


|br|

.. _section-codepipeline-setting-pipeline-deploy-backend-staging-label:

マイクロサービスのステージング環境へのデプロイパイプラインの設定
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

前回作成したベースのパイプラインに、ステージング環境として構築したECSクラスタにマイクロサービスをデプロイするパイプラインを追加します。
CodePipelineサービスメニューから、前回作成したパイプラインを選択し、「編集する」ボタンを押下して末尾にある「ステージを追加する」ボタンを押下します。

|br|

.. figure:: img/automation_infra_devops_codepipeline/management_console_codepipeline_edit_project_13_deploy_backend_staging.png


|br|

ステージングへデプロイするためのステージを追加します。

|br|

.. figure:: img/automation_infra_devops_codepipeline/management_console_codepipeline_edit_project_14_deploy_backend_staging.png


|br|

「アクションを追加する」ボタンを押下して、以下の要領でステージングデプロイアクションを設定します。

* アクション名：100文字内の任意のアクション名を設定
* アクションプロバイダー：Amazon ECSを選択
* リージョン：VPC及びECSクラスタを構築したリージョンを選択
* 入力アーティファクト：前回、作成したマイクロサービスのビルドパイプラインのアウトプットアーティファクトを指定()デフォルトでは、BuildArtifact)。
* クラスタ名：前節で構築したECSクラスタを設定します
* サービス名：前節で起動した、同じコンテナイメージで起動しているECSサービスを選択します。
* イメージ定義ファイル：前回のビルドパイプライン処理でアウトプットしたimagedefinitions.jsonのファイル名を設定します。

|br|

.. note:: なお、入力アーティファクトは、前回のパイプラインの出力アーティファクトですが、S3上に実体となるファイルが保存されています。

|br|

.. figure:: img/automation_infra_devops_codepipeline/management_console_codepipeline_edit_project_15_deploy_backend_staging.png


|br|

作成が完了したら、「変更をリリースする」ボタンを押下して、パイプラインを実行しなおしてみましょう。グリーンで表示されれば正常完了です。
ECSタスクのリビジョンがあがり、ECSサービスのイベントログでは、コネクションドレイニングされて、実行コンテナが入れ替わったことがわかります。

|br|

.. figure:: img/automation_infra_devops_codepipeline/management_console_codepipeline_edit_project_16_deploy_backend_staging.png


|br|

|br|

.. figure:: img/automation_infra_devops_codepipeline/management_console_codepipeline_edit_project_17_deploy_backend_staging.png



|br|

.. figure:: img/automation_infra_devops_codepipeline/management_console_codepipeline_edit_project_18_deploy_backend_staging.png


|br|

これでBakcendサービスのコンテナイメージをステージング環境へデプロイするパイプラインが完成し、プッシュしたコンテナがECSクラスタ上で起動しました。
次回以降は、Web(BFF)アプリケーションをビルドし、Seleniumを使用したE2Eテストを自動実行するパイプラインを作成します。

|br|

著者紹介
------------------------------------------------------------------

川畑 光平(KAWABATA Kohei) - NTTデータ 課長代理

.. figure:: img/automation_infra_devops_overview/pic_image01.jpg


金融機関システム業務アプリケーション開発・システム基盤担当を経て、現在はソフトウェア開発自動化関連の研究開発・推進に従事。

Red Hat Certified Engineer、Pivotal Certified Spring Professional、AWS Certified Solutions Architect Professional等の資格を持ち、アプリケーション基盤・クラウドなど様々な開発プロジェクト支援にも携わる。

`2019 APN AWS Top Engineers & Ambassadors <https://aws.amazon.com/jp/blogs/psa/japan-apn-ambassador-2019/>`_ 選出。
