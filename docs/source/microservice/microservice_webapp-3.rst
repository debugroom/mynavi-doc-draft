.. include:: ../module.txt

.. _section-aws-microservice-webapp-3-label:

【第6回】Webアプリケーションからマイクロサービスを呼び出す実装(1)
====================================================================

|br|

本連載では、以下に示すようなマイクロサービスアーキテクチャのアプリケーション環境を構築しています。

|br|

.. figure:: img/webapp/service-architecture.png

|br|


前回は上図に示すバックエンドサブネットに配置することを想定し、ユーザ情報をリソースとして取り扱うマイクロサービスをローカルで実行できるように実装しました。
今回は第3・4回で作成したWebアプリケーションのSpringSecurityを使った実装で、バックエンドのマイクロサービスから取得したユーザリソースを使って認証処理を行うようアプリケーションを修正していきます。
なお、本連載で実際に作成するアプリケーションでは `GitHub <https://github.com/debugroom/mynavi-sample-aws-microservice/tree/feature_2-backend-user-service>`_ 上にコミットしています。
以降に記載するソースコードでは、import文など本質的でない記述を省略している部分があるので、実行コードを作成する際は、必要に応じて適宜GitHubにあるソースコードも参照してください。


|br|

Spring Securityを使ったWebアプリケーションの修正(1)
------------------------------------------------------------------

|br|

第3・4回で実装したコンポーネントに対して、以下の通り修正・追加します。アプリケーションコンポーネントのレイヤ構成については、
連載「AWSで実践！基盤・デプロイ自動化」の連載 `第4回 <https://news.mynavi.jp/itsearch/article/devsoft/4475>`_
で記載したBackendForFrontendアプリケーションと同様のプロジェクト・パッケージ構成とし、 `TERAOLUNAのガイドライン 「アプリケーションのレイヤ化」 <http://terasolunaorg.github.io/guideline/5.6.0.RELEASE/ja/Overview/ApplicationLayering.html>`_ の考え方に準拠するものとします。

|br|

.. list-table:: アプリケーション
   :widths: 3, 1, 6

   * - コンポーネント
     - 追加／修正
     - 説明

   * - WebApp
     - 変更なし
     - SpringBootアプリケーションを実行する起動クラス

   * - MvcConfig
     - 修正
     - ユーザの権限に応じてメニューを切り替えるインターセプタ設定定義を追加する修正を行う

   * - SecurityConfig
     - 修正
     - パスワードエンコーダをBcryptPasswordEncoderを使用するように設定を修正する。
       BCryptを使った方法については `TERASOLUNAガイドライン BCryptPasswordEncoder <https://terasolunaorg.github.io/guideline/5.4.2.RELEASE/ja/Security/Authentication.html#bcryptpasswordencoder>`_ を参照すること※

   * - DevConfig
     - 追加
     - ローカルで動作する「dev」プロファイルをもつアプリケーションがバックエンドマイクロサービスにアクセスするためのドメインを設定するクラス

   * - DomainConfig
     - 追加
     - ServiceやRepository等ドメイン層の設定クラス

   * - SampleController
     - 修正
     - CustomUserDetailsを引数のパラメータとして受け取り、画面へモデルとして渡すように修正を行う。

   * - CustomUserDetails
     - 修正
     - バックエンドマイクロサービスから取得したUserResourceの情報を使ってIDとパスワードを返却するように修正を行う。

   * - CustomUserDetailsServie
     - 修正
     - バックエンドマイクロサービスからUserResourceを取得し、CustomuUserDetailsにセットして生成、返却するよう修正を行う。

   * - LoginSuccessHandler
     - 変更なし
     - ログインが成功したのちに実行されるハンドラクラス

   * - SessionExpiredDetectingLoginUrlAuthenticationEntryPoint
     - 変更なし
     - セッションが無効になったことを検出し、ログイン画面へ遷移するためのハンドラクラス

   * - Menu
     - 追加
     - メニューを表すモデルクラス。ポータル、ログアウト、ユーザ管理をEnum型として表す。

   * - SetMenuInterceptor
     - 追加
     - CustomUserDetailsが保持するGrantedAuthoriyに応じて画面に表示させるメニューリストを生成するインターセプタ

   * - PortalInformation
     - 追加
     - ポータル画面に表示するデータを集約するモデルクラス

   * - ServiceProperties
     - 追加
     - バックエンドマイクロサービスのドメインを保持するプロパティクラス。application-dev.ymlから取得するよう設定する。

   * - UserResourceRepository(Impl)
     - 追加
     - バックエンドマイクロサービスにアクセスするためのRepositoryクラス。例外処理など集約するためRepositoryクラスを作成し、WebClientを使ったアクセスクラスを実装。

   * - OrchestrationService(Impl)
     - 追加
     - 複数のマイクロサービスへのアクセスを実行制御するためのServiceクラス。Webアプリケーションのビジネスロジックのトランザクション境界として実装され、リトライ制御や補償トランザクションなどの役割をもつ。

