.. include:: ../module.txt

.. _section-automation-infra-devops-codepipeline-8-label:

基盤・デプロイ自動化実践
==================================================================

マイクロサービスアーキテクチャの基盤・デプロイ自動化
-------------------------------------------------------------------------------------------------------------------------------------

.. _section-codepipeline-setting-codepipeline-pipeline-build-production-label:

(7)プロダクション環境へリリースするコンテナイメージをビルドして、DockerHubへプッシュするパイプラインの構築
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

本連載では、マイクロサービスアーキテクチャでの継続的デリバリ(Continuous Delivery:CD)を以下のようなパイプラインで実現していきます。

|br|

.. figure:: img/automation_infra_devops_codepipeline/pipeline.png


|br|

前回は(6)ステージング環境で、後続の性能テスト、セキュリティテスト、受入テストを実行したのち、プロダクション環境リリースを承認するパイプラインを構築しました。
続く今回は、リリースするアプリケーションのコンテナイメージをビルドしてDockerHubへプッシュするパイプラインを作成します。

|br|

7. ステージング環境でテスト済みのコンテナイメージをリリース用にタグ付けして、DockeHubへプッシュ

* (7-1) CodeBuildがアプリケーションのプロダクション環境向けbuildspec.ymlに記載したビルド処理を行うコンテナを実行します(BackendおよびBFFは並列実行)。
* (7-2) 各々ビルド処理の中で、アプリケーション実行コンテナイメージを作成し、DockerHubへプッシュします。

|br|

.. figure:: img/automation_infra_devops_codepipeline/pipeline-7.png


|br|

.. _section-codepipeline-create-production-buildspec-label:

プロダクション環境のコンテナイメージをビルドするbuildspec.ymlの作成
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

:ref:`section-codepipeline-create-backend-staging-buildspec-label` や :ref:`section-codepipeline-create-buildspec-for-building-bff-label` の手順と同様、
下記のようなフォルダ構成で、各アプリケーションのコンテナイメージをビルドするbuildspec.ymlを作成します(上記イメージ7-1で実行)。

|br|

.. sourcecode:: bash

   [project-root]
       │
       ├-[backend]
       │   ├- src
       │   │   ├- main .....
       │   │   └- test .....
       │   ├- build
       │   │   ├- production
       │   │   |   ├- dockerfile
       │   │   │   └- buildspec.yml
       │   │  .....
       │   └- pom.xml
       │
       ├-[backend-for-frontend]
       │   ├- src
       │   │   ├- main .....
       │   │   └- test .....
       │   ├- build
       │   │   ├- production
       │   │   |   ├- dockerfile
       │   │   │   └- buildspec.yml
       │   │  .....
       │   └- pom.xml
       │  .....
       └- pom.xml

|br|

buildspec.ymlの基本的な処理の構成や説明は、前回の手順とほぼ同様ですが、プロダクション環境では、ステージング環境でビルドしたコンテナイメージをそのまま利用する形で、DockerHubからコンテナイメージをプルしてからプロダクション環境へのタグへ付け替え、プッシュするようにします。
Backendマイクロサービスのbuildspec.ymlは以下の通りです。

|br|

.. sourcecode:: bash

   version: 0.2
   env:
     parameter-store:
       DOCKER_USER: "DOCKER_USER"
       DOCKER_PASSWORD: "DOCKER_PASSWORD"
       DOCKER_REPO : "DOCKER_REPO"
       IMAGE_REPO_NAME: "BACKEND_IMAGE_REPO_NAME"
       IMAGE_TAG_STAGING: "BACKEND_IMAGE_TAG_STAGING"
       IMAGE_TAG: "BACKEND_IMAGE_TAG_PRODUCTION"
   phases:
     install:
       runtime-versions:
         docker: 18
     pre_build:
       commands:
         - echo Logging in to Docker Hub...
         - docker login -u $DOCKER_USER -p $DOCKER_PASSWORD $DOCKER_REPO
     build:
       commands:
         - echo Build started on `date`
         - echo Building the Docker image...
         - docker pull $IMAGE_REPO_NAME:$IMAGE_TAG_STAGING
         - docker tag $IMAGE_REPO_NAME:$IMAGE_TAG_STAGING $IMAGE_REPO_NAME:$IMAGE_TAG
     post_build:
       commands:
         - echo Build completed on `date`
         - echo Pushing the Docker image...
         - docker push $IMAGE_REPO_NAME:$IMAGE_TAG
         - printf '[{"name":"mynavi-sample-continuous-delivery-backend-production","imageUri":"%s"}]' $IMAGE_REPO_NAME:$IMAGE_TAG > imagedefinitions.json
   artifacts:
     files:
         - imagedefinitions.json

