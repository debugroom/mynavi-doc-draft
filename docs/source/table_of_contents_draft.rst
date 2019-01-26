.. include:: module.txt

.. _section-table-of-contents-draft-label:

記事目次案
==================================================================

1. 基盤自動化・デプロイ自動化実践編
------------------------------------------------------------------

【趣旨】
基盤・デプロイ自動化で記述した概要に書かれている事例を実際に構築してみる回。
記事でも記載した通り、アプリケーションアーキテクチャ、基盤、リリースプロセスを一体として考える必要があるため、
各々回を分け、エッセンスを記述する。

【想定読者】
小規模開発のアプリケーション・システム基盤開発者、大規模開発でシステム基盤開発者

1-1. マイクロサービスアーキテクチャでのAP実装
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

#. SpringBootアプリケーションの実装

   #. `単一のマイクロサービスの実装 <http://debugroom.github.io/doc/java/spring/spring-boot/usage.html#web>`_
   #. サービス分割指針
   #. サービス間連携する場合の実装

#. `AWS ECSを使ったマイクロサービスの登録 <http://debugroom.github.io/doc/cloud/aws/article/computing.html#elastic-container-service>`_ (競合・類似製品：Kubernetes、AWS EKS)

   #. Dockerファイルの作成
   #. ECSクラスタの作成
   #. ECSタスクの定義
   #. ロードバランサの設定
   #. ECSサービスの定義

1-2. AWS CodePipeLineを使った継続的デリバリ
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

#. `AWS CodeBuild構築 <http://debugroom.github.io/doc/cloud/aws/article/devops.html#codebuild>`_ (競合・類似製品：Travis CIなど)

   #. GitHubへのコミット
   #. プロジェクトの設定
   #. アプリケーションのビルド

#. `AWS CodePipeLine構築 <http://debugroom.github.io/doc/cloud/aws/article/devops.html#codepipeline>`_ (競合・類似製品：Jenkins Pipeline、Spring Cloud Pipeline)

   #. CodePipeLineの設定

1-3. AWS CloudFormationを使った基盤自動化(競合・類似製品：Terraformなど)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

#. CloudFormationによる基盤構築

   #. VPCの構築
   #. ELBの構築
   #. セキュリティグループの構築
   #. ECSクラスタの構築

#. CloudFormationによるCI/CD環境構築

   #. CodeBuild、CodePipeLineの構築
   #. Blue-Green Deploymentの実現



+++++++




マイクロサービス関連補足記事
------------------------------------------------------------------

2. 関連記事：クラウドネイティブアプリケーションの実装
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

【趣旨】
クラウド環境でアプリケーション実装する場合に、クラウドのメリットを最大限に引き出す設計・実装方法、パターンについて、
主にAWSのサービスを活用しながら設計する方法について記述する。

【想定読者】
アプリケーション基盤開発者

[基本]

#. AmazonS3データアクセス

   #. EC2アプリケーションからのS3アクセス
   #. クライアントからのS3ダイレクトアクセス
   #. クライアントからのS3ダイレクトアップロード

#. AmazonRDSの利用
#. AmazonElasticCacheの利用
#. AmazonDynamoDBの利用
#. AWS Lambdaの利用
#. AWS SQSの利用
#. AWS CloudFrontの利用
#. AWS API Gatewayの利用
#. AWS ECSの利用

[発展]

#. AWS Cognitoの利用
#. AWS EKSの利用
#. AWS Kinesisの利用
#. AWS IoTの利用
#. AWS Cloud9の利用

3. 関連記事：マイクロサービスの設計指針
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

【趣旨】
マイクロサービスでアプリケーション実装する場合の設計指針、注意すべきポイントなどを記述する。

【想定読者】
アプリケーション基盤開発者

#. マイクロサービスアーキテクチャの12の関心事
#. Domain Driven Design実践
#. モノリシックデザインとの対比
