.. include:: ../module.txt

.. _section-aws-microservice-webapp-1-label:

【第3回】Webアプリケーションの実装(1)
====================================================================

マイクロサービスアプリケーション環境の詳細と利用するサービス・ライブラリ
----------------------------------------------------------------------------------

|br|

前回はマイクロサービスの具体的なアーキテクチャの全体像と、そのアーキテクチャを実現する上で、難しいハードルのひとつである認証／認可処理の全体像や考え方を解説しました。
今回からアプリケーションを実装していくにあたり、詳細なアーキテクチャや使用するマネージドサービス・ライブラリなどを整理し、実際に実装を進めていきます。
本連載では、段階的にアプリケーションやそれを実行するのに必要な環境を構築していきますが、まず最初に最終的なアプリケーション構成や使用するサービスのイメージを示します。
下記の図は代表的なクライアントになるWebアプリケーションとそれが呼び出すマイクロサービスをピックアップして詳細化したものです。

|br|

.. figure:: img/webapp/service-architecture.png

|br|

何故こういう構成にするのかを簡単に説明します。前回も解説した通り、Webアプリケーションや管理用のアプリケーション、モバイルアプリケーションに加えてサードパーティの外部サービスなど
様々なクライアントがマイクロサービスを利用したいとします。マイクロサービスは時々のリクエスト量に応じてスケールアウト、スケールインして負荷を分散しながらも、
正しい権限をもったクライアントからのリクエストであるか、その正当性を確認しなければなりません。基本的にクライアントはWebアプリケーションの他にもモバイルやサードパーティ製サービスからなる不特定多数であり、
VPCを分けて、全て信頼されたネットワークからしかアクセスできないように制御したインターナルなALBを経由して、オートスケーリング設定を施したECSクラスタに配置されたマイクロサービスを呼び出させるように構成します。

このような構成により不正アクセスやDDos攻撃などのセキュリティの脅威を可能な限り排除します。モノリシックなアプリケーション構成と比べて多段化するので、
AWS X-Rayを用いてアプリケーションやサービス間の呼び出しを可視化します。リクエスト頻度が高いサービスがアクセスするデータベースは、場合によってRDSの他にも
DynamoDBのようなスケーラビリティ性を有するものを使用した方がよいでしょう。アプリケーション環境の構築やデプロイは迅速性・正確性が要求されるため、CloudFormationを使って構築します。
なお、フロントエンドサブネットでElastiCacheを配置しているのは、フロントに配置されたアプリケーションをスケールさせた場合にセッション共有するためです。
その他、不要なバックエンドのマイクロサービスの呼び出しをさけるためにキャッシュとしても利用します。

今回からはフロントエンドサブネットにあるマイクロサービスを利用するクライアントとなるWebアプリケーションを実装していきます。
このアプリケーションは連載「AWSで実践! 基盤構築・デプロイ自動化」の `第4回 <https://news.mynavi.jp/itsearch/article/devsoft/4475>`_ で示したアプリケーションパッケージ・コンポーネントとほぼ同等に構成していきます。
まず作成するのは、バックエンドの呼び出しを行わない簡単なWebアプリケーションのサンプルですが、通常Webアプリケーション自体にも認証認可処理が必要であり、これを追加します。SpringBootアプリケーションで認証認可を行う場合、一般にSpringSecurityを使用します。
次回以降の解説で、Webアプリケーションから呼び出すマイクロサービスにもAWS CognitoおよびSpring Securityを使ったOAuth2のアクセストークンによる認可制御を実装するので、まずSpring Securityの基本的な概要や使い方を押さえておきましょう。

|br|

Spring Securityの概要
------------------------------------------------------------------

|br|