|br|

マイクロサービスを呼び出すBFFアプリケーションのプロダクション環境向けbuildspec.ymlは以下の通りです。構成は同様で、DockerHubからコンテナイメージをプルしてからタグをプロダクション環境のものに付け替えてプッシュします。

|br|

.. sourcecode:: bash

   version: 0.2
   env:
     parameter-store:
       DOCKER_USER: "DOCKER_USER"
       DOCKER_PASSWORD: "DOCKER_PASSWORD"
       DOCKER_REPO : "DOCKER_REPO"
       IMAGE_REPO_NAME: "BACKEND_FOR_FRONTEND_IMAGE_REPO_NAME"
       IMAGE_TAG_STAGING: "BACKEND_FOR_FRONTEND_IMAGE_TAG_STAGING"
       IMAGE_TAG: "BACKEND_FOR_FRONTEND_IMAGE_TAG_PRODUCTION"
   phases:
     install:
       runtime-versions:
         docker: 18
     pre_build:
       commands:
         - echo Logging in to Docker Hub...
         - docker login -u $DOCKER_USER -p $DOCKER_PASSWORD $DOCKER_REPO
     build:
       commands:
         - echo Build started on `date`
         - echo Building the Docker image...
         - docker pull $IMAGE_REPO_NAME:$IMAGE_TAG_STAGING
         - docker tag $IMAGE_REPO_NAME:$IMAGE_TAG_STAGING $IMAGE_REPO_NAME:$IMAGE_TAG
     post_build:
       commands:
         - echo Build completed on `date`
         - echo Pushing the Docker image...
         - docker push $IMAGE_REPO_NAME:$IMAGE_TAG
         - printf '[{"name":"mynavi-sample-continuous-delivery-bff-production","imageUri":"%s"}]' $IMAGE_REPO_NAME:$IMAGE_TAG > imagedefinitions.json
   artifacts:
     files:
       - imagedefinitions.json


|br|


.. _section-codepipeline-setting-ssm-definition-backend-bff-production-label:

AWS Sysmtems Managers Parameter Storeでの環境変数の定義
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

CodePipelineの設定を行う前に、前節のbuildspec.ymlで、新たに使用する環境変数を定義しておきます。設定の要領は、 :ref:`section-codebuild-setting-sms-label` と同様です。以下のパラメータを定義します。

* "BACKEND_FOR_FRONTEND_IMAGE_TAG_PRODUCTION"：1.0.RELEASE
* "BACKEND_IMAGE_TAG_PRODUCTION"：1.0.RELEASE

|br|

.. _section-codepipeline-setting-pipeline-for-build-production-label:

ビルドプロジェクトの作成とCodePipelineの設定
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

これまで作成してきたCodePipelineを編集して、各アプリケーションのプロダクション向けコンテナイメージを作成するためのCodeBuildプロジェクトを作成し、設定します。
AWSコンソールの「CodePipeline」サービスを選択し、パイプラインを選択して、「編集する」ボタンを押下します。

|br|

.. figure:: img/automation_infra_devops_codepipeline/management_console_codepipeline_edit_project_31_build_bff_staging.png


|br|

今回は、前回作成した承認のステージの後にアクションを追加するので、「ステージを追加する」ボタンを押下し、アクションを追加します。

|br|

.. figure:: img/automation_infra_devops_codepipeline/management_console_codepipeline_edit_project_50_build_backend_production.png


|br|

releaseステージを追加し、「アクショングループを追加する」ボタンを押下して、アクションを追加します。

|br|

.. figure:: img/automation_infra_devops_codepipeline/management_console_codepipeline_edit_project_51_build_backend_production.png


|br|

.. figure:: img/automation_infra_devops_codepipeline/management_console_codepipeline_edit_project_52_build_backend_production.png


|br|

まずBackendのプロダクション向けコンテナイメージをビルドするための設定を行います。以下の要領でアクションを設定し、「完了」ボタンを押下します。

