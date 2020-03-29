.. include:: ../module.txt

.. _section-automation-infra-devops-codepipeline-7-label:

【第18回】AWS CodePipeLineを用いた継続的デリバリ自動化(6)
-------------------------------------------------------------------------------------------------------------------------------------

.. _section-codepipeline-setting-codepipeline-approve-production-label:

(6)プロダクション環境へのリリース承認を行うパイプラインの構築
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

本連載では、マイクロサービスアーキテクチャでの継続的デリバリ(Continuous Delivery:CD)を以下のようなパイプラインで実現していきます。

|br|

.. figure:: img/automation_infra_devops_codepipeline/pipeline.png


|br|

前回は(5)E2Eテストが完了したWeb(BackendForFrontend:BFF)アプリケーションのコンテナイメージをステージング環境へデプロイするパイプラインを構築しました。
続く今回は、AmazonSNSへのトピックの作成とパイプラインの一時停止を行い、ステージング環境で、後続の性能テスト、セキュリティテスト、受入テストを実行したのち、プロダクション環境へのリリースを承認するパイプラインを構築します。

|br|

6. パフォーマンステストやセキュリティテスト、受入等のその他テスト後の管理者によるリリース承認

* (6-1) CodePipelineがSNSに対し承認のトピックを発行し、パイプラインを一時停止させます。
* (6-2) デプロイされたBackendおよびBFFアプリケーションに対してパフォーマンステストやセキュリティテストを実行します。
* (6-3) テストの結果を踏まえ、プロダクトオーナーやマネージャがAWSコンソールでCodePipeline上で保留しているトピックを承認・否認します。

|br|

.. figure:: img/automation_infra_devops_codepipeline/pipeline-6.png


|br|

管理者による承認プロセスはAmazon SNS(Simple Notification Service)を使用して行い、承認が完了後、後続のパイプラインが動き出すように設定します。
まず、Amazon SNSの概要とトピックの設定方法について解説します。

|br|

.. _section-codepipeline-setting-codepipeline-pipeline-create-approve-sns-topic-label:

Amazon SNSの概要と承認トピックの作成
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

Amazon SNSはPublish/Subscribe型のメッセージ配信サービスです。Publish/Subscribe型では、発行者(Publisher)は、複数の購読者(Subscriber：WebServer, Mail, Amazon SQS, Amazon Lamdaなど)へ非同期にメッセージを送信できます。
SNSで配信するメッセージの特徴を列挙すると以下の通りです。

* 単一で複数の購読者へメッセージを送信できる。
* メッセージの順序は保証されない。
* 発行済みのメッセージは削除できない。
* メッセージ配信の失敗時は配信ポリシーによりリトライが可能。
* メッセージには最大256KBのテキストデータ(XML、JSON、テキスト)を含めることができる。

Amazon SNS を使用するときは、所有者としてトピックを作成し、そのトピックと通信できるPublisherとSubscriberを決定するアクセスポリシーを定義します。
Publisherは、自分が作成したトピック、または発行を許可されたトピックにメッセージを送信します。
トピックに送信されたメッセージは、Amazon SNSが、定義されたSubscriberにメッセージを送信します。AWSコンソール上から以下の操作設定が可能です。

|br|

.. list-table:: Amazon SNSの操作
   :widths: 20, 80
   :header-rows: 1

   * - API
     - 概要

   * - CreateTopic
     - 通知の公開先トピック(受け口・発信元の定義)を作成する。
   * - Subscribe
     - エンドポイントに登録確認メッセージを送信して、 |br| エンドポイントを受信(Subscribe)登録する。
   * - DeleteTopic
     - トピックとその登録サブスクリプションをすべて削除する。
   * - Publish
     - トピックを受信登録しているエンドポイントにメッセージを送信する。

|br|

それでは、プロダクション環境へのリリースを承認するためのトピックを作成しましょう。「Amazon SNS」サービスから、「トピック」メニューを選択し、「トピックの作成」ボタンを押下して、以下の要領に従って、入力します。
今回はSNSトピックの名称以外はデフォルト設定で問題ありません。

|br|