|br|

.. note:: ※ `最新のガイドライン <https://terasolunaorg.github.io/guideline/5.6.0.RELEASE/ja/Security/Authentication.html#springsecurityauthenticationpasswordhashing>`_ では、PBKDF2アルゴリズムを使ったパスワード認証方法が推奨されています。

|br|

実装のポイントになるものについて説明します。まず、ドメイン層のコンポーネントとして追加するOrchestrationServiceクラスです。このクラスは
バックエンドのマイクロサービスの呼び出しが複数になるような場合などで実行フローを制御する役割をもち、Webアプリケーションのビジネスロジックのトランザクション境界として実装します。
下の例は前回実装したバックエンドのユーザサービスのみを呼び出していますが、必要に応じ別のマイクロサービスの呼び出しや、
リトライ制御、エラーが発生してロールバックしたい場合の補償トランザクションなどの処理もこのクラスで行うとよいでしょう。

|br|

.. sourcecode:: java
   :caption: OrchestrationServiceImpl

   package org.debugroom.mynavi.sample.aws.microservice.frontend.webapp.domain.service;

   import org.debugroom.mynavi.sample.aws.microservice.common.apinfra.exception.BusinessException;
   import org.debugroom.mynavi.sample.aws.microservice.common.model.UserResource;
   import org.debugroom.mynavi.sample.aws.microservice.frontend.webapp.domain.repository.UserResourceRepository;
   import org.springframework.beans.factory.annotation.Autowired;
   import org.springframework.stereotype.Service;

   @Service
   public class OrchestrationServiceImpl implements OrchestrationService{

       @Autowired
       UserResourceRepository userResourceRepository;

       @Override
       public UserResource getUserResource(String loginId) throws BusinessException {
           return userResourceRepository.findOneByLoginId(loginId);
       }
   }

|br|

マイクロサービスの呼び出し処理は、Repositoryクラスを作成し、その中で実装します。以下の例では、
呼び出し処理自体をorg.springframework.web.reactive.function.client.WebClientを使用して行っています。
サービス呼び出し時に発生する例外処理を一元的にこのクラスの責務として担うことで、上述のOrchestrationServiceクラスの記述をすっきりさせることができます。
なお、今回はコメントアウトしていますが、マイクロサービスのAPIバージョンが変更された場合に備えて、Profileアノテーションなどで切り替えるようにするとよいでしょう。

|br|

.. sourcecode:: java
   :caption: UserResourceRepositoryImpl

   package org.debugroom.mynavi.sample.aws.microservice.frontend.webapp.domain.repository;

   // omit

   import com.fasterxml.jackson.databind.ObjectMapper;
   import org.springframework.beans.factory.annotation.Autowired;
   import org.springframework.context.MessageSource;
   import org.springframework.context.annotation.Profile;
   import org.springframework.stereotype.Component;
   import org.springframework.web.reactive.function.client.WebClient;
   import org.springframework.web.reactive.function.client.WebClientResponseException;

   import org.debugroom.mynavi.sample.aws.microservice.common.apinfra.exception.BusinessException;
   import org.debugroom.mynavi.sample.aws.microservice.common.apinfra.exception.BusinessExceptionResponse;
   import org.debugroom.mynavi.sample.aws.microservice.common.apinfra.exception.ErrorResponse;
   import org.debugroom.mynavi.sample.aws.microservice.common.apinfra.exception.SystemException;
   import org.debugroom.mynavi.sample.aws.microservice.common.model.UserResource;

   //@Profile("v1")
   @Component
   public class UserResourceRepositoryImpl implements UserResourceRepository{

       private static final String SERVICE_NAME = "/backend/user";
       private static final String API_VERSION = "/api/v1";

       @Autowired
       WebClient webClient;

       @Autowired
       ObjectMapper objectMapper;

       @Autowired
       MessageSource messageSource;

       @Override
       public UserResource findOne(String userId) {
           String endpoint = SERVICE_NAME + API_VERSION + "/users/{userId}";
           return webClient.get()
             .uri(uriBuilder -> uriBuilder.path(endpoint).build(userId))
             .retrieve()
             .bodyToMono(UserResource.class)
             .block();
       }

       @Override
       public UserResource findOneByLoginId(String loginId) throws BusinessException {
           String endpoint = SERVICE_NAME + API_VERSION + "/users/user";
           try{
               return webClient.get()
                 .uri(uriBuilder -> uriBuilder
                         .path(endpoint).queryParam("loginId", loginId).build())
                 .retrieve()
                 .bodyToMono(UserResource.class)
                 .block();
           }catch (WebClientResponseException e){
               try {
                   ErrorResponse errorResponse = objectMapper.readValue(
                     e.getResponseBodyAsString(), ErrorResponse.class);
                   if(errorResponse instanceof BusinessExceptionResponse){
                       throw ((BusinessExceptionResponse)errorResponse).getBusinessException();
                   }else {
                       String errorCode = "SE0002";
                       throw new SystemException(errorCode, messageSource.getMessage(
                         errorCode, new String[]{endpoint}, Locale.getDefault()), e);
                   }
               }catch (IOException e1){
                   String errorCode = "SE0002";
                   throw new SystemException(errorCode, messageSource.getMessage(
                     errorCode, new String[]{endpoint}, Locale.getDefault()), e);
               }
           }
       }

       @Override
       public List<UserResource> findAll() {
           String endpoint = SERVICE_NAME + API_VERSION + "/users";
           return Arrays.asList(
             webClient.get()
                     .uri(uriBuilder -> uriBuilder.path(endpoint).build())
                     .retrieve().bodyToMono(UserResource[].class).block());
       }

   }

