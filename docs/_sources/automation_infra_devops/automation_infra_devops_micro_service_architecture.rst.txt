.. include:: ../module.txt

【第4回】マイクロサービスにおけるテスト自動化とテスト戦略
------------------------------------------------------------------

.. _section-application-test-overview-for-microservice-label:

マイクロサービスにおけるテスト自動化とテスト戦略の重要性
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

連載第１回「 `マイクロサービスのアプリケーション構成と構築 <https://news.mynavi.jp/itsearch/article/devsoft/4379>`_ 」で、
マイクロサービスの一般的な特徴やメリットを述べましたが、実際のプロジェクトでマイクロサービスアーキテクチャを採用する場合、
アプリケーションを迅速にリリースし、顧客やエンドユーザの要求・フィードバックに応じて、機能追加や仕様変更を行っていくプロジェクト特性であるケースが多いと思います。

機能追加等でアプリケーションを改修する場合、追加した部分のテストはもちろんリファクタリング、変更が入っていない既存部分でも無影響確認テストなどを
繰り返し実施する必要があります。そのため、試験はテストコードを実装して、ソースコードコミットやビルド時にテストを実行するなどして
改修時の問題の早期検出の仕組みを整え、継続的インテグレーションで極力自動化し、修正にかかる工数を省力化しておくことが重要になります。

とはいえ、テストコードの実装には相応の工数が必要になります(筆者の感覚的ですが、アプリケーションコードの実装に加えてテストコードの実装は3倍近い工数がかかる場合もあります)。
初めから完璧にテストコードを整備しておく必要もありませんし、必要以上のテストコードの実装でかえって開発のアジリティを損なうようでは本末転倒です。
ただ、テストが疎かになるとせっかくの継続的インテグレーションも機能しません。
開発のスピードと品質を両立するために、テスト計画やスコープ、検証の観点を明示的に策定しておくことが重要です。
次節以降はどのようなテスト戦略で、どのようなテストを実施すべきか、SpringBootを使ったアプリケーションテスト実装を通して実践していきます。

|br|

.. _section-application-component-for-microservice-label:

アプリケーションのパッケージ・コンポーネント構成
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

テスト戦略について述べる前に、本連載で構築するマイクロサービスアーキテクチャでのアプリケーションのパッケージ・コンポーネント構成について説明します。
以下の通り、アプリケーションはBackendとBackendForFrontend(BFF)に分けてコンテナを構築しています。

|br|

.. figure:: img/automation_infra_devops_overview/MicroServiceArchitecture.png


|br|

本連載では、Backendでデプロイしているアプリケーションがマイクロサービスに相当し、各マイクロサービスを呼び出し、
呼び出した結果をフロントエンド(クライアント)向けに画面(HTML)などにアグリゲーション(集約)して返却するのが
BFFの役割としています。また、Backend、BFFともにアプリケーションのコンポーネントをアプリケーションレイヤ、ドメインレイヤ、インフラストラクチャレイヤで分割して各々構成するものとします。
以降登場する各レイヤやコンポーネントの用語の意味や役割は `TERASOLUNAのガイドライン「アプリケーションのレイヤ化」 <http://terasolunaorg.github.io/guideline/5.5.1.RELEASE/ja/Overview/ApplicationLayering.html>`_
での記述にほぼ準拠しますので、必要に応じて、リンクを参照してください。

なお、実際に作成したアプリケーションは `GitHub <https://github.com/debugroom/mynavi-sample-continuous-integration>`_ 上にコミットしています。以降、解説で使用しているソースコードでは、記述を一部省略しているので、実行コードを作成する場合は、必要に応じて適宜GitHub上のソースコードも参照してください。

|br|

