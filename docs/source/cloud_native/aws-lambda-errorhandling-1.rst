.. include:: ../module.txt

.. _section-cloud-native-lambda-error-handling-1st-label:

【第6回】AWS Lambdaにおけるサーバレスエラーハンドリング(1)
----------------------------------------------------------------------------------------

|br|

前回は、S3へファイルアップロードした後の後続処理をAWS Lambdaを使ってサーバレスで実行するアプリケーションを実装しました。
今回からはAWS Lambdaでエラーが発生した場合の様々なエラーハンドリングの実装方法について解説していきます。

|br|

.. _section-cloud-native-lambda-error-handling-label:

AWS Lambdaを使ったエラーハンドリング
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

サーバレスコンピューティングサービスであるLambdaを利用したエラーハンドリングは通常のアプリケーションで実装する場合と何が異なるのでしょうか？
大きな特徴として、以下のようなものがあります。

|br|

.. list-table:: Lambdaにおけるエラーハンドリングの特徴
   :widths: 2, 8

   * - 特徴
     - 説明

   * - マネージドサービスからの非同期実行
     - 前回まで解説してきたS3へファイルアップロードした後のイベント駆動実行のように、AWSのマネージドサービスから非同期的に処理が実行されるケースでも使用されます。
       クラウドネイティブアプリケーションの基本第1回でも解説したように、API Gatewayから同期的にLambdaを呼び出す場合もありますが、
       サーバにかかる負荷を逃したいケースなどサーバレスが求められる場合では、非同期処理のかたちでLambdaを利用する機会も多くなります。
       非同期処理におけるエラーハンドリングで、マネージドサービスを活用して、然るべき対象に適切なエラー通知を実装していく必要があります。

   * - リトライ・通知機能、マネージドサービス連携
     - AWS Lambdaで処理中に発生したエラーのハンドリング処理として、Lambda自体でリトライしたり、CloudWatchやSNS/SQSなど、
       様々なマネージドサービスと連携して通知することができます。通知先や内容によって適切に使い分けることが必要です。

そこで、今回から代表的なエラーハンドリングのパターンとして、以下の図のようなケースを解説していきます。

|br|

.. figure:: img/aws-lambda-errorhandling/errorhandling-overview.png

|br|

各ケースの詳細は以下の通りです。

|br|

.. list-table:: Lambdaのエラーハンドリングのユースケース
   :widths: 1, 19

   * - 項番
     - 説明

   * - (1)
     - AWS Lambdaを同期的に呼び出す場合のエラーハンドリングパターンです。LambdaはAPI Gatewayを介して、同期的に呼び出されます。

   * - (1-a)
     - AWS Lambdaを同期的に呼び出し、実行中にエラーが発生した場合、呼び出し元にエラーメッセージを通知します。

   * - (1-b)
     - AWS Lambdaを同期的に呼び出し、実行中にエラーが発生した場合、CloudWatch Logsへエラーが出力されます。

   * - (1-c)
     - CloudWatch Logsへ特定のエラーが出力されたことを検知して、エラーの後処理を行う別のLambdaファンクションを実行します。

   * - (1-d)
     - CloudWatch Logsへ出力されたメッセージをフックして、Mattermostなどのチャットコミュニケーションツールの特定のチャネルへ出力します。

   * - (1-e)
     - システムの管理者や運用担当者がボット用のチャネルを通じて、通知されたエラーを確認します。

   * - (2)
     - AWS Lambdaが非同期的に呼び出される場合のエラーハンドリングパターンです。前回までの連載で解説した通り、S3へファイルアップロードされたイベントを契機に、Lambdaファンクションが呼び出されます。

   * - (2-a)
     - S3へファイルがアップロードされたことをイベントの契機として、Lambdaファンクションが実行されます。

   * - (2-b)
     - 非同期に呼び出されたAWS Lambdaの実行中にエラーが発生した場合、CloudWatch Logsへエラーが出力されます。

   * - (2-c)
     - 非同期に呼び出されたAWS Lambdaの実行中にエラーが発生した場合、エラー発生内容をアップロードしたユーザに通知するためにSQSへキューを送信します。

   * - (2-d)
     - 非同期に呼び出されたAWS Lambdaの実行中にエラーが発生した場合、デッドレターキューを使って、Amazon SNSへ通知を行います。

   * - (2-e)
     - デッドレターキューが通知するSNSトピックがエラー発生をシステム管理者や運用担当者へSMSで通知します。

   * - (2-f)
     - デッドレターキューが通知するSNSトピックがエラー発生をシステム管理者や運用担当者へメールで通知します。