|br|

WebClientで呼び出すバックエンドマイクロサービスのURLドメインは以下の通り設定クラスで一律に設定することが可能です。
devとなる開発環境固有のapplication-dev.ymlからConfigurationPropertiesなどで取得できるように実装しておきましょう。

|br|

.. sourcecode:: java
   :caption: DevConfig

   package org.debugroom.mynavi.sample.aws.microservice.frontend.webapp.config;

   // omit

   @Profile("dev")
   @Configuration
   public class DevConfig {

       @Autowired
       ServiceProperties serviceProperties;

       @Bean
       public WebClient userWebClient(){
           return WebClient.builder()
             .baseUrl(serviceProperties.getDns())
             .build();
       }

   }

|br|

.. sourcecode:: java
   :caption: ServiceProperties

   package org.debugroom.mynavi.sample.aws.microservice.frontend.webapp.domain;

   import org.springframework.boot.context.properties.ConfigurationProperties;
   // omit

   @ConfigurationProperties(prefix = "service")
   public class ServiceProperties {

       private String dns;

   }

|br|

.. sourcecode:: none
   :caption: application-dev.yml

   service:
     dns: http://localhost:8081

|br|

.. note:: これまでの連載でREST APIの呼び出しはRestTemplateを使って実装してきましたが、Spring5.0からメンテナンスモードに移行したため、
          より通信のコストパフォーマンスが良いWebClientを使った実装に切り替えています。なお、WebClientを使用する際は以下の通り、
          spring-boot-starter-webfluxをpom.xmlの依存性に追加する必要があります。

          .. sourcecode:: xml
             :caption: frontend-webapp/pom.yml

             <dependency>
                 <groupId>org.springframework.boot</groupId>
                 <artifactId>spring-boot-starter-webflux</artifactId>
             </dependency>


|br|

今回はバックエンドのユーザサービスを呼び出すコンポーネントをUserResourceRepositoryとして実装し、OrchestrationService経由で呼び出すように実装しました。
次回は、CustomUserDetailsServiceからこれらを呼び出すようなかたちにして、認証処理がバックエンドから取得したUserResourceを使って行われるように実装しなおしてみます。

|br|

著者紹介
------------------------------------------------------------------

川畑 光平(KAWABATA Kohei) - NTTデータ

.. figure:: img/overview/aws_361383_075.jpeg

金融機関システム業務アプリケーション開発・システム基盤担当、ソフトウェア開発自動化関連の研究開発を経て、デジタル技術関連の研究開発・推進に従事。

Red Hat Certified Engineer、Pivotal Certified Spring Professional、AWS Certified Solutions Architect Professional等の資格を持ち、アプリケーション基盤・クラウドなど様々な開発プロジェクト支援にも携わる。

`AWS Top Engineers & Ambassadors <https://aws.amazon.com/jp/partners/ambassadors/?cards-body.sort-by=item.additionalFields.ambassadorName&cards-body.sort-order=asc&cards-body.q=kawabata&cards-body.q_operator=AND>`_ 選出。

本連載記事の内容に対するご意見・ご質問は `Facebook <https://www.facebook.com/kohei.kawabata.5>`_ まで。