まず、マイクロサービスとなるBackendアプリケーションのパッケージ・コンポーネント構成は以下の通りです。
なお、アプリケーション内のデータベースアクセスはSpring Data JPAを用いており、ここでは詳細な解説は行いませんが、詳細は
`連載 AWSで作るクラウドネイティブアプリケーションの基本「RDSへアクセスするSpringアプリケーション」 <https://news.mynavi.jp/itsearch/article/devsoft/4426>`_ や `TERASOLUNAのガイドライン「データベースアクセス（JPA編）」 <http://terasolunaorg.github.io/guideline/5.5.1.RELEASE/ja/ArchitectureInDetail/DataAccessDetail/DataAccessJpa.html>`_
を適宜参照してください。また、テストではインメモリDBであるHSQLを利用します。

|br|

.. sourcecode:: bash

   [backend]
     └src
       └main
         ├java
         │ └org
         │   └debugroom
         │     └mynavi
         │       └sample
         │         └continuous
         │           └integration
         │             └backend
         │               └app                                  ... アプリケーション層のパッケージ
         │               │ └model                              ... リクエストパラメータのモデルクラスパッケージ
         │               │ │ ├Xxxxx.java                       ... 入力チェックルール等が定義されるモデルクラス
         │               │ │ └XxxxxMapper.java                 ... ドメイン層のモデルクラスと相互変換するマッパークラス
         │               │ └web                                ... MvcConfigでコンポーネントスキャンの対象とするパッケージ
         │               │   └BackendController.java           ... リクエストハンドリング・ドメインサービス呼び出して、Resourceを返却するコントローラクラス
         │               └domain                               ... ドメイン層のパッケージ
         │               │ └model
         │               │ │ └entity                           ... JPAConfigでスキャン対象とするエンティティクラスパッケージ
         │               │ │   └Xxxxx.java                     ... JPAエンティティクラス
         │               │ ├repository                         ... JPAConfigでスキャン対象とするレポジトリクラスパッケージ
         │               │ │ ├specification                    ... JPAでテーブル結合等の条件を指定するクラスパッケージ
         │               │ │ │ └Xxxxx.java                     ... JPAでテーブル結合等の条件を指定するクラス
         │               │ │ └XxxxxRepository.java             ... レポジトリインタフェースクラス
         │               │ └service                            ... DomainConfigでコンポーネントスキャンの対象とするサービスクラスパッケージ
         │               │   ├SampleService.java               ... DBへ基本的なCRUDアクセスを行うサービスインタフェースクラス
         │               │   ├SampleServiceImpl.java           ... SampleServiceの実装クラス
         │               │   ├SampleOneToOneService.java       ... 1対1の関連をもつテーブルアクセスを行うサービスインタフェースクラス
         │               │   ├SampleOneToOneServiceImpl.java   ... SampleOneToOneServiceの実装クラス
         │               │   ├SampleOneToManyService.java      ... 1対多の関連をもつテーブルアクセスを行うサービスインタフェースクラス
         │               │   ├SampleOneToManyServiceImpl.java  ... SampleOneToOneServiceの実装クラス
         │               │   ├SampleManyToManyService.java     ... 多対多の関連をもつテーブルアクセスを行うサービスインタフェースクラス
         │               │   └SampleManyToManyServiceImpl.java ... サービス実装クラス
         │               └config                               ... 設定クラス用のパッケージ
         │                   ├App.java                         ... アプリケーション起動クラス
         │                   ├DevConfig.java                   ... 開発環境固有の設定クラス
         │                   ├DomainConfig.java                ... ドメイン層に関する設定クラス
         │                   ├JPAConfig.java                   ... JPA設定クラス
         │                   └MvcConfig.java                   ... アプリケーション層に関する設定クラス
         └resources
           ├application.yml                                    ... アプリケーション設定ファイル
           └application-dev.yml                                ... プロファイル"dev"で有効になるアプリケーション設定ファイル

|br|

また、マイクロサービスを呼び出すBFFアプリケーション(Webアプリケーションとなります)のパッケージ・コンポーネント構成は以下の通りとします。
WebアプリケーションのHTMLテンプレートエンジンとしてThymeleafを使用しています。こちらも詳細な説明は割愛しますが、必要に応じて、`Thymeleaf公式ドキュメント <https://www.thymeleaf.org/documentation.html>`_ や
`Macchinetta Framework テンプレートエンジン <https://macchinetta.github.io/server-guideline-thymeleaf/current/ja/ArchitectureInDetail/WebApplicationDetail/Thymeleaf.html>`_ を参照してください。