次回以降、上記のケースをSpring Cloud FunctionやCloudFormationを使って構築する実装を順次解説してきますが、
その前の前提として、本連載で取り扱うエラーの概念について説明します。


|br|

.. _section-cloud-native-lambda-error-handling-concept-label:

エラーの概念、対処の考え方
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

本連載では、エラーをビジネスエラーとシステムエラーの２種類に分けて取り扱います。これらは、`TERASOLUNAガイドライン 例外  <http://terasolunaorg.github.io/guideline/5.6.1.RELEASE/ja/ArchitectureInDetail/WebApplicationDetail/ExceptionHandling.html#exception-handling-exception-type-label>`_
で述べられているところの「ビジネス例外」と「システム例外」とほぼ同義になります。ポイントを整理すると以下の通りです。

|br|

.. list-table:: エラーの概念と基本的な対処方針
   :widths: 2, 8

   * - 種類
     - 説明

   * - ビジネスエラー
     - アプリケーションのビジネスロジック上想定されるエラー(在庫がない、入力されたログインIDがない等)。システム管理者や運用者による対処が不要で、エラーが発生した処理を実行したユーザにエラー内容を通知し、適切な対処やオペレーションを促す

   * - システムエラー
     - システムが正常稼働しているときには発生しない、もしくはシステム全体に影響を及ぼす致命的な問題が発生しているときのエラー(アプリケーションバグで発生したエラーやデータベースがダウンしている等)。システム管理者や運用者による対処が必要であり、ユーザにエラー内容を通知しても、ユーザ側では対処しようがないため、エラーが発生したことのみを通知し、コールセンターへの連絡や、時間をおいてからの再実行を促す。

|br|

エラーの種別により、通知するべき対象や内容は異なります。加えて、前節でも説明した通り、Lambdaではマネージドサービスを契機とした非同期処理などでユーザ側への通知処理が複雑化するケースがあります。
また、通知を行う機能やマネージドサービスが複数存在し、通知する内容によって適した方式も変わってきますので、Lambdaで実装する処理の内容に応じて、適切なエラーハンドリング方法を選択するようにしましょう。

|br|


今回はLambdaファンクションのエラーハンドリングの特徴と代表的なパターン、エラーの概念・対処の考え方を説明しました。次回以降はAPI GatewayとLambdaおよびSpringCloudFuntionを使って構築したLambdaを同期的に呼び出した場合に発生したエラーの取り扱い方や環境構築方法を解説します

|br|

著者紹介
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

川畑 光平(KAWABATA Kohei) - NTTデータ エグゼクティブ ITスペシャリスト ソフトウェアアーキテクト・デジタルテクノロジーストラテジスト(クラウド)

.. figure:: img/aws-s3-and-lambda/aws_361383_075.jpeg

金融機関システム業務アプリケーション開発・システム基盤担当、ソフトウェア開発自動化関連の研究開発を経て、デジタル技術関連の研究開発・推進に従事。

Red Hat Certified Engineer、Pivotal Certified Spring Professional、AWS Certified Solutions Architect Professional等の資格を持ち、アプリケーション基盤・クラウドなど様々な開発プロジェクト支援にも携わる。

`AWS Top Engineers & Ambassadors <https://aws.amazon.com/jp/partners/ambassadors/?cards-body.sort-by=item.additionalFields.ambassadorName&cards-body.sort-order=asc&cards-body.q=kawabata&cards-body.q_operator=AND>`_ 選出。

本連載記事の内容に対するご意見・ご質問は `Facebook <https://www.facebook.com/kohei.kawabata.5>`_ まで。
