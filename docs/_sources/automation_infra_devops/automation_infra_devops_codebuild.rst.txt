.. include:: ../module.txt

.. _section-automation-infra-devops-codebuild-label:

基盤・デプロイ自動化実践
==================================================================

マイクロサービスアーキテクチャの基盤・デプロイ自動化
-------------------------------------------------------------------------------------------------------------------------------------

|br|

.. _section-automation-infra-devops-continuous-integration-automation-using-codebuild-2-label:

AWS CodeBuildを用いた継続的インテグレーション自動化2
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

本連載では、以下のイメージに沿って「CodeBuild」「SonarQube」を使った継続的インテグレーション(CI：Continuous integration)環境を実際に構築しています。|br|

.. figure:: img/automation_infra_devops_actual_experience/sample-continuous-integration.png


|br|

前回は、AWSから提供されている、buildspec.ymlの挙動をローカル端末で確認できるCodeBuild Local環境を構築し、環境変数を一元管理するAWS Sysmtes Managerを設定して、buildspec.ymlを作成して検証しました。
続く今回は、AWS CodeBuildを設定し、GitHubへのプッシュやプルリクエストに対して、CodeBuildを実行するよう設定してみます。

|br|

.. _section-codebuild-create-security-group-label:

事前準備：セキュリティグループの作成
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

CodeBuild設定の事前準備として、CodeBuildが実行するビルド環境のコンテナイメージに適用するセキュリティグループを、「VPC」サービスで作成しておきましょう。
ビルドに必要なソフトウェアやGitHubへソースコードを取得するための任意のアウトバウンド許可設定があれば、特にインバウンド許可は必要ありません。

|br|

.. figure:: img/automation_infra_devops_codebuild/management_console_vpc_create_security_group_1.png


|br|

.. figure:: img/automation_infra_devops_codebuild/management_console_vpc_create_security_group_2.png


|br|

.. _section-codebuild-setting-codebuild-label:

CodeBuildの設定
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

AWSコンソールメニューから、「CodeBuild」サービスを選択し、「プロジェクトの作成」ボタンを押下します。

|br|

.. figure:: img/automation_infra_devops_codebuild/management_console_codebuild_create_project_bff_1.png


|br|

CodeBuildの主な設定項目は以下の通りです。説明に記載の要領に従って入力したのち、「ビルドプロジェクトを作成する」ボタンを押下してください。
なお、各項目の必須有無は `AWS公式 CodeBuildでビルドプロジェクトを作成する <https://docs.aws.amazon.com/ja_jp/codebuild/latest/userguide/create-project.html#create-project-console>`_ にある表を参考にしてください。

|br|