|br|

.. sourcecode:: bash

   [backend for frontend]
     └src
       └main
         ├java
         │ └org
         │   └debugroom
         │     └mynavi
         │       └sample
         │         └continuous
         │           └integration
         │             └bff
         │               └app                                      ... アプリケーション層のパッケージ
         │               │ └model                                  ... リクエストパラメータのモデルクラスパッケージ
         │               │ │ ├XxxxxForm.java                       ... HTMLフォームを表現するモデルクラス
         │               │ │ ├Xxxxx.java                           ... 入力チェックルール等定義するモデルクラス
         │               │ │ └XxxxxMapper.java                     ... commonプロジェクトにあるResourceクラスと相互変換するマッパークラス
         │               │ └web                                    ... MvcConfigでコンポーネントスキャンの対象とするパッケージ
         │               │   └BackendForFrontendController.java    ... リクエストハンドリング・ドメインサービス呼び出し後、テンプレートViewへ遷移するコントローラクラス
         │               └domain                                   ... ドメイン層のパッケージ
         │               │ ├repository                             ... レポジトリクラスパッケージ
         │               │ │ ├XxxxxResourceRepository.java         ... Resourceレポジトリインタフェースクラス
         │               │ │ └XxxxxResourceRepositoryImpl.java     ... マイクロサービスへアクセスするRestClientを使用したレポジトリ実装クラス
         │               │ └service                                ... DomainConfigでコンポーネントスキャンの対象とするサービスクラスパッケージ
         │               │   ├SampleService.java                   ... シンプルに１つのマイクロサービスにアクセスするサービスクラス
         │               │   ├SampleServiceImpl.java               ... SampleServiceの実装クラス
         │               │   ├OrchestrateService.java              ... マイクロサービスに複数回アクセスするサービスクラス
         │               │   └OrchestrateServiceImpl.java          ... OrchestrateServiceインタフェースの実装クラス
         │               └config                                   ... 設定クラス用のパッケージ
         │                   ├WebApp.java                          ... Webアプリケーション起動クラス
         │                   ├DomainConfig.java                    ... ドメイン層に関する設定クラス
         │                   └MvcConfig.java                       ... アプリケーション層に関する設定クラス
         └resources
           ├static                                                 ... 静的リソースフォルダ
           │ ├css                                                  ... CSSフォルダ
           │ │ ├Xxxxx.css                                          ... 各ページごとのCSSファイル(幅1280px以上)
           │ │ ├Xxxxx_mobile.css                                   ... 各ページごとのCSSファイル(幅320px以上)
           │ │ └Xxxxx_tablet.css                                   ... 各ページごとのCSSファイル(幅768px以上)
           │ └js                                                   ... JavaScriptフォルダ
           │   └Xxxxx.js                                           ... 各ページごとのJSファイル
           ├template                                               ... Thymeleaf用テンプレートフォルダ
           │ └Xxxxx.html                                           ... ThymeleafテンプレートHTML
           ├application.yml                                        ... アプリケーション設定ファイル
           ├application-dev.yml                                    ... プロファイル"dev"で有効になるアプリケーション設定ファイル
           ├messages.properties                                    ... デフォルトメッセージ定義ファイル
           ├messages_ja.properties                                 ... ロケールがjaの際に有効になるメッセージ定義ファイル
           └ValidationMessages.properties                          ... 入力チェックエラーメッセージ定義ファイル

|br|

また、２つのアプリケーションで共通に使用する部品のパッケージ・コンポーネント構成は以下の通りです。
主な構成要素としては、マイクロサービスから返却されるResourceクラスと、共通的に利用するアプリケーション基盤部品ですが、
ほとんどがマイクロサービスと連携する場合やテストコードの検証に必要になったために実装した共通部品です。なお、
例外の種類や考え方、SpringFrameworkを用いた例外ハンドリングの基本はTERASOLUNAガイドライン `例外ハンドリング <http://terasolunaorg.github.io/guideline/5.5.1.RELEASE/ja/ArchitectureInDetail/WebApplicationDetail/ExceptionHandling.html>`_ や、
`RESTful Web Serviceの例外ハンドリング <http://terasolunaorg.github.io/guideline/5.5.1.RELEASE/ja/ArchitectureInDetail/WebServiceDetail/REST.html#resthowtouseexceptionhandling>`_ も参考にしてください。

