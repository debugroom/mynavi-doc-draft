.. include:: ../module.txt

.. _section-aws-microservice-webapp-2-label:

【第4回】Webアプリケーションの実装(2)
====================================================================

|br|

本連載では、以下に示すようなマイクロサービスアーキテクチャのアプリケーション環境を構築しています。

|br|

.. figure:: img/webapp/service-architecture.png

|br|


前回は上図に示すマイクロサービスおよびクライアントとなるWebアプリケーションのアーキテクチャの詳細を説明し、SpringSecurityを使ったWebアプリケーションの設定クラスを実装しました。
今回は引き続き、SpringSecutiyを使った様々なカスタム設定やWebアプリケーションログインページの実装の解説を行い、実際にアプリケーションを起動してみます。

|br|

Spring Securityを使ったWebアプリケーション(2)
------------------------------------------------------------------

|br|

前回実装したSpringSecurity設定クラスにBean定義したカスタムクラスを解説します。まずCustomUserDetailsServiceです。
SpringSecurityによるログイン処理では、リクエストパラメータとして送信されてきたIDとパスワードと一致しているか検証するロジックが既に実装されて組み込まれています。
リクエストパラメータのID・パスワードと一致するかを検証する対象となるモデルのオブジェクトは、IDとパスワードがプロパティとして定義されている
org.springframework.security.core.userdetails.UserDetailsというインターフェースを実装していればよく、このモデルオブジェクトを取得できるサービスクラスをユーザが任意にカスタマイズできます。
このサービスクラスはorg.springframework.security.core.userdetails.UserDetailsServiceインターフェースを実装しておく必要があります。
今回簡単なサンプルとして以下の通り、CustomUserDetailsServiceを実装しています。こちらの作成は任意ですが、認証情報のモデルオブジェクトをSpringSecurityのデフォルトほぼそのまま利用するケースはないので、必須と考えてよいでしょう。

|br|

.. sourcecode:: java
   :caption: CustomUserDetailsService

   package org.debugroom.mynavi.sample.aws.microservice.frontend.webapp.app.web.security;

   import org.springframework.stereotype.Service;

   import org.springframework.security.core.authority.AuthorityUtils;
   import org.springframework.security.core.userdetails.UserDetails;
   import org.springframework.security.core.userdetails.UserDetailsService;
   import org.springframework.security.core.userdetails.UsernameNotFoundException;

   @Service
   public class CustomUserDetailsService implements UserDetailsService {

       @Override
       public UserDetails loadUserByUsername(String s) throws UsernameNotFoundException {
           return CustomUserDetails.builder()
             .authorities(AuthorityUtils.createAuthorityList("ROLE_USER"))
             .build();
       }

   }

|br|

UserDetailsServiceではloadUserByUsername()メソッドを実装する必要があります。これは戻り値としてUserDetailsインターフェースを実装したクラスを返却します。
ログイン処理におけるIDとパスワードはこの返却したUserDetailsを実装したクラスオブジェクトを使ってSpringSecurityが検証処理を行います。
したがって、このメソッド内でユーザのIDとパスワードに相当する情報を取得してUserDetailsを実装したクラスオブジェクトを生成して返却する処理を実装すればよいです。
通常、データベースなどからキーとなるIDを元にパスワードが格納されているユーザ情報を取得して、UserDetailsクラスを実装したモデルオブジェクトにマッピングして返せば良いですが、
上記の例では、簡単のために特にloadUserByUsername()メソッドの引数として渡される文字列型のIDパラメータによらず、UserDetailsクラスを実装したCustomUserDetailsに、認可情報となるAuthorityListを生成・設定して返しています。
このメソッドの中で生成されるCustumUserDetailsの実装は以下の通りです。

|br|

.. sourcecode:: java
   :caption: CustomUserDetails

   package org.debugroom.mynavi.sample.aws.microservice.frontend.webapp.app.web.security;

   // omit

   import org.springframework.security.core.GrantedAuthority;
   import org.springframework.security.core.userdetails.UserDetails;

   public class CustomUserDetails implements UserDetails {                      //(A)

       private final Collection<GrantedAuthority> authorities;

       @Override
       public Collection<? extends GrantedAuthority> getAuthorities() {         //(B)
           return authorities;
       }

       @Override
       public String getPassword() {                                            //(C)
           return "{noop}test";                                                 //(D)
       }

       @Override
       public String getUsername() {                                            //(E)
           return "test";
       }

       @Override
       public boolean isAccountNonExpired() {                                   //(F)
           return true;
       }

       @Override
       public boolean isAccountNonLocked() {                                    //(G)
           return true;
       }

       @Override
       public boolean isCredentialsNonExpired() {                               //(H)
           return true;
       }

       @Override
       public boolean isEnabled() {                                             //(I)
           return true;
       }

   }

|br|

CustomUserDetailsクラスコードの説明は以下の通りです。

|br|