Spring Securityに関するドキュメントは `Springの公式 <https://spring.io/projects/spring-security#overview>`_ 以外にも `TERASOLUNAのガイドライン セキュリティ対策 <http://terasolunaorg.github.io/guideline/5.6.0.RELEASE/ja/Security/index.html>`_ が詳しいですが、
端的に説明すれば、SpringFrameworkを用いてWebアプリケーションを作成する場合に認証認可処理を簡単に実装できるフレームワークです。JavaWebアプリケーションの標準的な機能であるサーブレットフィルタを使って、
リクエストが処理される前に正当性の検証や不正リクエストのブロックなど一般的なセキュリティ脆弱性をついた攻撃に対して防御となる機能を提供しています。
他にもパスワードのハッシュ化や暗号化、本連載のメイントピックの一つになるOAuth2を使ったトークン認証など様々なセキュリティ対策の処理を実装しています。
セキュリティ脆弱性をついた攻撃に対する防御方法は一般に難解であり、このような問題に個々で対処するには難易度も高いのでこのようなライブラリを使用する方がベターです。
セキュリティ対策の難解性ゆえに、SpringSecurityを扱うこと自体も難しく捉えられがちですが、非常に拡張性も高く、少量のコーディングで様々なセキュリティ対策処理を実装することが可能です。

|br|

.. note:: 前回はOIDCやOAuth2.0を用いた認証認可について説明していますが、今回はまず最初にWebアプリケーションで実装される認証・認可処理を取り上げます。
          OIDCやOAuth2.0では、前回も説明した通り、あるアプリケーションの一部の処理をWebサービスとして公開したい場合に、不特定多数のクライアントやサードパーティサービスが
          適切な権限をもってサービスを安全に利用できるようにするための認証・認可の仕組みです。SpringSecurityではどちらも用途でもカバーしています。

|br|

Spring Securityを使ったWebアプリケーション(1)
------------------------------------------------------------------

|br|
では早速、SpringSecurityを使ってIDとパスワードを使ってログイン・認証を行う簡単なアプリケーションを実装してみましょう。
本連載で実際に作成するアプリケーションでは `GitHub <https://github.com/debugroom/mynavi-sample-aws-microservice/tree/feature_1-frontend-webapp>`_ 上にコミットしています。
以降に記載するソースコードでは、import文など本質的でない記述を省略している部分があるので、実行コードを作成する際は、必要に応じて適宜GitHubにあるソースコードも参照してください。

なお、動作環境は以下のバージョンで実施しています。

|br|

.. list-table::
   :widths: 5, 5

   * - 動作対象
     - バージョン

   * - Java
     - 11

   * - Spring Boot
     - 2.3.3.RELEASE

   * - Spring Security
     - 5.3.4.RELEASE

|br|

まず、使用するライブラリを以下の通り、pom.xmlに定義します。SpringBootを使ったWebアプリケーションを作成するためのspring-boot-starter-web、
SpringSecurityを使用するためのspring-boot-starter-security、およびテンプレートエンジンであるThymeleafを使用するためのspring-boot-starter-thymeleafを依存定義します。

|br|

.. sourcecode:: xml

   <dependency>
     <groupId>org.springframework.boot</groupId>
     <artifactId>spring-boot-starter-security</artifactId>
   </dependency>
   <dependency>
     <groupId>org.springframework.boot</groupId>
     <artifactId>spring-boot-starter-thymeleaf</artifactId>
   </dependency>
   <dependency>
     <groupId>org.springframework.boot</groupId>
     <artifactId>spring-boot-starter-web</artifactId>
   </dependency>

|br|

それでは、アプリケーションの実装に進みます。今回作成するアプリケーションは以下の構成です。

.. list-table:: アプリケーション
   :widths: 3, 6, 1

   * - コンポーネント
     - 説明
     - 必須

   * - WebApp
     - SpringBootアプリケーションを実行する起動クラス
     - ◯

   * - MvcConfig
     - SpringMVCの設定を行うクラス
     - ◯

   * - SecurityConfig
     - SpringSecurity設定クラス
     - ◯

   * - SampleController
     - ログイン画面やログイン後にポータル画面へ遷移するよう定義したController
     - ◯

   * - CustomUserDetails
     - SpringSecurityでユーザ情報を表すモデルオブジェクトを継承したカスタムクラス
     - ◯

   * - CustomUserDetailsServie
     - CustomUserDetailsを取得するためのカスタムクラス
     - ◯

   * - LoginSuccessHandler
     - ログインが成功したのちに実行されるハンドラクラス
     -

   * - SessionExpiredDetectingLoginUrlAuthenticationEntryPoint
     - セッションが無効になったことを検出し、ログイン画面へ遷移するためのハンドラクラス
     -

