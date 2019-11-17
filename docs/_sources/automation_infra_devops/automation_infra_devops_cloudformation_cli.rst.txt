.. include:: ../module.txt

.. _section-automation-infra-devops-cloudformation-2-label:

基盤・デプロイ自動化実践
==================================================================

マイクロサービスアーキテクチャの基盤・デプロイ自動化
-------------------------------------------------------------------------------------------------------------------------------------

|br|

本連載では、以下のイメージの構成のようなマイクロサービスアーキテクチャにおける、基盤自動化環境の構築を実践していきます。

|br|

.. figure:: img/automation_infra_devops_overview/MicroServiceArchitecture.png


|br|

前回は、CloudFormationの概要やテンプレート記述の要領、様々な機能を概説しました。続く今回からは実際にCloudFormationテンプレートを作成していく上で必要な開発環境を構築していきます。
前回も説明しましたが、CloudFormationでは、定められた形式に従って記述されたテンプレートファイルを、AWSコンソール上もしくはCLI(CommandLineInterface)を使って実行します。
しかし、単純な正攻法で、テキストエディタを使って、一からテンプレートを作成し、コンソール上からテンプレートをアップロードして、動作を確認するといったステップを踏んでいると、
文法誤りやタイプミス、論理的な矛盾、スタック構築順序誤りなど様々な人為的なエラーにより、期待通り動作するまで至るにはかなりの労力・時間を要します。

そうした労力を少しでも軽減するために、以下のような開発環境でテンプレートを記述することを推奨します。

#. AmazonCLI(CommandLineInterface)の実行環境/AWS認証情報の設定
#. テンプレートのコード補完・文法チェック等を実行する各種プラグインインストール
#. 統合開発環境(IntelliJ IDEA)の設定
#. CloudFormation実行ヘルパースクリプトの作成

1点目として、CLIを用いることにより、コンソール上でファイルをアップロードするやり方に比べて、格段に迅速にCloudFormationを繰り返し実行できるようになります。
(2)のプラグインはソースコードを記述している際の、記法の誤りや、プロパティ名の妥当性、階層構造チェックなどを実行するため、テンプレート記述の正確性が向上します。
これらのプラグインはIntelliJ IDEAで多くサポートされており、他のアプリケーション開発言語と同様の統合開発環境で行うことができるので、不要なエディタの切り替えなど発生せず作業生産性が向上します(3)。
また、CLIによるCloudFormationの実行ですが、コマンド自体は単純であるものの、入力パラメータも多く、長くなりがちなため、最後に(4)手軽に実行できるヘルパー用のシェルスクリプト等を作成し、
統合開発環境上で実行できるようにしておいた方がよいでしょう。なお、CLIからのCloudFormation実行は、これまでのAWSサービスをローカルで実行してきた手法と同様、
認証情報が必要になります。加えて、CloudFormationでスタックとして構築する全てのAWSリソースで必要な権限を設定しておく必要があります。

|br|

.. _section-amazon-cli-overview-label:

AmazonCLI(CommandLineInterface)の概要と実行環境設定
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

AWS環境・サービスを使用するには大きく3つの方法があります。AWSコンソールによる操作、SDK(SoftwareDevelopmentKit：AWSにより提供される各プログラム言語ごとにあるライブラリ)を介したアクセス、
そして、AmazonCLIを介したコマンドラインの実行です。ローカル端末等にCLIのインストールを行い、ターミナルなどのコマンド操作ツールを通して、awsコマンドに実行サービス名やパラメータを与えて実行します。
GUIと比べ、シェルスクリプトなどと組み合わせることにより、作業の迅速化、自動実行、作業ミスの防止が図れるのがメリットです。

CLIをインストールするには、インストール対象のマシンがPython2.6.5以上、もしくはPython3.3以上をインストール済みである必要があります。
ここでは、ローカルマシンとしてMacOSにCLIをインストールする手順を紹介します。`公式サイト AWS CLIのインストール <https://docs.aws.amazon.com/ja_jp/cli/latest/userguide/cli-chap-install.html>`_ では、
CLIのインストールはpipコマンドを用いてインストールを行なっているため、事前にpythonを実行できる環境を構築しておきましょう。

|br|

.. note::

   MacOS Sierra以降はHomebrewからpythonをインストールする手段が簡易です。以下のコマンドにより、標準インストールされているpython(/usr/bin/python)ではなく、/usr/local/bin/pythonが使用されるようになります。

   .. sourcecode:: bash

      brew update
      brew install python

|br|

pipを利用してCLIをインストールするには以下のコマンドを実行します。

|br|

.. sourcecode:: bash

   pip3 install awscli --upgrade --user

|br|

インストール後に.bash_profileや.bashrcにインストールしたawsコマンドがあるパスを通しておきましょう。

|br|

.. sourcecode:: bash

   export PATH="/Users/XXXXXXXX/Library/Python/3.6/bin/:$PATH"

|br|

sourceコマンドによる.bash_profileの再読み込みやターミナルの再起動後、以下のコマンドが正常実行できることを確認します。

.. sourcecode:: bash

   aws --version
   aws-cli/1.16.241 Python/3.6.5 Darwin/18.7.0 botocore/1.12.231

|br|

また、CLIのコマンド実行により、AWSへアクセスするにはアクセスしたいサービスの権限を付与したユーザの作成および、認証情報の設定が必要になります。
今回のようにCloudFormationをローカル端末から使用する場合は、CloudFormationアクセス許可ポリシーをアタッチして権限をしたユーザの認証情報をローカル端末のホームフォルダ配下の.aws/credentialsに置いておきます。
ユーザや認証情報をIAMを使って作成する実際の手順は `Amazon RDSにアクセスするSpringアプリケーション(1) ユーザ／認証情報の作成 <https://news.mynavi.jp/itsearch/article/devsoft/4422>`_ に記載していますが、
これ同様の要領で、ユーザに「AmazonCloudFormationFullAccess」ポリシーを付与して、認証情報をローカル端末へ設定しておきましょう。
また、CloudFormationで構築するスタックのAWSリソースに対してもアクセス権限を付与しておく必要があるので注意が必要です。

|br|

.. figure:: img/automation_infra_devops_cloudformation/management_console_iam_attach_cloudformation_access.png

|br|

.. note:: aws configureコマンドを使うと、.aws/配下にある認証情報や設定を行うこともできます。コマンド実行後認証キーやリージョンなど対話形式で設定を行えます。



以上で、CLIおよびCloudFormationをローカル端末上から実行できる準備が整いました。次回はテンプレートを効率的に実装するためのcfn-python-lintプラグインや統合開発環境として利用するIntelliJ IDEAの設定を行います。

|br|

著者紹介
------------------------------------------------------------------

川畑 光平(KAWABATA Kohei) - NTTデータ 課長代理

.. figure:: img/automation_infra_devops_overview/pic_image01.jpg


金融機関システム業務アプリケーション開発・システム基盤担当を経て、現在はソフトウェア開発自動化関連の研究開発・推進に従事。

Red Hat Certified Engineer、Pivotal Certified Spring Professional、AWS Certified Solutions Architect Professional等の資格を持ち、アプリケーション基盤・クラウドなど様々な開発プロジェクト支援にも携わる。

`2019 APN AWS Top Engineers & Ambassadors <https://aws.amazon.com/jp/blogs/psa/japan-apn-ambassador-2019/>`_ 選出。