.. list-table:: SecurityConfigクラスコードの説明
   :widths: 1, 19

   * - 項番
     - 説明

   * - (A)
     - org.springframework.security.core.userdetails.UserDetailsを実装します。このインターフェースを実装すると(B)、(C)、(E)、(F)、(G)、(H)、(I)のメソッドをオーバーライドする必要があります。

   * - (B)
     - UserDetailsで実装が必要なメソッドです。認可に相当するGrantedAutorityをCollection型で保持して返却します。

   * - (C)
     - UserDetailsで実装が必要なメソッドです。リクエストで送信されるパスワードと一致するか検証するパスワードを返却します。

   * - (D)
     - このサンプルでは、パスワードを固定値"test"で返します。接頭辞"{noop}"は特にハッシュ化も暗号化せずに処理を行うパスワードエンコーダであるorg.springframework.security.crypto.password.NoOpPasswordEncoderを使用する場合に付与します。

   * - (E)
     - UserDetailsで実装が必要なメソッドです。リクエストで送信されるIDと一致するか検証するIDを返却します。

   * - (F)
     - UserDetailsで実装が必要なメソッドです。アカウントの期限が有効である場合にtrueを返却するよう実装します。このサンプルでは常にtrueで返却します。

   * - (G)
     - UserDetailsで実装が必要なメソッドです。アカウントのロックされていない場合にtrueを返却するよう実装します。このサンプルでは常にtrueで返却します。

   * - (H)
     - UserDetailsで実装が必要なメソッドです。認証情報が有効である場合にtrueを返却するよう実装します。このサンプルでは常にtrueで返却します。

   * - (I)
     - UserDetailsで実装が必要なメソッドです。このアカウントが有効である場合にtrueを返却するよう実装します。このサンプルでは常にtrueで返却します。

|br|

次にログインが成功した後に実行されるLoginSuccessHanderクラスです。AuthenticationSuccessHandlerを実装し、onAuthenticationSuccess()メソッドを実行しておくと、
ログインが成功した後、このハンドラクラスが呼ばれます。以下はパス"/frontend/portal"へリダイレクトする処理を実装しています。

|br|

.. sourcecode:: java
   :caption: LogoutSuccessHandler

   package org.debugroom.mynavi.sample.aws.microservice.frontend.webapp.app.web.security;

   import org.springframework.security.core.Authentication;
   import org.springframework.security.web.authentication.AuthenticationSuccessHandler;

   import javax.servlet.ServletException;
   import javax.servlet.http.HttpServletRequest;
   import javax.servlet.http.HttpServletResponse;
   import java.io.IOException;

   public class LoginSuccessHandler implements AuthenticationSuccessHandler {

       @Override
       public void onAuthenticationSuccess(HttpServletRequest httpServletRequest
         , HttpServletResponse httpServletResponse
         , Authentication authentication) throws IOException, ServletException {
           httpServletResponse.sendRedirect("/frontend/portal");
       }

   }

|br|

.. note:: ログイン後のリダイレクトは設定クラス内のフォーム認証の設定でderaultSuccressUrl()でデフォルトのページを指定することもできますが、
          ユーザの権限ごとに異なるページに遷移させたい場合や、ログイン時間を記録したりなどより高度なカスタマイズ処理を加えたい場合に利用するとよいでしょう。

|br|

続いて、認証処理で何らかのエラーが発生した場合に処理をカスタマイズしたい場合に利用するクラスを紹介します。以下の
SessionExpiredDetectingLoginUrlAuthenticationEntryPointはセッションが無効になった場合にそれを検出して、
タイムアウト画面へ遷移させるためのカスタムクラスです。通常エラーが発生した際は、org.springframework.security.web.authentication.LoginUrlAuthenticationEntryPoint
に設定されたloginFormUrlに基づいてリダイレクトされますが、その処理の過程でセッションの有効状態を検証し、無効な状態になっているようであれば、別の画面へリダイレクトさせるという例です。
LoginUrlAuthenticationEntryPointを継承して、リダイレクトURLを生成する箇所をオーバーライドして挙動をカスタマイズします。

|br|

.. sourcecode:: java
   :caption: SessionExpiredDetectingLoginUrlAuthenticationEntryPoint

   package org.debugroom.mynavi.sample.aws.microservice.frontend.webapp.app.web.security;

   import org.springframework.security.core.AuthenticationException;
   import org.springframework.security.web.authentication.LoginUrlAuthenticationEntryPoint;

   import javax.servlet.http.HttpServletRequest;
   import javax.servlet.http.HttpServletResponse;
   import java.util.Objects;

   public class SessionExpiredDetectingLoginUrlAuthenticationEntryPoint
        extends LoginUrlAuthenticationEntryPoint {

       public SessionExpiredDetectingLoginUrlAuthenticationEntryPoint(
            String loginFormUrl) {
           super(loginFormUrl);
       }

       @Override
       protected String buildRedirectUrlToLoginPage(HttpServletRequest request
         , HttpServletResponse response, AuthenticationException authException) {
           String redirectUrl = super.buildRedirectUrlToLoginPage(
             request, response, authException);
           if(isRequestedSessionInvalid(request)){
               redirectUrl = "timeout";
           }
           return redirectUrl;
       }

       private boolean isRequestedSessionInvalid(HttpServletRequest request){
           return Objects.nonNull(request.getRequestedSessionId())
             && !request.isRequestedSessionIdValid();
       }
   }