.. list-table:: CodeBuildの設定
   :widths: 2, 3, 7

   * - 入力箇所
     - 項目
     - 説明

   * - プロジェクトの設定
     - プロジェクト名
     - ビルドの単位でプロジェクト名を記載します。

   * -
     - ビルドバッジ
     - ビルドバッジはビルドステータスを個別定義・明示化するオプションです。詳細は `ビルドバッジサンプル <https://docs.aws.amazon.com/ja_jp/codebuild/latest/userguide/sample-build-badges.html>`_ を参照してください。

   * -
     - 追加設定：タグ
     - AWSサービスで使用するタグの名前と値を入力します。

   * - 送信元
     - ソースプロバイダ
     - ソースコードが格納されているプロバイダを提供します。AmazonS3、CodeCommit、Bitbucket、GitHub、GitHub Enterpriseから選択可能です。以降はGitHubを選択した場合の設定要領を記載します。

   * -
     - レポジトリ
     - パブリックレポジトリか特定のGitHubアカウントのレポジトリかを選択します。特定のGitHubアカウントは初回OAuth認証により接続します。

   * -
     - GitHubレポジトリ
     - GitHubアカウントのレポジトリを選択した場合、プルダウンでレポジトリを指定します。

   * -
     - 追加設定：Gitクローンの深さ
     - コミットされた最新履歴からのバージョンの数を指定します。最新は1で、直前のコミットは2と順次続きます。全てはFULLです。

   * -
     - 追加設定：Gitサブモジュール
     - Gitのサブモジュール機能(別の外部プロジェクト)を含む場合、「Gitサブモジュールを使用する」を選択します。

   * - プライマリリソースのウェブフックイベント
     - ウェブフック
     - レポジトリへのコードプッシュ時やプルリクエスト実行時にビルドを実行する場合チェックします。詳細は次節にて詳述します。

   * - 環境
     - 環境イメージ
     - ビルドを実行するコンテナイメージを指定します。デフォルトではUbuntuが使われます。

   * -
     - ランタイム※マネージド型イメージの場合
     - ビルドコンテナの種別を選択します。Strandardのみ選択が可能です。

   * -
     - イメージ※マネージド型イメージの場合
     - コンテナイメージのバージョンを選択します。2019年6月時点では1.0もしくは2.0が選択可能です。

   * -
     - サービスロール
     - CodeBuildの実行に必要なポリシーをアタッチしたサービスロールを設定します。複数のビルドプロジェクトで関連付けが可能ですが、最大10のビルドプロジェクトまでしか関連付けができないので注意が必要です。

   * -
     - 追加設定：タイムアウト
     - ビルド処理にかかるタイムアウト時間を設定します。デフォルト1時間です。

   * -
     - 追加設定：キュータイムアウト
     - ビルドをキューに入れてからタイムアウトするまでの時間を設定します。

   * -
     - 追加設定：証明書
     - GitHub Enterpriseなどで証明書をインストールしている場合にアクセス許可するための証明書を設定します。

   * -
     - 追加設定：コンピューティング
     - ビルドするコンテナの処理スペックを選択します。

   * -
     - 追加設定：VPC
     - ビルドコンテナからアクセスするVPCを設定します。

   * -
     - 追加設定：環境変数
     - コンテナイメージへ設定したい環境変数を設定します。

   * - Buildspec
     - ビルド仕様
     - ビルド処理仕様としてbuildspec.ymlからコマンドを設定として入力するか選択します。

   * -
     - buildspec名
     - デフォルトではソースコードのルートディレクトリにあるbuildspec.ymlが選択されますが、別の名前や場所を使用している場合に入力します。

   * - アーティファクト
     - タイプ
     - ビルド実行時に生成されるアプリケーションを出力する方式を選択します。S3か、コンテナイメージを作成する場合、なしを選択します。

   * -
     - 追加設定:暗号キー
     - アーティファクトを暗号化する場合に使用されるAWS KMSカスタマーマスターキーを指定します。

   * -
     - 追加設定:キャッシュタイプ
     - アーティファクトをキャッシュする場合に選択します。S3もしくはローカルが選択可能です。

   * - ログ
     - CloudWatch Logs
     - CloudWatch Logsにビルド時の出力ログをアップロードするか選択します。

   * -
     - グループ名
     - CloudWatch Logsのグループ名を指定します。

   * -
     - ストリーム名
     - CloudWatch Logsのストリーム名を指定します。

|br|

.. figure:: img/automation_infra_devops_codebuild/management_console_codebuild_create_project_bff_2.png


|br|


.. figure:: img/automation_infra_devops_codebuild/management_console_codebuild_create_project_bff_3.png


|br|

.. figure:: img/automation_infra_devops_codebuild/management_console_codebuild_create_project_bff_4.png


|br|

.. figure:: img/automation_infra_devops_codebuild/management_console_codebuild_create_project_bff_5.png


|br|

.. figure:: img/automation_infra_devops_codebuild/management_console_codebuild_create_project_bff_6.png


|br|

.. figure:: img/automation_infra_devops_codebuild/management_console_codebuild_create_project_bff_7.png


|br|

.. figure:: img/automation_infra_devops_codebuild/management_console_codebuild_create_project_bff_8.png


|br|

早速ビルドの実行と行きたいところですが、その前にビルド用のコンテナイメージがAWS Systems Manager ParameterStoreへアクセスするための権限設定を追加で実施する必要があります。

|br|

.. _section-codebuild-iam-setting-ssm-label:

AWS Systems Manager Parameter Storeへのアクセス権限の付与
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

前回設定したAWS Systems Manager Parameter Storeで定義したデータを参照するのに、CodeBuildの実行ロールにも、アクセス権限を付与しておかなければ実行エラーとなるため、
前節で作成されるサービスロールにSystems Managerのアクセス権限を割り当てておきます。

AWSコンソールで、「IAM」サービスから「ロール」メニューを選択し、作成したポリシーに対して、以下の通り、「AmazonSSMFullAccess」のポリシーをアタッチしてください。

|br|

.. figure:: img/automation_infra_devops_codebuild/management_console_iam_attach_ssm_access.png


|br|

.. _section-codebuild-build-execution-label:

Buildの実行とWebHook設定
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

アクセス権限の付与が完了したら、早速ビルドを実行してみましょう。「ビルドの開始」ボタンを押下し、buildspec.ymlに記載された各フェーズやログが出力されることを確認します。

|br|

.. figure:: img/automation_infra_devops_codebuild/management_console_codebuild_build_execution_1.png


|br|

.. figure:: img/automation_infra_devops_codebuild/management_console_codebuild_build_execution_2.png


|br|

また、buildspec.ymlに記載した通り、Mavenビルド中に実行したSonarScannerの実行結果も、以前の連載で設定していたSonarQubeServerへ送信されるようになります。
Scannerの結果も合わせて確認し、QualityGateをパスしているか確認しましょう。

|br|

.. figure:: img/automation_infra_devops_codebuild/sonarqube-scanner-result.png


|br|

なお、SonarQubeServerもWebhookにより解析が完了したタイミングで、別の外部サービス(SlackやTrelloなど)へ処理完了通知することも可能です。
詳細は `SonarQube公式 Webhooks <https://docs.sonarqube.org/latest/project-administration/webhooks/>`_ や `API/Webhooks <https://next.sonarqube.com/sonarqube/web_api/api/webhooks>`_ を参照してください。

|br|

ビルドが正常に実行したことを確認したら、次はプルリクエストや特定のブランチに対するプッシュを契機に上記のビルド処理が実行されるよう、
CodeBuildやGitHubのWebhookの設定を行います。今回は以下のように、Git Flowをベースとしたブランチを構成し、赤い吹き出しの箇所でCodeBuildによるビルド処理が実行されるように設定するものとします。

|br|

.. figure:: img/automation_infra_devops_codebuild/branch_strategy.png


|br|

図のイメージの構成では、masterブランチをベースにdevelopブランチを作成し、またdevelopブランチから、featureブランチを作成しています。
Git Flowのブランチモデルに従って、機能開発を各々featureブランチに対しておこなう想定とし、feature/＊＊＊＊＊となるブランチに対して、プッシュが行われたタイミングで、そのコミットによるソースコードの追加・変更が問題ないかビルドとテストを実行するように設定します。
自らの開発対象であるfeatureブランチはもちろんですが、並行して行われる他のfeatureブランチの変更を、プルリクエスト(PullRequest:PR)されたdevelopブランチを経由で変更を取り込みながら、ビルド・テストが問題なく実行終了するか、その担保をとってから開発を進めることができるようになります。
なお、developブランチに対するPRによるビルド処理は、次回以降「CodePipelineを使った継続的デリバリー」で扱います。

引き続き、GitHub上で上記イメージの構成通り、ブランチを作成して、ローカル端末へフェッチしていきます。住所系とメール系の機能を拡充するためのfeature/addressとfeature/emailを作成します。

|br|

.. figure:: img/automation_infra_devops_codebuild/github_create_branches.png


|br|

ブランチ作成後は、ローカルへフェッチして、ブランチをチェックアウトし、feature/addressブランチへ切り替えましょう。以下はターミナルコマンドなどで実行する例です。

.. sourcecode:: bash

   > git fetch
   > git checkout -b feature/address origin/feature/address

|br|

住所系機能のベースモジュールを追加で実装しましょう。取り急ぎ、先に `住所系機能で追加実装した結果のリンク <https://github.com/debugroom/mynavi-sample-continuous-integration/commit/a0bd03a7d9066d9570377d415218ef4207889375>`_ を載せて起きますが、
コミットしてプッシュする前に、CodeBuildで以下の通り、「feature/xxxxxxブランチにプッシュが行われた場合、ビルドを実行する設定」を行います。

|br|

「CodeBuild」サービスから、「ビルドプロジェクト」メニューを選択し、前節で作成したプロジェクトを選択して、「ビルドの詳細」タブにある、「プライマリソースのウェブフックイベント」の「編集」ボタンを押下します。

|br|

.. figure:: img/automation_infra_devops_codebuild/management_console_codebuild_edit_webhook_backend_1.png


|br|

「送信元」の変更はしませんが、プライマリソースのウェブフックイベントを以下の通り設定します。

* 「コードの変更がこのレポジトリにプッシュされるたびに再構築する」にチェック
* 「イベントタイプ」には「プッシュ」を設定。
* 「これらの条件でビルドを開始する」において、「HEAD_REF」オプションに、正規表現「^refs/heads/feature/.*」を設定