.. list-table:: Amazon SNSの設定
   :widths: 2, 3, 7

   * - 入力箇所
     - 項目
     - 説明

   * - 詳細
     - 名前
     - SNSトピックの名称を入力します。

   * -
     - 表示名(オプション)
     - ショートメッセージサービスでの表示名を入力します。

   * - 暗号化
     - 暗号化
     - 通信の暗号化に加えて、トピックのメッセージを暗号化するオプションです。有効化する場合、サーバ側の暗号化の有効化を設定します。暗号化にはAWS Key ManagementServiceを利用します。

   * - アクセスポリシー
     - メソッドの選択
     - アクセスポリシーの設定方法を選択します。「基本」ではラジオボタン選択、「高度な」ではJSONエディタによりアクセス制御を設定します。

   * - 配信再試行ポリシー
     - デフォルトの配信再試行ポリシーの使用
     - デフォルトの再試行ルールでメッセージを再送します。

   * - 配信ステータスのログ記録
     - CloudWatch Logsへのメッセージ配信ステータス設定
     - CloudWatch Logsへログ記録する対象とサンプルレート、サービスロールを設定します。

   * - タグ
     - キーと値
     - SNSトピックへ割り当てるメタデータラベルのキーと値を設定します。


|br|

.. figure:: img/automation_infra_devops_codepipeline/management_console_sns_create_approval_topic.png


|br|

なお、アクセスポリシーには、必要に応じてリリース承認を行える管理者ユーザを追加してください。

|br|

.. _section-codepipeline-setting-pipeline-for-release-approval-label:


プロダクション環境へのリリース承認フローパイプラインの設定
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

前回までのパイプラインでステージング環境へのデプロイは完了しているので、スループットやレスポンスタイムを確認するための性能テストや、ペネイトレイトテストや脆弱性診断といったセキュリティテスト、受入テストなどの完了後(ここではテストについては実施済みのものとして先へ進みます)に
承認を行うフローを想定した承認アクションをこれまで作成してきたパイプラインへ設定します。AWSコンソールの「CodePipeline」サービスを選択し、パイプラインを選択して、「編集する」ボタンを押下します。

|br|

.. figure:: img/automation_infra_devops_codepipeline/management_console_codepipeline_edit_project_31_build_bff_staging.png


|br|

新たにパイプラインステージを追加します。

|br|

.. figure:: img/automation_infra_devops_codepipeline/management_console_codepipeline_edit_project_45_approve_release.png


|br|

ステージ追加後、アクションを追加します。以下の要領に沿って、アクションを追加した後、「完了」ボタンを押下してください。

* アクション名：任意のアクション名を追加します。
* アクションプロバイダー：「Manual approval」を選択します。
* SNSトピックのARN：前節で作成したSNS Topicを設定します。
* レビュー用URL：テストの証跡や静的解析の結果などリリースの判断になる成果物があればリンクを設定します。
* コメント：管理者が承認画面を開いた場合に表示させるコメントを入力します。

|br|

.. figure:: img/automation_infra_devops_codepipeline/management_console_codepipeline_edit_project_46_approve_release.png


|br|

パイプライン変更後は、「変更をリリースする」ボタンを押下し、SNSトピックへメッセージが発行されて、承認を待機する状態となるか確認し、「Review」ボタンを押下します。

|br|

.. figure:: img/automation_infra_devops_codepipeline/management_console_codepipeline_edit_project_47_approve_release.png


|br|

コメントを入力し、承認するボタンを押下します。

|br|

.. figure:: img/automation_infra_devops_codepipeline/management_console_codepipeline_edit_project_48_approve_release.png


|br|

承認すると、正常完了し、後続のパイプラインへ処理が移行するようになります。

|br|

.. figure:: img/automation_infra_devops_codepipeline/management_console_codepipeline_edit_project_49_approve_release.png


|br|

ここまでで、プロダクション環境へのリリースを承認するパイプラインを設定できました。次回は、プロダクション環境にリリースするコンテナイメージを作成するパイプラインを構築します。

|br|

著者紹介
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

川畑 光平(KAWABATA Kohei) - NTTデータ 課長代理

.. figure:: img/automation_infra_devops_overview/pic_image01.jpg


金融機関システム業務アプリケーション開発・システム基盤担当を経て、現在はソフトウェア開発自動化関連の研究開発・推進に従事。

Red Hat Certified Engineer、Pivotal Certified Spring Professional、AWS Certified Solutions Architect Professional等の資格を持ち、アプリケーション基盤・クラウドなど様々な開発プロジェクト支援にも携わる。

`2019 APN AWS Top Engineers & Ambassadors <https://aws.amazon.com/jp/blogs/psa/japan-apn-ambassador-2019/>`_ 選出。