|br|

SpringSecurityを使った認証をアプリケーションに組み込むには、設定クラスを実装して、アプリケーション起動クラスの読み込み設定対象に加える必要があります。
今回は以下のような設定クラスを実装し、起動クラスと同一パッケージに配置しておきます。

|br|

.. sourcecode:: java

   package org.debugroom.mynavi.sample.aws.microservice.frontend.webapp.config;

   import org.springframework.beans.factory.annotation.Autowired;
   import org.springframework.context.annotation.Bean;
   import org.springframework.security.config.annotation.authentication.builders.AuthenticationManagerBuilder;
   import org.springframework.security.config.annotation.web.builders.HttpSecurity;
   import org.springframework.security.config.annotation.web.builders.WebSecurity;
   import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
   import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;
   import org.springframework.security.core.userdetails.UserDetailsService;
   import org.springframework.security.crypto.factory.PasswordEncoderFactories;
   import org.springframework.security.crypto.password.PasswordEncoder;
   import org.springframework.security.web.AuthenticationEntryPoint;

   import org.debugroom.mynavi.sample.aws.microservice.frontend.webapp.app.web.security.CustomUserDetailsService;
   import org.debugroom.mynavi.sample.aws.microservice.frontend.webapp.app.web.security.LoginSuccessHandler;
   import org.debugroom.mynavi.sample.aws.microservice.frontend.webapp.app.web.security.SessionExpiredDetectingLoginUrlAuthenticationEntryPoint;

   @EnableWebSecurity                                                           //(A)
   public class SecurityConfig extends WebSecurityConfigurerAdapter {           //(B)

       // omit

       @Override
       public void configure(WebSecurity web) throws Exception {
           web.ignoring().antMatchers("/static/**");                            //(C)
       }

       @Override
       protected void configure(HttpSecurity http) throws Exception {
           http.authorizeRequests()
                .antMatchers("/favicon.ico").permitAll()                        //(D)
                .antMatchers("/webjars/**").permitAll()
                .antMatchers("/static/**").permitAll()
                .antMatchers("/timeout").permitAll()
                .anyRequest().authenticated()                                   //(E)
                .and()
                .csrf().disable()                                               //(F)
                .formLogin()
                .loginProcessingUrl("/authenticate")                            //(G)
                .loginPage("/login")                                            //(H)
                .successHandler(loginSuccessHandler())                          //(I)
                .failureUrl("/login")                                           //(J)
                .usernameParameter("username")                                  //(K)
                .passwordParameter("password")                                  //(L)
                .permitAll()                                                    //(M)
                .and()
                .exceptionHandling()
                .authenticationEntryPoint(authenticationEntryPoint())           //(O)
                .and()
                .logout()                                                       //(P)
                .logoutSuccessUrl("/login")
                .permitAll();
       }

       @Bean
       public LoginSuccessHandler loginSuccessHandler(){                        //(Q)
           return new LoginSuccessHandler();
       }

       @Bean
       AuthenticationEntryPoint authenticationEntryPoint() {                    //(R)
           return new SessionExpiredDetectingLoginUrlAuthenticationEntryPoint("/login");
       }

       @Bean
       public PasswordEncoder passwordEncoder(){                                //(S)
           return PasswordEncoderFactories.createDelegatingPasswordEncoder();
       }

       @Override
       protected UserDetailsService userDetailsService() {                      //(T)
           return new CustomUserDetailsService();
       }

       @Override
       protected void configure(AuthenticationManagerBuilder auth) throws Exception {
            auth
                .userDetailsService(userDetailsService())
                .passwordEncoder(passwordEncoder);                              //(U)
       }
   }

|br|

SecurityConfigクラスコードの説明は以下の通りです。

|br|

