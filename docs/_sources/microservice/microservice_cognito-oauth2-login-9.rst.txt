.. include:: ../module.txt

.. _section-aws-microservice-cognito-oauth2-login-9-label:

【第20回】Amazon Cognito + Spring Sercurityを使ったOAuth2 Loginの実装(9)
===============================================================================================

|br|

前回から、以下のイメージのようにOAuth2 Loginをベースとしたアーキテクチャを想定した環境構築を進めています。

|br|

.. figure:: img/cognito-oauth2-login/oauth2-login-flow.png

|br|

前回は、FrontendのアプリケーションでCognitoを認証プロバイダとするOAuth2 LoginをSpring Securityを使って実装しました(上図で1〜8の処理に相当)。
今回からは、ログイン時に取得したアクセストークンを使って、バックエンドサービスへのアクセス制御を実現する実装を紹介します。
上記の図で、(0)と(9)の実装方法になります。

なお、実際のソースコードは `GitHub <https://github.com/debugroom/mynavi-sample-aws-microservice/tree/feature_5_backend-cognito-authorization>`_ 上にコミットしています。以降のソースコードでは本質的でない記述を一部省略しているので、実行コードを作成する場合は、必要に応じて適宜GitHub上のソースコードも参照してください。

|br|

バックエンドサービスの呼び出しにアクセストークンを設定する実装
----------------------------------------------------------------------------------

|br|

前回解説した通り、Spring Securityでは、OAuth2 Login が成功したのち、Controllerの引数として、
org.springframework.security.oauth2.core.oidc.user.OidcUserや
org.springframework.security.oauth2.client.authentication.OAuth2AuthenticationTokenを受け取ることができます。
このオブジェクトの中には、IDトークン、アクセストークン、リフレッシュトークンの他に、OIDCで定められた
ユーザクレイム(ユーザ情報)が含まれており、アプリケーション内でこれらを自由に利用することができます。

|br|

.. sourcecode:: java

   package org.debugroom.mynavi.sample.aws.microservice.frontend.webapp.app.web;

   // omit

   import org.springframework.beans.factory.annotation.Autowired;
   import org.springframework.security.core.annotation.AuthenticationPrincipal;
   import org.springframework.security.oauth2.client.OAuth2AuthorizedClient;
   import org.springframework.security.oauth2.client.OAuth2AuthorizedClientService;
   import org.springframework.security.oauth2.client.authentication.OAuth2AuthenticationToken;
   import org.springframework.security.oauth2.core.oidc.user.OidcUser;
   import org.springframework.stereotype.Controller;
   import org.springframework.ui.Model;
   import org.springframework.web.bind.annotation.GetMapping;

   import org.debugroom.mynavi.sample.aws.microservice.frontend.webapp.app.web.security.CustomUserDetails;

   @Controller
   public class SampleController {

       @Autowired
       OAuth2AuthorizedClientService oAuth2AuthorizedClientService;

       // omit

       @GetMapping(value = "/oauth2LoginSuccess")
       public String oauth2SuccessPortal(@AuthenticationPrincipal OidcUser oidcUser
                         , OAuth2AuthenticationToken oAuth2AuthenticationToken, Model model){
           OAuth2AuthorizedClient oAuth2AuthorizedClient =
             oAuth2AuthorizedClientService.loadAuthorizedClient(
                     oAuth2AuthenticationToken.getAuthorizedClientRegistrationId(),
                     oAuth2AuthenticationToken.getName());
           model.addAttribute("oidcUser", oidcUser);
           model.addAttribute( oAuth2AuthorizedClientService
             .loadAuthorizedClient(
                     oAuth2AuthenticationToken.getAuthorizedClientRegistrationId(),
                     oAuth2AuthenticationToken.getName()));
           model.addAttribute("accessToken", oAuth2AuthorizedClientService
             .loadAuthorizedClient(
                     oAuth2AuthenticationToken.getAuthorizedClientRegistrationId(),
                     oAuth2AuthenticationToken.getName()).getAccessToken());
           return "oauth2Portal";
      }
   }

|br|

Spring Securityには、バックエンドサービスなどリクエストを呼び出す際に、取得したこれらのトークンを
リクエストヘッダへ透過的に設定する仕組みが用意されています。`第6回 <https://news.mynavi.jp/techplus/article/techp5173/>`_ でも解説した通り、
このアプリケーションではバックエンドサービスの呼び出しにWebClientを使用していますが、
このWebClientを使ったサービス呼び出し時にリクエストヘッダへアクセストークンを設定するには、
以下の通り、設定クラスでWebClientをBean定義する際にorg.springframework.security.oauth2.client.web.reactive.function.client.ServletOAuth2AuthorizedClientExchangeFilterFunctionを設定します。

