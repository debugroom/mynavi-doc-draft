.. include:: ../module.txt

.. _section-cloud-native-api-gateway-and-lambda-label-2:

AWSで作るクラウドネイティブアプリケーションの基本
========================================================================================

.. _section-cloud-native-api-gateway-and-lambda-2nd-label:

第1回 API GatewayとLambdaを使ったサーバレスSpringアプリケーション(2)
----------------------------------------------------------------------------------------

|br|

クラウド時代が到来し、ますます広がりを見せつつあるサーバレス開発。初回はAWS LambdaとAPI Gatewayを使ったサーバレス環境下でSpringアプリケーションを構築する方法を解説します。

|br|

前回の記事「 :ref:`section-cloud-native-api-gateway-and-lambda-1st-label` 」ではSpring Cloud Functionでサーバレスアプリケーションを実装しました。
今回は、作成したアプリケーションをAWS Lambdaへ登録してみましょう。

|br|

.. _section-setting-aws-lambda-label:

(2)AWS Lambdaの設定
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

作成したサーバレスアプリケーションをAWS Lambda Functionとして設定します。AWSコンソールのLambdaサービスを選択し、
「関数の作成」ボタンを押下してください。

|br|

.. figure:: img/aws-lambda-and-api-gateway/management-console-lambda-create-function-1.png
   :scale: 100%

|br|

次に、「１から作成」メニューを選択し、以下の通り設定を行います。

* 名前：任意の名前を設定
* ランタイム：Java8を選択
* ロール：初めてLambdaを使用する場合でロールを初めて作成する場合はカスタムロールの作成を選択してください。

|br|

.. figure:: img/aws-lambda-and-api-gateway/management-console-lambda-create-function-2.png
   :scale: 100%

|br|

カスタムロールの作成を選択すると、IAMロール画面へのウィンドウが開き、Lambda関数の実行やCloudWatchのログ書き込み権限など最低限の権限を設定したIAMロールが自動で表示されます。
「新しいIAMロールの作成」を選んだまま、任意のロール名を入力し、「許可ボタン」を押してください。

|br|

.. figure:: img/aws-lambda-and-api-gateway/management-console-lambda-create-function-3.png
   :scale: 100%

|br|

その後、Lambda作成画面に戻り、「関数の作成」ボタンを押下すると、Lambda関数が作成されます。

|br|

.. figure:: img/aws-lambda-and-api-gateway/management-console-lambda-create-function-4.png
   :scale: 100%

|br|

そのまま下にスクロールすると、関数のコードや実行条件に関する詳細な設定項目が表示されます。関数コード・環境変数の設定で以下の要領に則って作成します。

.. list-table:: 関数のコードと環境変数での入力の要領
   :widths: 3, 7

   * - 入力項目
     - 説明

   * - コードエントリタイプ
     - Lambda関数として実行するアプリケーションをアップロードする方式を決定します。|br| ファイルを直接アップロードするか、S3経由でアップロードするかになりますが、|br| JavaではJAR形式のファイルをアップロードする形になります。
       |br| 10MBをオーバーしていてもアップロードできないことはありませんが、ファイルサイズが大幅に大きくなる場合は |br| S3へアップロードする方法を指定してください。

   * - ランタイム
     - Lambda関数として実行するアプリケーションのランタイムを指定します。 |br| 今回はJava8を指定しておいてください。

   * - ハンドラ
     - 今回作成した3種類のクラスのうち、HandlerクラスをFQCNで設定します。

   * - 関数パッケージ
     - コードエントリタイプでzip、jar形式を指定していた状態で、押下すると |br| アップロードダイアログが表示されます。作成したアプリケーションのJARファイルを指定してください。|br| S3経由のアップロードを選択していた場合は、オブジェクトキーを指定します。

   * - 環境変数
     - Spring Cloud Functionでは、実行する関数を環境変数FUNCTION_NAMEで指定した |br| Bean名で取得します。
       ここでは、今回作成した3種類のクラスのうち、FunctionクラスのBean名を指定します。|br|
       クラス名をキャメルケース(最初の英大文字を小文字にする形で)指定してください。

|br|

最低限設定が必要なものは上記になります、実行条件はデフォルトのままで問題ありませんので、入力後は画面上部にある保存ボタンを押下してください。

.. note:: 重たい処理などはあらかじめ実行メモリを大きめにとっておいた方が良い場合があります。HelloWorldだけのシンプルなアプリケーションでも最低100MBはメモリ消費します。

|br|

.. figure:: img/aws-lambda-and-api-gateway/management-console-lambda-create-function-5.png
   :scale: 100%

|br|

.. warning:: 設定を保存して「テスト」ボタンを実行しても、この段階ではエラーになります。次回詳細を記述しますが、API Gatewayで、Lambdaプロキシ統合オプションに基づくリクエストが必要になるので、ここでエラーになっても気にしないでください。

|br|

これで、作成したアプリケーションをLambda Functionとして設定できました。次回は、サーバレスアプリケーションを設定したLambdaをAmazon API Gatewayを使って呼び出してみます。

著者紹介
------------------------------------------------------------------

川畑 光平(KAWABATA Kohei)

.. figure:: img/aws-lambda-and-api-gateway/pic_image01.jpg
   :scale: 100%

某システムインテグレータにて、金融機関システム業務アプリケーション開発・システム基盤担当を経て、現在はソフトウェア開発自動化関連の研究開発・推進に従事。

Red Hat Certified Engineer、Pivotal Certified Spring Professional、AWS Certified Solutions Architect Professional等の資格を持ち、アプリケーション基盤・クラウドなど様々な開発プロジェクト支援にも携わる。

本連載記事の内容に対するご意見・ご質問は `Facebook <https://www.facebook.com/kohei.kawabata.5>`_ まで。