|br|

.. figure:: img/automation_infra_devops_codebuild/management_console_codebuild_edit_webhook_backend_2.png


|br|

.. figure:: img/automation_infra_devops_codebuild/management_console_codebuild_edit_webhook_backend_3.png


|br|

上記の手順は、Backend、BFFなどCodeBuildを作成しているプロダクト単位に設定を行います。なお、設定の詳細は、
`AWS公式の「CodeBuildのGitHubプルリクエストとウェブフックフィルタのサンプル」 <https://docs.aws.amazon.com/ja_jp/codebuild/latest/userguide/sample-github-pull-request.html#sample-github-pull-request-filter-webhook-events-console>`_ も適宜参照してください。

設定完了後、GitHubのfeature/addressブランチに対してプッシュしたのち、CodeBuildが実行されることを確認します。

|br|

.. figure:: img/automation_infra_devops_codebuild/github_push_add_feature_address.png


|br|

.. figure:: img/automation_infra_devops_codebuild/management_console_codebuild_start_webhook_bff.png


|br|

.. _section-codebuild-conclusion-label:

マイクロサービスにおける継続的インテグレーションのまとめ
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

以上、ここまでの連載で、ECSコンテナ、RDSを使ったSonarQubeServerによる静的チェック環境の構築から、SpringBootを使ったマイクロサービスにおける各種テストコードの実装、
GitHubへプッシュされたソースコードに対するCodeBuildを使ったビルド・テスト、SonarScannerによる静的チェック結果の可視化といった一連の継続的インテグレーション自動化の仕組みを解説してきました。
今回は触れていませんが、より上位のオプションとして、SonarQubeServer Developer Editionを使えば、静的チェックの結果をレビューコメントとしてGitHubへ反映することも可能です(なお、ComunityEditionでもGitへ連携するプラグインはありますが、現時点で既に非推奨となっているため、本連載では除外しています)。

SonarQubeServerではALB、ECS、RDSを使用して環境構築することにより、自動バックアップなどの保守性や耐障害性も高まります。
また、クラウドをベースとしたCodeBuildでマシンリソースを気にすることなく、静的解析、ビルド・テスト実行を自動化し、迅速に問題検出しながら、複数人での開発をスピーティかつ効率的に進めていくことができます。
マイクロサービスアーキテクチャをベースとしたアプリケーション開発では、そのアジリティを高めるために、こうしたクラウドの利点をフル活用した、継続的インテグレーションの自動化環境が求められます。

ただし、今回、SpringBootをベースとしたWebアプリケーションで実装したSeleiumベースのE2Eテストは、バックエンドのマイクロサービスが起動している必要があるため、ビルド時に実行しないように設定しています。
本来であれば、以降、E2Eテストを含め、以下のようなプロセス例でプロダクション環境へのリリースを検討します。

#. バックエンドのマイクロサービスのコンテナイメージのビルド、DockerHubへのプッシュ
#. マイクロサービスコンテナイメージをプロダクションとほぼ同等のステージング環境へデプロイ
#. Webアプリケーション(BFF)のE2Eテストとして、Seleniumテストコード実行
#. Webアプリケーション(BFF)のコンテナイメージのビルド、DockerHubへのプッシュ
#. Webアプリケーション(BFF)のステージング環境へデプロイ
#. パフォーマンステストやセキュリティテスト、受入等のその他テスト後の管理者によるリリース承認
#. ステージング環境でテスト済みのコンテナイメージをリリース用にタグ付け、DockerHubへのプッシュ
#. プロダクション環境へリリース

次回以降、AWS CodePipelineを使って、上記をパイプライン的に自動化する仕組みを構築・解説していきます。

|br|


著者紹介
------------------------------------------------------------------

川畑 光平(KAWABATA Kohei) - NTTデータ 課長代理

.. figure:: img/automation_infra_devops_overview/pic_image01.jpg


金融機関システム業務アプリケーション開発・システム基盤担当を経て、現在はソフトウェア開発自動化関連の研究開発・推進に従事。

Red Hat Certified Engineer、Pivotal Certified Spring Professional、AWS Certified Solutions Architect Professional等の資格を持ち、アプリケーション基盤・クラウドなど様々な開発プロジェクト支援にも携わる。

`2019 APN AWS Top Engineers & Ambassadors <https://aws.amazon.com/jp/blogs/psa/japan-apn-ambassador-2019/>`_ 選出。