* アクション名：任意のアクション名を追加します。
* アクションプロバイダー：「AWS CodeBuild」を選択します。
* リージョン：プロダクション環境があるリージョンを選択します。
* 入力アーティファクト：１番目のパイプラインで出力アーティファクトとなっている「SourceArtifact」を選択します。なお、これはGitHubからチェックアウトしたソースコードプロジェクトで実体はS3にZIPアーカイブされて保存されています。
* プロジェクト名：「プロジェクトを作成する」ボタンを押下して、後述する要領で「CodeBuild」プロジェクトを作成します。
* 出力アーティファクト：出力アーティファクトはこれまでのアーティファクト名と重複しない任意の名称を入力します。

|br|

.. figure:: img/automation_infra_devops_codepipeline/management_console_codepipeline_edit_project_53_build_backend_production.png


|br|

|br|

また、CodeBuildプロジェクトの設定要領は :ref:`section-codebuild-setting-codebuild-label` と同様ですが、以下の点に留意して設定しましょう。

|br|

.. list-table:: CodeBuildの設定
   :widths: 2, 3, 7

   * - 入力箇所
     - 項目
     - 説明

   * - 環境
     - 特権付与
     - dockerコマンドなどを実行する場合、チェックが必要です。

   * -
     - サービスロール
     - CodeBuildの実行に必要なポリシーをアタッチしたサービスロールを設定します。複数のビルドプロジェクトで関連付けが可能ですが、最大10のビルドプロジェクトまでしか関連付けができないので注意が必要です。なお、サービスロールを新規作成する場合は作成後にSystemsManagerのアクセス権限を付与します。

   * - Buildspec
     - buildspec名
     - デフォルトではソースコードのルートディレクトリにあるbuildspec.ymlが選択されますが、別の名前や場所を使用している場合に入力します。ここでは、前節で作成したbackend/build/production/buildspec.ymlを指定します。


|br|

.. figure:: img/automation_infra_devops_codepipeline/management_console_codepipeline_edit_project_54_codebuild_backend_production.png


|br|
|br|

.. figure:: img/automation_infra_devops_codepipeline/management_console_codepipeline_edit_project_55_codebuild_backend_production.png


|br|
|br|

.. figure:: img/automation_infra_devops_codepipeline/management_console_codepipeline_edit_project_56_codebuild_backend_production.png


|br|

作成が完了したら、引き続きBFFアプリケーションのプロダクション向けコンテナイメージをビルドするための設定を行います。E2Eテストが必要だったステージング環境とは違い、今回のプロダクション環境の設定では特にBackendアプリケーションを先に構築する必要はないので、並列化して実行するように設定しましょう。

|br|

.. figure:: img/automation_infra_devops_codepipeline/management_console_codepipeline_edit_project_57_build_bff_production.png


|br|

Backendと同じ要領で、CodeBuildプロジェクトを作成、アクションを設定し、「完了」ボタンを押下します。

|br|

.. figure:: img/automation_infra_devops_codepipeline/management_console_codepipeline_edit_project_58_build_bff_production.png


|br|

パイプラインの編集完了後、保存ボタンを押下します。サービスロールを新規作成している場合は、CodeBuildからSystemsManagerParameterStoreに環境変数のアクセスができるよう、サービスロールに権限を付与しておきます。

|br|

.. figure:: img/automation_infra_devops_codepipeline/management_console_codepipeline_edit_project_62_add_ssm.png


|br|
|br|

作成したパイプラインを実行し、DockerHub上にプロダクション向けのコンテナイメージが作成されていることを確認します。

|br|

.. figure:: img/automation_infra_devops_codepipeline/management_console_codepipeline_edit_project_63_build_bff_production.png


|br|
|br|

.. figure:: img/automation_infra_devops_codepipeline/dockerhub_backend_production.png



ここまでで、プロダクション環境のコンテナイメージを作成し、DockerHubへプッシュするパイプラインを設定できました。次回は、プロダクション環境にデプロイするパイプラインを構築します。

著者紹介
------------------------------------------------------------------

川畑 光平(KAWABATA Kohei) - NTTデータ 課長代理

.. figure:: img/automation_infra_devops_overview/pic_image01.jpg


金融機関システム業務アプリケーション開発・システム基盤担当を経て、現在はソフトウェア開発自動化関連の研究開発・推進に従事。

Red Hat Certified Engineer、Pivotal Certified Spring Professional、AWS Certified Solutions Architect Professional等の資格を持ち、アプリケーション基盤・クラウドなど様々な開発プロジェクト支援にも携わる。

`2019 APN AWS Top Engineers & Ambassadors <https://aws.amazon.com/jp/blogs/psa/japan-apn-ambassador-2019/>`_ 選出。