|br|

.. sourcecode:: bash

   [common]
     └src
       └main
         ├java
         │ └org
         │   └debugroom
         │     └mynavi
         │       └sample
         │         └continuous
         │           └integration
         │             └common
         │               └web
         │               │ └model
         │               │   └XxxxxResoure.java                   ... 各エンティティに対応するResourceクラス
         │               └apinfra                                 ... アプリケーション基盤部品パッケージ
         │                 ├exception                             ... BackendとBFF双方のアプリケーション間で連携する例外クラスパッケージ
         │                 │ ├BusinessException.java              ... ビジネス例外クラス
         │                 │ ├BusinessExceptionResponse.java      ... マイクロサービスからレスポンスするビジネス例外をラップする実装クラス
         │                 │ ├CommonExceptionHandler.java         ... 双方のアプリケーションのコントローラで例外ハンドリングする共通クラス
         │                 │ ├ErrorResponse.java                  ... マイクロサービスからレスポンスする共通例外インタフェースクラス
         │                 │ ├SystemException.java                ... システム例外クラス
         │                 │ ├SystemExceptionResponse.java        ... マイクロサービスからレスポンスするシステム例外をラップする実装クラス
         │                 │ ├ValidationError.java                ... マイクロサービスで発生した入力チェックエラーを連携するクラス
         │                 │ ├ValidationErrorMapper.java          ... BindingResultとValidationErrorに変換するクラス
         │                 │ └ValidationErrorResponse.java        ... マイクロサービスからレスポンスする入力チェック例外をラップする実装クラス
         │                 ├test
         │                 │ └junit                               ... JUnitテストで利用する部品パッケージ
         │                 │   ├BusinessExceptionMatcher.java     ... ビジネス例外の検証に使用するカスタムMatcherクラス
         │                 │   ├SystemExceptionMatcher.java       ... システム例外の検証に使用するカスタムMatcherクラス
         │                 │   ├UnitTest.java                     ... 単体テストの識別子として利用するインタフェースクラス
         │                 │   ├IntegrationTest.java              ... 結合テストの識別子として利用するインタフェースクラス
         │                 │   └E2ETest.java                      ... EndToEndテストの識別子として利用するインタフェースクラス
         │                 └util
         │                   └DateUtil.java                       ... 日付に関するユーティリティクラス
         └resources
           ├messages.properties                                   ... ビジネス・システムエラー等のメッセージ定義ファイル
           └messages_ja_JP.properties                             ... ロケールjp_JPで有効になるメッセージ定義ファイル

|br|

次節以降のテストコード実装の解説時に、上記の幾つかのコンポーネントについては必要に応じて説明する場合もありますが、
ここで示しておきたいことは、Backend、BFFともにController、Service、Repositoryがアプリケーションの中核的なコンポーネントだということです。
SpringBootをベースとするアプリケーションでは、提供されているアノテーションの関係上、同じようなコンポーネント構成をとるケースがほとんどですので、
以降は、Controller、Service、Repositoryに関するテストを前提として話を進めていきます。

|br|


.. _section-test-strategy-for-microservice-label:

アプリケーションのテスト観点
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

上記のように構成した、マイクロサービス(Backend)、Webアプリケーション(BFF)のコンポーネント構成では、以下のような観点で各テストごとに品質を担保していくのがベターです。

|br|

.. figure:: img/automation_infra_devops_test/microservice-test-scope.png


|br|

.. _命名規約によるSQLクエリの自動組立: http://terasolunaorg.github.io/guideline/5.5.1.RELEASE/ja/ArchitectureInDetail/DataAccessDetail/DataAccessJpa.html#how-to-specify-query-mathodname-label

