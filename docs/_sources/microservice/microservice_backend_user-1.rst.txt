.. include:: ../module.txt

.. _section-aws-microservice-backend-user-1-label:

【第5回】バックエンドマイクロサービスの実装
===================================================================================

|br|

本連載では、以下に示すようなマイクロサービスアーキテクチャのアプリケーション環境を構築しています。

|br|

.. figure:: img/webapp/service-architecture.png

|br|


前回はSpringSecutiyを使った様々なカスタム設定やWebアプリケーションログインページの実装の解説を行い、実際にアプリケーションを起動してみました。
今回はバックエンドにユーザ情報をリソースとして取り扱うマイクロサービスを作成します。

本連載で実際に作成するアプリケーションでは `GitHub <https://github.com/debugroom/mynavi-sample-aws-microservice/tree/feature_2-backend-user-service>`_ 上にコミットしています。
なお、前回の実装とブランチを変更しているので参照する際は留意してください。以降に記載するソースコードでは、import文など本質的でない記述を省略している部分があるので、実行コードを作成する際は、必要に応じて適宜GitHubにあるソースコードも参照してください。

|br|

バックエンドマイクロサービスの実装
------------------------------------------------------------------

|br|

バックエンドサブネットに構築するマイクロサービスですが、連載「AWSで実践！基盤・デプロイ自動化」の連載 `第4回 <https://news.mynavi.jp/itsearch/article/devsoft/4475>`_ と同様のプロジェクト・パッケージ構成で、
シンプルなRESTfulなアプリケーションとして作成します。マイクロサービスから共通的に利用するcommonプロジェクトも合わせて作成し、Mavenのpom.xmlから参照設定を行っています。

アプリケーションコンポーネントのレイヤ構成は第4回と同様、 `TERAOLUNAのガイドライン 「アプリケーションのレイヤ化」 <http://terasolunaorg.github.io/guideline/5.6.0.RELEASE/ja/Overview/ApplicationLayering.html>`_ の考え方に準拠するものとします。
今回実装するユーザリソースを返却するバックエンドマイクロサービスのコンポーネント構成は以下の通りです。

|br|

.. list-table:: バックエンドマイクロサービスのコンポーネント構成
   :widths: 3, 6, 1

   * - コンポーネント
     - 説明
     - 必須

   * - App
     - SpringBootアプリケーションを実行する起動クラス
     - ◯

   * - MvcConfig
     - SpringMVCの設定を行うクラス
     - ◯

   * - JpaConfig
     - JPAおよびSpringDataJPAの設定クラス
     - ◯

   * - DomainConfig
     - ServiceやRepository等ドメイン層の設定クラス
     - ◯

   * - DevConfig
     - プロファイル「dev」におけるHSQLの環境構築やデータソース接続設定クラス
     - ◯

   * - BackendUserController
     - ユーザリソースを返すRestController
     - ◯

   * - XxxxResourceMapper
     - リソースとエンティティクラスを変換するMapper
     -

   * - SampleService(Impl)
     - データベースへアクセスするRepositoryなどを使ってユーザエンティティクラスを返却するサービスクラス
     - ◯

   * - UserRepository
     - ユーザテーブルへアクセスするためのSpringDataJPAにおけるRepositoryインターフェースクラス
     - ◯

   * - User
     - ユーザを表すエンティティクラス
     - ◯

   * - Credential・CredentialPK
     - 認証を表すエンティティクラスおよびプライマリキークラス
     - ◯

|br|

今回の実装でポイントとなる箇所を説明します。まずローカル環境で起動するアプリケーションをプロファイル「dev」で実行する構成とします。devプロファイル環境ではインメモリDBであるHSQLを利用します。

|br|

.. sourcecode:: java
   :caption: DevConfig

   package org.debugroom.mynavi.sample.aws.microservice.backend.user.config;

   // omit

   @Profile("dev")
   @Configuration
   public class DevConfig {

       @Bean
       public DataSource dataSource(){
           return (new EmbeddedDatabaseBuilder())
                .setType(EmbeddedDatabaseType.HSQL)
                .addScript("classpath:schema-hsql.sql")
                .addScript("classpath:data-hsql.sql")
                .build();
       }
   }

|br|

アプリケーション内のデータベースアクセスはSpring Data JPAを用いており、ここでは詳細な解説は行いませんが、必要に応じて 連載「AWSで作るクラウドネイティブアプリケーションの基本」の `第12回 <https://news.mynavi.jp/itsearch/article/devsoft/4426>`_ や
TERASOLUNAのガイドライン「 `データベースアクセス（JPA編） <http://terasolunaorg.github.io/guideline/5.6.0.RELEASE/ja/ArchitectureInDetail/DataAccessDetail/DataAccessJpa.html>`_ 」 を参照してください。
なお、返却するユーザリソースの元になるUserエンティティおよび、それと1対多の関係を持つCredentialエンティティは以下のようなプロパティをもつクラスとします。

|br|