|br|

このようにSpringSecurityではアプリケーションの認証・認可を行う処理の実装をほぼほぼ提供しながらも、要件に応じて様々なカスタマイズができるようにカスタムエントリポイントとなるクラスやメソッドを多数提供しています。
最後にログイン画面やログイン成功後に遷移するポータル画面を実装しましょう。ポイントとなるのはログイン画面でエラーが発生した場合に、同じ画面に遷移しますが、エラーメッセージが表示されるようにしておくことと、
ログイン処理を行うURLのパスやフォームのIDとパスワードのリクエストパラメータ名をSpringSecurity設定クラスで指定したものと合わせておくことです。なお、ポータル画面は特に説明すべきことはないので割愛します。

なお、本稿の趣旨とは外れるので、Thymeleafテンプレートエンジンに関する説明は省略しますが、必要に応じて、`Thymeleaf公式ドキュメント <https://www.thymeleaf.org/documentation.html>`_ や
`Macchinetta Framework テンプレートエンジン <https://macchinetta.github.io/server-guideline-thymeleaf/current/ja/ArchitectureInDetail/WebApplicationDetail/Thymeleaf.html>`_ を参照してください。



|br|

.. sourcecode:: html
   :caption: templates/login.html

   <!DOCTYPE HTML>
   <html xmlns:th="http://www.thymeleaf.org" lang="ja">
   <!-- omit -->
   <body>
   <h1>Mynavi Microservice WebApp Login</h1>

   <!-- エラーが発生した場合にエラ〜メッセージを表示させる -->
   <div th:if="${param.containsKey('error')}"
      th:with="exception = ${SPRING_SECURITY_LAST_EXCEPTION} ?: ${session[SPRING_SECURITY_LAST_EXCEPTION]}">
      <ul th:if="${exception != null}" class="alert alert-error">
        <li th:text="${exception.message}"></li>
      </ul>
   </div>

   <!-- ログインフォーム：ログイン処理を行うURLパスやフォームのIDとパスワードのリクエストパラメータ名を設定クラスのものと合わせておく。 -->
   <form th:action="@{/authenticate}" method="post">
     <table>
       <tr>
         <td><label for="username">User:</label></td>
         <td><input type="text" id="username" name="username" placeholder="LoginId" value="test">(demo)</td>
       </tr>
       <tr>
         <td><label for="password">Password:</label></td>
         <td><input type="password" id="password" name="password" placeholder="Password" value="test">(demo)</td>
       </tr>
       <tr>
         <td>&nbsp;</td>
         <td><input name="submit" type="submit" value="Login"></td>
       </tr>
     </table>
   </form>
   </body>

|br|

起動クラスを実行し。アプリケーションを起動して、以下の通りログインを行うと、ポータル画面へ遷移できます。ログイン後はセッションが有効になるのでセッションIDを表示させています。

|br|

.. figure:: img/webapp/webapp-login-1.png

|br|

.. figure:: img/webapp/webapp-portal-1.png

|br|

今回はSpringSecutiyを使ったWebアプリケーションログインページの実装やカスタム設定について解説を行い、実際にアプリケーションを起動してみました。
次回以降は、RDSに保存したユーザ情報を取得するバックエンドマイクロサービスを作成し、今回作成したCustomUserDetailsServiceから呼び出すようなかたちにして
認証処理がデータベースに保存されたものを使って正しく行われるように実装しなおしてみます。その際、AWS X-Rayを使ってサービス呼び出しを可視化する方法も合わせて紹介します。

|br|

著者紹介
------------------------------------------------------------------

川畑 光平(KAWABATA Kohei) - NTTデータ

.. figure:: img/overview/aws_361383_075.jpeg

金融機関システム業務アプリケーション開発・システム基盤担当、ソフトウェア開発自動化関連の研究開発を経て、デジタル技術関連の研究開発・推進に従事。

Red Hat Certified Engineer、Pivotal Certified Spring Professional、AWS Certified Solutions Architect Professional等の資格を持ち、アプリケーション基盤・クラウドなど様々な開発プロジェクト支援にも携わる。

`AWS Top Engineers & Ambassadors <https://aws.amazon.com/jp/partners/ambassadors/?cards-body.sort-by=item.additionalFields.ambassadorName&cards-body.sort-order=asc&cards-body.q=kawabata&cards-body.q_operator=AND>`_ 選出。

本連載記事の内容に対するご意見・ご質問は `Facebook <https://www.facebook.com/kohei.kawabata.5>`_ まで。