.. list-table::
   :widths: 3, 3, 3, 11

   * - アプリケーション
     - 試験
     - コンポーネント
     - 検証観点

   * - マイクロサービス |br| (Backend)
     - 単体試験
     - Respository
     - ・エンティティクラスがテーブル定義と一致しているか |br| ・O/Rマッピング設定が妥当か |br| ・記載したSQLクエリや集合関数が正しく実行されるか |br| ・該当しないデータが発生した場合に期待された戻り値が返されるか |br| ・ `命名規約によるSQLクエリの自動組立`_ が正しく実行されるか |br| ・指定した結合条件でデータが正しく取得できるか

   * -
     -
     - Service
     - ・Service実行の結果、正しくアウトプットが返されるか |br| ・Service実行の結果、正しくビジネス例外が返されるか |br| ・例外に正しくメッセージが設定されているか

   * -
     -
     - Controller
     - ・指定したHTTPメソッドやURLで正しくリクエストハンドリングされるか |br| ・リクエストパラメータやパス変数が正しくマッピングされるか |br| ・入力チェックが正しく行われているか |br| ・入力チェックエラーやビジネスエラー発生時に正しいHTTPステータスを返却するか |br| ・入力チェックエラーやビジネスエラー発生時に正しいメッセージやパラメータを返却するか |br| ・レイヤ間のモデルオブジェクト変換は正しくマッピングされるか

   * -
     - 結合試験
     - Service⇔Repository
     - ・データベースから正しく値が取得できるか |br| ・データベースへ正しくデータが反映できるか |br| ・設定ファイルが正しく動作するか

   * -
     -
     - Controller⇔Service⇔Repository
     - ・期待したレスポンスが返却されるか |br| ・モデル間のデータマッピングが正しく実行されているか |br| ・設定ファイルが正しく動作するか

   * - Webアプリケーション |br| (BFF)
     - 単体試験
     - Respository(RestClient)
     - ・正しくビジネス例外が返されるか |br| ・異常なレスポンスを受け取った場合正しくシステム例外が返されるか |br| ・例外に正しくメッセージが設定されるか |br|・マイクロサービス側のサーバエラー発生時に正しくシステム例外が返されるか

   * -
     -
     - Service
     - ・Service実行の結果、正しくアウトプットが返されるか |br| ・Service実行の結果、正しくビジネス例外が返されるか |br| ・例外に正しくメッセージが設定されているか

   * -
     -
     - (View⇔)Controller
     - ・指定したHTTPメソッドやURLで正しくリクエストハンドリングされるか |br| ・リクエストパラメータやパス変数が正しくマッピングされるか |br| ・入力チェックが正しく行われているか |br| ・入力チェックやビジネスエラー発生時に正しいメッセージやパラメータを返却するか |br| ・サービス実行結果が正しく画面に表示されるか |br|  ・非同期通信の実行結果が正しく画面に表示されるか

   * -
     - EndToEndテスト
     - [BFF] ⇔ [Backend]
     - ・ユースケースシナリオ通り操作した時に、正しく画面に結果が表示されるか |br|  ・ユースケースシナリオ通り操作した時に、エラーメッセージが正しく表示されるか |br|・データベースへ正しくデータが反映できるか |br|  ・画面表示のレイアウトが崩れていないか |br| ・ブラウザごとに表示が異なっていないか

|br|

次回以降は、上記の観点に従い、SpringBootを使って、ControllerやService、Repositoryのテストの実装方法を解説していきます。

|br|

著者紹介
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

川畑 光平(KAWABATA Kohei) - NTTデータ 課長代理

.. figure:: img/automation_infra_devops_overview/pic_image01.jpg


金融機関システム業務アプリケーション開発・システム基盤担当を経て、現在はソフトウェア開発自動化関連の研究開発・推進に従事。

Red Hat Certified Engineer、Pivotal Certified Spring Professional、AWS Certified Solutions Architect Professional等の資格を持ち、アプリケーション基盤・クラウドなど様々な開発プロジェクト支援にも携わる。