.. list-table:: SecurityConfigクラスコードの説明
   :widths: 1, 19

   * - 項番
     - 説明

   * - (A)
     - @EnalbleWebSecurityアノテーションを設定クラスへ付与します。このアノテーションにより、SpringSecurityの設定クラスとして認識されます。

   * - (B)
     - SpringSecurityの設定クラスとなるクラスにはorg.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapterを継承します。
       継承したクラスの設定用のメソッドをオーバーライドすることでSpringSecurityの基本設定やカスタマイズを行います。

   * - (C)
     - 静的リソースであるCSSやJavaScriptなどsrc/main/resources/static配下に置いていますが、これらのリソースに対するリクエストは対象外とするよう設定します。

   * - (D)
     - セキュリティ設定はビルダークラスであるHttpSecurityに対して、SpringSecurityのチェック対象外とするパスをpermitAll()で設定し、これらのリソースに対するリクエストは対象外とするよう設定します。

   * - (E)
     - (D)以外のリクエストを全て認証が必要とするよう設定します。

   * - (F)
     - 簡単のためにCSRFトークンの送信が必須になる設定をオフにしておきます。次回以降、改めて有効化させます。

   * - (G)
     - ログイン処理を行うリクエストURIのパスを設定します。

   * - (H)
     - ログインフォームを表示するページを設定します。

   * - (I)
     - ログインが成功したときに実行するハンドラクラスを設定します。次回解説します。

   * - (J)
     - ログインが失敗したときに遷移するページを設定します。ここでは(H)と同じくログインフォームのページに遷移させます。ユーザ側からは同じページなので遷移しているようには見えません。ただし、エラ〜メッセージが表示されるように設定します。

   * - (K)
     - ログイン処理でIDに相当するリクエストパラメータを規定します。ここではデフォルトと同じく。「username」を設定します。このパラメータは(H)のログインのフォームのパラメータと一致させておく必要があります。

   * - (L)
     - ログイン処理でパスワードに相当するリクエストパラメータを規定します。ここではデフォルトと同じく。「password」を設定します。このパラメータは(H)のログインフォームのパラメータと一致させておく必要があります。

   * - (M)
     - ログイン処理は認証が必要とする設定から除外するように設定します。

   * - (O)
     - 認証処理で例外が発生した場合のハンドリングする設定をauthenticationEntryPointへ行います。次回以降解説します。

   * - (P)
     - ログアウト時の挙動を設定します。ログアウトが成功したら遷移する画面を(H)と同じく、ログイン画面に設定します。

   * - (Q)
     - ログインが成功したときの処理をカスタムクラスLoginSuccessHanlerとして実装し、Bean定義します。このカスタムクラスは次回解説します。

   * - (R)
     - 認証処理で例外が発生した場合の処理をカスタムクラスSessionExpiredDetectingLoginUrlAuthenticationEntryPointとして実装し、Bean定義します。このカスタムクラスは次回解説します。

   * - (S)
     - パスワードのエンコーダクラスをBean定義します。

   * - (T)
     - 認証情報となるUserDetailsを取得するためのカスタムUserDetailsServiceクラスをBean定義します。このカスタムクラスは次回解説します。

   * - (U)
     - 認証の設定を行うAuthenticationManagerBuilderクラスにUserDetailsServiceやPasswordEncoderを設定します。

|br|

今回は、マイクロサービスおよびクライアントとなるWebアプリケーションのアーキテクチャの詳細を説明し、SpringSecurityを使ったWebアプリケーションの設定クラスを実装しました。
次回は引き続き、SpringSecutiyを使ったWebアプリケーションログインページの実装やカスタム設定について解説を行い、実際にアプリケーションを起動してみます。

|br|

著者紹介
------------------------------------------------------------------

川畑 光平(KAWABATA Kohei) - NTTデータ

.. figure:: img/overview/aws_361383_075.jpeg

金融機関システム業務アプリケーション開発・システム基盤担当、ソフトウェア開発自動化関連の研究開発を経て、デジタル技術関連の研究開発・推進に従事。

Red Hat Certified Engineer、Pivotal Certified Spring Professional、AWS Certified Solutions Architect Professional等の資格を持ち、アプリケーション基盤・クラウドなど様々な開発プロジェクト支援にも携わる。

`AWS Top Engineers & Ambassadors <https://aws.amazon.com/jp/partners/ambassadors/?cards-body.sort-by=item.additionalFields.ambassadorName&cards-body.sort-order=asc&cards-body.q=kawabata&cards-body.q_operator=AND>`_ 選出。

本連載記事の内容に対するご意見・ご質問は `Facebook <https://www.facebook.com/kohei.kawabata.5>`_ まで。
