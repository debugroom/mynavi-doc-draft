.. include:: ../module.txt

.. _section-aws-microservice-cognito-oauth2-login-3-label:

【第14回】Amazon Cognito + Spring Sercurityを使ったOAuth2 Loginの実装(3)
===============================================================================================

|br|

前回から、以下のイメージのようにOAuth2 Loginをベースとしたアーキテクチャを想定した環境構築を進めています。

|br|

.. figure:: img/cognito-oauth2-login/oauth2-login-flow.png

|br|

前回は、OAuth2 Loginを実現するための認証・認可サーバとなるAmazon Cognitoについて概説し、ユーザプールをマネジメントコンソール上から構築しました。
今回以降は、OAuth2 Loginに必要なアプリクライアントの設定を行い、IDプールを構築していきます。

アプリクライアント・ドメインの設定
----------------------------------------------------------------------------------

|br|

ユーザープールは前回作成したので、同時に作成されたアプリクライアントの定義にOAuth2 Loginに必要な設定を行います。

AWSコンソール上で Amazon Cognitoサービスから、「ユーザプールの管理」で前回作成したユーザプールを選択し、「アプリクライアントの設定」を選択して以下の要領で設定を行います。

|br|

.. figure:: img/cognito-oauth2-login/management-console-cognito-setting-appclient.png

|br|

以下の設定内容に従って、アプリクライアントの設定を行います。

|br|

.. list-table::
   :widths: 1, 5

   * - 設定内容
     - 説明

   * - 有効なIDプロバイダ
     - プロバイダとして利用するユーザディレクトリを選択します。唯一のオプションになりますが、Cognitoユーザプールを指定しておきます。

   * - コールバックURL
     - アプリクライアントからリダイレクトされたCognitoで認証が完了した後、ブラウザが認可コードをCognitoから付与されて、アプリクライアントへ再びリダイレクトするURL(冒頭の図(5)の箇所)を設定します。
       次回以降、改めて解説しますが、Spring Securityで、このリダイレクトを受け入れるURLは「https://domain_name/context_path/login/oauth2/code/provider_name」形式です。
       今回は、ローカルで実行されるアプリケーション向けに「http://localhost:8080/frontend/login/oauth2/code/cognito」を設定します。なお、この設定値が正しく一致しなければ、正しく動作しません。

   * - サインアウトURL
     - (Cognitoユーザプールの)サインアウト後にリダイレクトするアプリケーションクライアントのURLを設定します。ここではローカルで実行されるフロントエンドWebアプリケーションの「http://localhost:8080/frontend」を設定しておきます。

   * - 許可されているOAuthフロー
     - 認可コードグラントである「Authorization code grant」を設定します。なお、他の選択肢であるインプリシットグラントは
       認可コードの代わりに直接アクセストークンを払い出す方式で、セキュリティ面で望ましくなく、クライアントクレデンシャルは用途が非常に限定された状況で用いられるオプションです。
       詳細は `TERASOLUNAガイドライン OAuth2.0のアーキテクチャ <http://terasolunaorg.github.io/guideline/5.6.1.RELEASE/ja/Security/OAuth.html#oautharchitecture>`_ も参考にしてください。

   * - 許可されているOAuthスコープ
     - OAuth 2.0では保護されたリソースに対するアクセスを制御する方法としてスコープという概念を使用しています。
       Cognitoはクライアントからの要求に対し、アクセストークンにスコープを含め、保護されたリソースに対するアクセス権（読み込み権限、書き込み権限など）を指定することが出来ます。
       (要はアクセストークンが認可された操作に相当する権限です)。ここでは、OIDCのユーザクレイム(ユーザ情報)を含むIDトークンや、ユーザプールのAPIオペレーションが可能な権限、ユーザ情報プロファイルにアクセス可能なよう、
       「openid」、「aws.cognito.signin.user.admin」、「profile」の３つを選択してください。詳細は
       `AWS公式の開発ガイド ユーザープールのアプリクライアントの設定 <https://docs.aws.amazon.com/ja_jp/cognito/latest/developerguide/cognito-user-pools-app-idp-settings.html>`_ も参考にしてください。

|br|

続いて、アプリクライアントからCognitoにリダイレクトされる際のドメインを設定します。ドメイン名から、有効なURLを作成してください。

|br|

.. figure:: img/cognito-oauth2-login/management-console-cognito-setting-domain.png

|br|

IDプールの設定
----------------------------------------------------------------------------------

|br|

引き続き、ユーザプールで発行されたトークンを検証するIDプールの作成を行います。
AWSコンソール上で Amazon Cognitoサービスから、「IDプールの管理」>「IDプールを作成する」を選択してください。

|br|

以下の設定内容を入力します。

* IDプール名：一意となるIDプール名
* 認証プロバイダ/Cognito/ユーザプールID：前回作成したユーザプールのID
* 認証プロバイダ/Cognito/アプリクライアントID：前回作成したアプリクライアントのID

|br|

.. figure:: img/cognito-oauth2-login/management-console-cognito-create-idpool-1.png

|br|

アクセス権限はデフォルトのままでかまいません。

|br|

.. figure:: img/cognito-oauth2-login/management-console-cognito-create-idpool-2.png

|br|


今回は、アプリクライアントおよびドメイン設定方法と、IDプールの作成について説明しました。
このようにマネジメントコンソールを使って手動で設定を行っても良いですが、前回も解説した通り、本連載では、アプリケーションから設定値をスタック経由で参照できる
CloudFormationを使って環境構築する方法を推奨します。次回以降は、CloudFormationを使ってCognitoの環境を作成してみます。

|br|

著者紹介
------------------------------------------------------------------

川畑 光平(KAWABATA Kohei) - NTTデータ エグゼクティブ ITスペシャリスト ソフトウェアアーキテクト・デジタルテクノロジーストラテジスト(クラウド)

.. figure:: img/overview/aws_361383_075.jpeg

金融機関システム業務アプリケーション開発・システム基盤担当、ソフトウェア開発自動化関連の研究開発を経て、デジタル技術関連の研究開発・推進に従事。

Red Hat Certified Engineer、Pivotal Certified Spring Professional、AWS Certified Solutions Architect Professional等の資格を持ち、アプリケーション基盤・クラウドなど様々な開発プロジェクト支援にも携わる。

`AWS Top Engineers & Ambassadors <https://aws.amazon.com/jp/partners/ambassadors/?cards-body.sort-by=item.additionalFields.ambassadorName&cards-body.sort-order=asc&cards-body.q=kawabata&cards-body.q_operator=AND>`_ 選出。

本連載記事の内容に対するご意見・ご質問は `Facebook <https://www.facebook.com/kohei.kawabata.5>`_ まで。
