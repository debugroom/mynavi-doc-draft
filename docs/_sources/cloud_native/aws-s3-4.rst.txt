.. include:: ../module.txt

.. _section-cloud-native-s3-4th-label:

【第1回】AmazonS3へダイレクトアクセスするアプリケーション実装(4)
----------------------------------------------------------------------------------------

|br|

これまで32回にわたり、 `AWSで作るクラウドネイティブアプリケーションの基本 <https://news.mynavi.jp/itsearch/series/devsoft/AWS.html>`_ で、
AWS ECSやALB、RDS、DynamoDB、ElastiCache、SQS、AWS Lambda、APIGateway等を使って、SpringBootをベースとしたアプリケーションを実装する方法を紹介してきました。

|br|

本連載では、以下のようなAWSサービスを使って、より発展的なアプリケーションの実装方法を紹介していきます。

|br|

* Amazon S3へのダイレクトアクセス
* Amazon DynamoDBにおける高度なデータモデル、より実践的なアプリケーション実装
* Amazon Keyspaces(for Apache Cassandra)を使ったアプリケーション実装
* Amazon ECS Fargateを使ったアプリケーション実装
* Amazon EKS を使ったアプリケーション実装
* Amazon Auroraを使ったアプリケーション実装
* Amazon Kinesisを使ったアプリケーション実装
* Amazon MSK(Managed Streaming for Apache Kafka)を使ったアプリケーション実装
* Amazon MQ(Message Queue)を使ったアプリケーション実装
* AWS Batchを使ったアプリケーション実装
* AWS StepFunctionsを使ったアプリケーション実装

|br|

.. note:: 順不同であり、上記のサービスを組み合わせるケースもあります。

|br|

発展編の初回のテーマとして、基本編で紹介したAmazon S3、およびSTS(Security Token Service)を使ったより高度なSpringBootアプリケーションの実装方法について解説します。
本編を読み進めるにあたって、`基本編25回 <https://news.mynavi.jp/itsearch/article/devsoft/4597>`_ 、`26回 <https://news.mynavi.jp/itsearch/article/cloud/4615>`_　`27回 <https://news.mynavi.jp/itsearch/article/devsoft/4651>`_ も適宜参照ください。

|br|

なお、本連載は以下の前提知識がある開発者を想定しています。

|br|

.. list-table::
   :widths: 3, 7

   * - 対象読者
     - 前提知識

   * - エンタープライズ開発者
     - Java言語及びSpringFrameworkを使った開発に従事したことがある経験者。経験がなければ、|br|
       `こちらのチュートリアル <http://terasolunaorg.github.io/guideline/5.5.1.RELEASE/ja/Tutorial/index.html>`_ を実施することを推奨します。TERASOLUNAはSpringのベストプラクティスをまとめた開発方法論で、このチュートリアルでは、JavaやSpring Frameworkを使った開発に必要な最低限の知識を得ることができます。

   * -
     - GitHubなどのバージョン管理ツールやApache Mavenなどのライブラリ管理ツールを使った開発に従事したことがある経験者。

   * - AWS開発経験者
     - AWSアカウントをもち、コンソール上で各サービスを実行したことがある経験者

|br|

また、動作環境は以下のバージョンで実施しています。

|br|


.. list-table::
   :widths: 5, 5

   * - 動作対象
     - バージョン

   * - Java
     - 1.8

   * - Spring Boot
     - 2.1.6.RELEASE

|br|

将来的には、AWSコンソール上の画面イメージや操作、バージョンアップによりJavaのソースコード内で使用するクラスが異なる可能性があります。

|br|

.. _section-cloud-native-s3-direct-upload-download-label:

Amazon S3へのダイレクトアップロード・ダウンロード
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

`26回 <https://news.mynavi.jp/itsearch/article/cloud/4615>`_ でも解説した通り、アプリケーションからAmazon S3にアクセスする方法は２つあります。
VPC内にあるEC2やECSで実行されているアプリケーション内からAmazon SDK(Software Developers Kit:ライブラリ)を使ってアクセスする方法と、
下記の図に示すように、AWS STS(Security Token Service)を使って、一時的に有効なURLや署名されたS3へアクセスポリシーを使って、クライアントからダイレクトにアップロード・ダウンロードを行う方法です。

|br|

.. figure:: img/aws-s3/S3DirectAccessNewIcon.png

|br|

前者のアプリケーション実装方法は基本編で解説していますので、今回はダイレクトアップロード・ダウンロードについて紹介していきます。まず、STSについてです。

|br|

.. _section-cloud-native-s3-sts-label:

AWS STSの概要
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

AWS STSではAWSリソースのアクセスを一時的に許可する認証情報を作成するためのサービスです。STSに対して、SDKやCLI(Command Line Interface)から、AssumeRoleというAPIを実行すると、
指定したIAMロールに設定された権限と同様のポリシーを一時的な認証情報として受け取ることができます(AWSコンソール上からはSTSのサービスはないので実行できません)。
もちろん、誰でもAssumeRole APIを実行してポリシーを取得できるわけではありません。事前にAsumeRole APIが実行可能な対象を、信頼された対象(エンティティ)として設定しておく必要があります。今回はその対象として指定したアプリケーションユーザーが、
引き受け対象として設定されたIAMロールに付与されたポリシーを取得し、一時的に許可されたAWSリソースへアクセスする権限を得るのです。
それでは、実際にIAMでAssumeRoleの設定とS3へのアクセスが許可されたポリシーを設定してみましょう。