.. sourcecode:: java
   :caption: User

   package org.debugroom.mynavi.sample.aws.microservice.backend.user.domain.model.entity;

   // omit

   @Entity
   @Table(name = "usr", schema = "public", catalog="sample")
   public class User implements Serializable{

       private long userId;
       private String firstName;
       private String familyName;
       private String loginId;
       private Boolean isLogin;
       private Boolean isAdmin;
       private Integer ver;
       private Timestamp lastUpdatedAt;
       private Set<Credential> credentialsByUserId;

       // omit

|br|

.. sourcecode:: java
   :caption: Credential

   package org.debugroom.mynavi.sample.aws.microservice.backend.user.domain.model.entity;

   // omit

   @Entity
   @IdClass(CredentialPK.class)
   public class Credential {

       private long userId;
       private String credentialType;
       private String credentialKey;
       private Timestamp validDate;
       private Integer ver;
       private User usrByUserId;

       // omit

|br|

BackendUserControllerクラスでは、以下のようにエンドポイントをもつREST API構成とします。
なお、JSONで返却されるリソースクラスは、バックエンドのマイクロサービスやフロントエンドのWebアプリケーション双方で参照可能なよう、commonプロジェクト内に作成します。

|br|

.. list-table:: バックエンドマイクロサービスのエンドポイント構成
   :widths: 1, 4, 5

   * - No
     - エンドポイント
     - 説明

   * - 1
     - /backend/user/api/v1/users
     - データベースにあるユーザリソースを全て返却するAPI

   * - 2
     - /backend/user/api/v1/users/X
     - Xで指定されたユーザIDをもつユーザリソースを返却するAPI

   * - 3
     - /backend/user/api/v1/users/user?loginId=Y
     - yで指定されたログインID情報をもつユーザリソース返却するAPI

|br|

このAPIを構成するためのController実装やアプリケーション設定ファイル(application.yml)は以下の通りです。

|br|

.. sourcecode:: java
   :caption: BackendUserController

   package org.debugroom.mynavi.sample.aws.microservice.backend.user.app.web;

   // omit

   import org.debugroom.mynavi.sample.aws.microservice.common.apinfra.exception.BusinessException;
   import org.debugroom.mynavi.sample.aws.microservice.common.model.UserResource;

   @RestController
   @RequestMapping("api/v1")
   public class BackendUserController {

       @Autowired
       SampleService sampleService;

       @GetMapping("/users")
       public List<UserResource> getUsers(){
           return UserResourceMapper.mapWithCredentials(sampleService.getUsers());
       }

       @GetMapping("/users/{id:[0-9]+}")
       public UserResource getUser(@PathVariable Long id) throws BusinessException {
           return UserResourceMapper.mapWithCredentials(sampleService.getUser(id));
       }

       @GetMapping("/users/user")
       public UserResource getUserByLoginId(
         @RequestParam("loginId") String loginId) throws BusinessException {
           return UserResourceMapper.mapWithCredentials(
             sampleService.getUserByLoginId(loginId));
       }

   }

|br|

.. sourcecode:: none
   :caption: application.yml

   server:
     servlet:
       context-path: /backend/user

|br|

application.ymlのserver.servlet.context-pathで指定している「/backend/user」はアプリケーションのURLドメイン名の配下に指定されるコンテキストパスになりますが、これを指定する理由は
複数のマイクロサービスが実行される場合に、ALBでターゲットグループを指定する際の識別子として利用するためです。また、今回は実装していませんが、
APIのバージョン変更が発生していく際は@ProfileアノテーションなどSpringのプロファイル機能を使って、バージョン情報を指定して実行するとよいです。
実行されるコンテナ内のマイクロサービスアプリケーションのControllerをプロファイルで切り替えつつ、ターゲットグループに「/backend/user/api/vX」といったかたちでバージョン事に異なるコンテナへ
ルーティングされるように構成すれば、複数のAPIバージョンをもつアプリケーションコンテナが混在する場合でも同一のECSクラスタ内に並行して実行でき、より柔軟性の高い運用が可能になります。
これらはまたALBやターゲットグループを作成するときに解説します。

|br|

なお、マイクロサービス内で発生した例外はcommonプロジェクト内に作成した、@ControllerAdviceを付与した以下のExceptionHanlderを使って一元的に設定することが可能です。
今回の実装では、想定されるビジネス例外やチェックエラーについて、必要なエラーステータスコードを設定して返却するものとします。
システム例外となるRuntimeException系のエラーはSpringが自動的に500系のInternalServerErrorでレスポンスするので設定はしていません。

|br|

.. sourcecode:: java
   :caption: CommonExceptionHandler

   package org.debugroom.mynavi.sample.aws.microservice.common.apinfra.exception;

   import java.util.List;

   import org.springframework.http.HttpStatus;
   import org.springframework.http.ResponseEntity;
   import org.springframework.validation.BindException;
   import org.springframework.validation.BindingResult;
   import org.springframework.validation.FieldError;
   import org.springframework.web.bind.MethodArgumentNotValidException;
   import org.springframework.web.bind.annotation.ControllerAdvice;
   import org.springframework.web.bind.annotation.ExceptionHandler;

   @ControllerAdvice
   public class CommonExceptionHandler {

       @ExceptionHandler(BusinessException.class)
       public ResponseEntity<ErrorResponse> handleBusinessException(
         BusinessException businessException){
           businessException.setStackTrace(new StackTraceElement[]{});
           return new ResponseEntity<>(
             BusinessExceptionResponse.builder()
                     .businessException(businessException).build(), HttpStatus.BAD_REQUEST);
       }

       @ExceptionHandler({MethodArgumentNotValidException.class, BindException.class})
       public ResponseEntity<ErrorResponse> handleValidationException(Exception exception){
           BindingResult bindingResult = null;
           if(exception instanceof MethodArgumentNotValidException){
               bindingResult = ((MethodArgumentNotValidException)exception).getBindingResult();
           }else if(exception instanceof BindException){
               bindingResult = ((BindException)exception).getBindingResult();
           }
           List<FieldError> fieldErrors = bindingResult.getFieldErrors();
           return new ResponseEntity<>(
             ValidationErrorResponse.builder().validationErrors(
                     ValidationErrorMapper.map(fieldErrors)).build(),
             HttpStatus.BAD_REQUEST);
       }
   }

|br|

アプリケーションをメインクラスから起動した後、エンドポイントにアクセスすると以下のような応答が返されます。

.. sourcecode:: bash

   curl --dump-header - http://localhost:8081/backend/user/api/v1/users/user?loginId=jiro.mynavi

   HTTP/1.1 200
   Content-Type: application/json
   Transfer-Encoding: chunked
   Date: Fri, 21 Aug 2020 16:53:47 GMT

   {"userId":"2","firstName":"jiro","familyName":"mynavi","loginId":"jiro.mynavi","credentialResources":[{"userId":2,"credentialType":"PASSWORD","credentialKey":"$2a$11$5knhINqfA8BgXY1Xkvdhvu0kOhdlAeN1H/TlJbTbuUPDdqq.H.zzi","validDate":"2018-12-31T15:00:00.000+00:00"},{"userId":2,"credentialType":"ACCESSTOKEN","credentialKey":"987654321poiuytrewq","validDate":"2015-12-31T15:00:00.000+00:00"}],"login":false,"admin":false}

|br|

指定したログインIDをもつユーザが存在しないなど、想定されるビジネス例外の場合は以下のようなレスポンスが返却されます。


|br|

.. sourcecode:: bash

   curl --dump-header - http://localhost:8081/backend/user/api/v1/users/user?loginId=jiro.mynavi2

   HTTP/1.1 400
   Content-Type: application/json
   Transfer-Encoding: chunked
   Date: Thu, 20 Aug 2020 19:31:43 GMT
   Connection: close

   {"@class":"org.debugroom.mynavi.sample.aws.microservice.common.apinfra.exception.BusinessExceptionResponse","businessException":{"cause":null,"stackTrace":[],"code":"BE0002","args":null,"message":"ログインIDは存在しません。 LoginID : jiro.mynavi2","suppresslizedMessage":"ログインIDは存在しません。 LoginID : jiro.mynavi2"}}

|br|

.. note:: 本来はHTTPステータスコードで404指定し、リソースが存在しないなどで返す方法が妥当なケースもありますが、複雑な理由でのビジネス例外やレスポンスにエラー内容を表すパラメータが必要な場合を想定して、今回はこのようにBusinessExceptionをJSONで返す形式で実装しています。
          APIの規約に応じて適切なレスポンスステータスコードを返却するように実装してください。

|br|

今回はデータベースに保存したユーザ情報を取得するバックエンドマイクロサービスを作成しました。
次回は、前回作成したフロントのWebアプリケーションからこのマイクロサービスを呼び出すようなかたちにして
認証処理がデータベースに保存されたものを使って正しく行われるように実装しなおしていきます。

|br|

著者紹介
------------------------------------------------------------------

川畑 光平(KAWABATA Kohei) - NTTデータ

.. figure:: img/overview/aws_361383_075.jpeg

金融機関システム業務アプリケーション開発・システム基盤担当、ソフトウェア開発自動化関連の研究開発を経て、デジタル技術関連の研究開発・推進に従事。

Red Hat Certified Engineer、Pivotal Certified Spring Professional、AWS Certified Solutions Architect Professional等の資格を持ち、アプリケーション基盤・クラウドなど様々な開発プロジェクト支援にも携わる。

`AWS Top Engineers & Ambassadors <https://aws.amazon.com/jp/partners/ambassadors/?cards-body.sort-by=item.additionalFields.ambassadorName&cards-body.sort-order=asc&cards-body.q=kawabata&cards-body.q_operator=AND>`_ 選出。

本連載記事の内容に対するご意見・ご質問は `Facebook <https://www.facebook.com/kohei.kawabata.5>`_ まで。