|br|

.. sourcecode:: java

   package org.debugroom.mynavi.sample.aws.microservice.frontend.webapp.config;

   // omit

   import org.springframework.beans.factory.annotation.Autowired;
   import org.springframework.context.annotation.Bean;
   import org.springframework.context.annotation.Configuration;
   import org.springframework.context.annotation.Profile;
   import org.springframework.security.oauth2.client.OAuth2AuthorizedClientManager;
   import org.springframework.security.oauth2.client.web.reactive.function.client.ServletOAuth2AuthorizedClientExchangeFilterFunction;
   import org.springframework.web.reactive.function.client.ClientRequest;
   import org.springframework.web.reactive.function.client.ExchangeFilterFunction;
   import org.springframework.web.reactive.function.client.WebClient;

   import org.debugroom.mynavi.sample.aws.microservice.frontend.webapp.domain.ServiceProperties;

   @Profile("dev")
   @Configuration
   public class DevConfig {

       @Autowired
       ServiceProperties serviceProperties;

       @Bean
       public WebClient userWebClient(OAuth2AuthorizedClientManager oAuth2AuthorizedClientManager){
           ServletOAuth2AuthorizedClientExchangeFilterFunction function =
             new ServletOAuth2AuthorizedClientExchangeFilterFunction(oAuth2AuthorizedClientManager);
           function.setDefaultClientRegistrationId("cognito");
           return WebClient.builder()
                     .baseUrl(serviceProperties.getApplicationLoadBalancer().getDns())
                     .filter(function)
                     .build();
       }

|br|

WebClientには、リクエスト呼び出し時に共通的な処理を実行するための仕組みとしてExchageFilterFunctionがあります。
Spring Securityで提供されているServletOAuth2AuthorizedClientExchangeFilterFunctionを設定することにより、
HTTPリクエストのAuthrozationヘッダに「 Bearer : XXXXXX」の形式でアクセストークンをセットするようになります。
なお、設定時にはClientRegistrationIdを指定しておきます。これは、複数の認証プロバイダをサポートする際に、
どのプロバイダのトークンをセットするのか識別するためです。

WebClientを使って実際にバックエンドサービスを呼び出す際は、特にこれまでと変わらず実装するだけでかまいません。

|br|

.. sourcecode:: java

   package org.debugroom.mynavi.sample.aws.microservice.frontend.webapp.domain.repository;

   // omit
   import org.springframework.web.reactive.function.client.WebClient;
   import org.debugroom.mynavi.sample.aws.microservice.common.model.UserResource;

   @Component
   public class UserResourceRepositoryImpl implements UserResourceRepository{

       private static final String SERVICE_NAME = "/backend/user";
       private static final String API_VERSION = "/api/v1";

       @Autowired
       WebClient webClient;

       // omit

       @Override
       public UserResource findOne(String userId) {
           String endpoint = SERVICE_NAME + API_VERSION + "/users/{userId}";
           return webClient.get()
             .uri(uriBuilder -> uriBuilder.path(endpoint).build(userId))
             .retrieve()
             .bodyToMono(UserResource.class)
             .block();
       }

|br|

今回はバックエンドサービスを呼び出す際に、アクセストークンをリクエストヘッダに設定する方法を解説しました。
次回は、バックエンドサービスでアクセストークンによりサービスを保護する実装方法について解説します。

著者紹介
------------------------------------------------------------------

川畑 光平(KAWABATA Kohei) - NTTデータ エグゼクティブ ITスペシャリスト ソフトウェアアーキテクト・デジタルテクノロジーストラテジスト(クラウド)

.. figure:: img/overview/aws_361383_075.jpeg

金融機関システム業務アプリケーション開発・システム基盤担当、ソフトウェア開発自動化関連の研究開発を経て、デジタル技術関連の研究開発・推進に従事。

Red Hat Certified Engineer、Pivotal Certified Spring Professional、AWS Certified Solutions Architect Professional等の資格を持ち、アプリケーション基盤・クラウドなど様々な開発プロジェクト支援にも携わる。

`AWS Top Engineers & Ambassadors <https://aws.amazon.com/jp/partners/ambassadors/?cards-body.sort-by=item.additionalFields.ambassadorName&cards-body.sort-order=asc&cards-body.q=kawabata&cards-body.q_operator=AND>`_ 選出。

本連載記事の内容に対するご意見・ご質問は `Facebook <https://www.facebook.com/kohei.kawabata.5>`_ まで。