|br|

.. _section-cloud-native-s3-create-assumerole-s3policy-label:

IAMロールとポリシーの設定
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

今回は、アプリケーション内でSDKのライブラリからSTSのAssumeRole APIを実行するので、信頼されたエンティティとして設定するアプリケーションユーザを事前に作成しておく必要があります。
基本編で既にアプリケーション用のユーザは作成しているので、本連載ではそれを利用しますが、新たにアプリケーションユーザを作成する場合は、`第11回 <https://news.mynavi.jp/itsearch/article/devsoft/4422>`_ を参考にして下さい。
なお、S3へのアクセスは後ほど作成するIAMロールに設定するので、アプリケーションユーザには特にS3へのアクセスポリシーは設定しなくてもかまいません。
(`26回 <https://news.mynavi.jp/itsearch/article/cloud/4615>`_ のようにアプリケーションから直接S3へアクセスする場合は別ですが、今回はAssumeRoleとして引き受けるIAMロールとそれにアタッチするポリシーが必要なものです。)

|br|

それでは、まずIAMロールにアタッチする、S3へのアクセスポリシーを作成します。AWSコンソールから、IAMサービスを選択し、ポリシーメニューから「ポリシーの作成」ボタンをクリックします。
サービスは「S3」、アクションはダウンロード向けに「GetObject」を選択します。

|br|

.. figure:: img/aws-s3/management-console-iam-create-policy-s3-download-1.png

|br|

.. figure:: img/aws-s3/management-console-iam-create-policy-s3-download-2.png

|br|

アクセス対象となるS3のバケットとオブジェクトキーを指定します。基本編で作成したS3バケット以下へのアクセスを許可する設定を行います。

|br|

.. figure:: img/aws-s3/management-console-iam-create-policy-s3-download-3.png

|br|

設定を確認して、「ポリシーの作成」ボタンを押下します。

|br|

.. figure:: img/aws-s3/management-console-iam-create-policy-s3-download-4.png

|br|

同様に、アップロード向けのポリシーとして、アクションを「PutObject」にして、ポリシーを作成します。

|br|

.. figure:: img/aws-s3/management-console-iam-create-policy-s3-upload-1.png

|br|

.. figure:: img/aws-s3/management-console-iam-create-policy-s3-upload-2.png

|br|

.. figure:: img/aws-s3/management-console-iam-create-policy-s3-upload-3.png

|br|

.. figure:: img/aws-s3/management-console-iam-create-policy-s3-upload-4.png

|br|

続いて、作成したポリシーをアタッチするロールを作成します。このロールが引き受けるAssumeRoleになります。同じくIAMロールメニューから、「ロールの作成」ボタンを押下し、
信頼されたエンティティとして「別のAWSアカウント」を選択し、自身のアカウントIDを入力して、「次のステップ：アクセス権限」ボタンをクリックします。

|br|

.. figure:: img/aws-s3/management-console-iam-create-role-s3-download-1.png

|br|

先ほど作成したポリシーをアタッチします。

|br|

.. figure:: img/aws-s3/management-console-iam-create-role-s3-download-2.png

|br|

.. figure:: img/aws-s3/management-console-iam-create-role-s3-download-3.png

|br|

上記はDownloadポリシーを設定した例ですが、同様にUpload向けのロールも作成しておいて下さい。

|br|

続いて、上記の作成したロールは信頼されたエンティティがアカウントになっているので、アプリケーション用のユーザとなるよう信頼されたエンティティを編集します。
作成したロールを選択し、「信頼関係」タブから、「信頼関係の編集」ボタンを押下して、以下の通り、作成したユーザを「arn:aws:iam:[account-id]:user/[app-user-name]」となるよう編集して、「信頼ポリシーの更新」ボタンを押下します。

|br|

.. figure:: img/aws-s3/management-console-iam-edit-role-entity-download-1.png

|br|

.. figure:: img/aws-s3/management-console-iam-edit-role-entity-download-2.png

|br|

上記はDownloadの例ですが、同様にアップロード用のロールも信頼されたエンティティを編集しておいてください。

|br|

これで、S3へダイレクトアップロード・ダウンロードするアプリケーションを実装する準備が整いました。次回は、S3へアクセスが許可された署名付きURLを生成してクライアントからダイレクトファイルダウンロードするアプリケーションを実装してみます。

|br|

著者紹介
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

川畑 光平(KAWABATA Kohei) - NTTデータ

.. figure:: img/aws-s3/aws_361383_075.jpeg

金融機関システム業務アプリケーション開発・システム基盤担当、ソフトウェア開発自動化関連の研究開発を経て、デジタル技術関連の研究開発・推進に従事。

Red Hat Certified Engineer、Pivotal Certified Spring Professional、AWS Certified Solutions Architect Professional等の資格を持ち、アプリケーション基盤・クラウドなど様々な開発プロジェクト支援にも携わる。

`AWS Top Engineers & Ambassadors <https://aws.amazon.com/jp/partners/ambassadors/?cards-body.sort-by=item.additionalFields.ambassadorName&cards-body.sort-order=asc&cards-body.q=kawabata&cards-body.q_operator=AND>`_ 選出。

本連載記事の内容に対するご意見・ご質問は `Facebook <https://www.facebook.com/kohei.kawabata.5>`_ まで。
