.. include:: ../module.txt

【第9回】マイクロサービスを呼び出すWebAPの単体テストコード・E2Eテストの実装(後編)
----------------------------------------------------------------------------------------

|br|

.. _section-application-test-2-for-web-application-label:

マイクロサービスを呼び出すWebアプリケーションの単体テスト・EndToEndテスト(後編)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

前回は、マイクロサービスを呼び出す側のWebアプリケーション(BackendForFrontend:BFFアプリケーション)におけるRepositoryやServiceの単体テストについて説明しました。
今回は引き続き、HTMLUnitを使用したControllerの単体テスト、Seleniumを使用したEndToEndテストについて説明します。

|br|

.. _section-controller-test-for-webapp-label:

マイクロサービスを呼び出すWebアプリケーションのController単体テスト実装
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|
BFFアプリケーションのControllerテストは、リクエストハンドリング・バリデーションの妥当性等、マイクロサービスでのテスト観点と重複する部分もありますが、
マイクロサービスのレスポンスがJSON文字列を返却するコンテンツタイプ「application/json」だったのに対し、BFFアプリケーションは「text/html」であるHTTPレスポンス返却が中心となります。
そこで、マイクロサービスのテストで実施した、MockMvcを使ったControllerのテスト観点に加え、生成するHTTPレスポンスの中で必要なパラメータやメッセージが含まれているか、
オープンソーステストライブラリであるHtmlUnitを使って行います。その準備として、Mavenプロジェクトのpom.xmlで、spring-boot-starter-testに加えて、HtmlUnitのライブラリを定義します。


|br|

.. sourcecode:: xml

   <dependencies>
     <dependency>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter-test</artifactId>
       <scope>test</scope>
     </dependency>
     <dependency>
       <groupId>net.sourceforge.htmlunit</groupId>
       <artifactId>htmlunit</artifactId>
     </dependency>
   </dependencies>

|br|

Controllerの単体テスト実装でのリクエストハンドリングやMock設定などの基本的な要領は、 マイクロサービスにおける :ref:`section-controller-test-for-microservice-label` とほぼ同様です。
以降は、com.gargoylesoftware.htmlunit.WebClientを使用した、HTMLの検証コードを中心に解説を進めます。なお、SpringMVCおよびMockMvcを使用したHtmlUnitの使用方法については、
`Springの公式ドキュメント HtmlUnit Integration <https://docs.spring.io/spring/docs/current/spring-framework-reference/testing.html#spring-mvc-test-server-htmlunit>`_ も適宜参照ください。
また、Webアプリケーションでは、バリデーションエラーのテストコードなどはマイクロサービスのテストのものとは異なり、HTMLページへレンダリングするために、org.springframework.validation.BindingResultにエラー項目が格納されます。
テストコードではこちらからエラーとなる項目を取り出し、期待したパラメータやメッセージが取得できるか検証を行う必要があります。
また、HTMLUnitはAJAX(非同期通信)における実行をサポートするため、画面遷移を伴わない処理も合わせて検証が可能です。サンプルテストコードは以下の通りです。

|br|

.. sourcecode:: java

   package org.debugroom.mynavi.sample.continuous.integration.bff.app.web;

   // omit

   import com.gargoylesoftware.htmlunit.BrowserVersion;
   import com.gargoylesoftware.htmlunit.BrowserVersion.BrowserVersionBuilder;
   import com.gargoylesoftware.htmlunit.NicelyResynchronizingAjaxController;
   import com.gargoylesoftware.htmlunit.WebClient;
   import com.gargoylesoftware.htmlunit.html.HtmlButton;
   import com.gargoylesoftware.htmlunit.html.HtmlElement;
   import com.gargoylesoftware.htmlunit.html.HtmlPage;
   import com.gargoylesoftware.htmlunit.html.HtmlTextInput;

   import org.springframework.beans.factory.annotation.Autowired;
   import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
   import org.springframework.test.context.junit4.SpringRunner;
   import org.springframework.test.web.servlet.MockMvc;
   import org.springframework.test.web.servlet.MvcResult;
   import org.springframework.test.web.servlet.htmlunit.MockMvcWebClientBuilder;

   // omit

   @RunWith(SpringRunner.class)                                       // …(A)
   @WebMvcTest(controllers = BackendForFrontendController.class)      // …(B)
   public static class UnitTest{

       WebClient webClient;

       @Autowired
       MockMvc mockMvc;

       @MockBean
       OrchestrateService orchestrateServiceMock;

       // omit

       @Before
       public void setUp() throws Exception{

           BrowserVersionBuilder browserVersionBuilder = new BrowserVersionBuilder(BrowserVersion.CHROME);
           browserVersionBuilder.setBrowserLanguage("jp_JP");
           webClient = MockMvcWebClientBuilder.mockMvcSetup(mockMvc)
                   .withDelegate(new WebClient(browserVersionBuilder.build()))
                   .build();                                                        // …(C)
           webClient.setAjaxController(new NicelyResynchronizingAjaxController());  // …(D)

           // omit
       }

       @Test
       public void getUsersTest() throws Exception{
           HtmlPage page = webClient.getPage("http://localhost:8080/getUsers");     // …(E)
           assertThat(page.getTitleText(), is("GetUsers"));
           assertThat(page.getElementById("td-firstName-0")
                .getFirstChild().asText(), is("太郎"));
                                                                                    // …(F)
           // omit
       }

       @Test
       public void isUsableLoginIdNormalTest() throws Exception{
           HtmlPage page = webClient.getPage("http://localhost:8080/portal");
           HtmlButton htmlButton = (HtmlButton)page.getElementById("isUsableLoginIdButton-0");
           // omit
           HtmlPage updatePage = htmlButton.click();
           webClient.waitForBackgroundJavaScript(10000);
           HtmlElement htmlElement = (HtmlElement)updatePage.getElementById("message-panel");

           assertThat(htmlElement.getFirstChild().asText(), is("使用可能なログインIDです。"));
                                                                                    // …(G)
       }

       // omit

       @Test
       public void addUsersInputParamTest() throws Exception{
           MultiValueMap<String, String> params = new LinkedMultiValueMap<String, String>();
           params.set("users[0].firstName", "taro");
           // omit
           MvcResult mvcResult = mockMvc.perform(MockMvcRequestBuilders
                                     .post("/addUsers").params(params)).andReturn();
           ModelAndView modelAndView = mvcResult.getModelAndView();
                                                                                    // …(H)

           assertThat(modelAndView.getViewName(), is("portal"));
           BindingResult bindingResult = (BindingResult)
              modelAndView.getModel().get("org.springframework.validation.BindingResult.addUsersForm");

           List<FieldError> fieldErrors = bindingResult.getFieldErrors();
           assertThat(fieldErrors.size(), is(6));
                                                                                    // …(I)
           // omit
       }

   }

|br|

.. list-table:: BFFアプリケーションのController単体テストコードの説明
   :widths: 1, 19

   * - 項番
     - 説明

   * - (A)
     - テストランナーとして、SpringRunnerを指定します。

   * - (B)
     - @WebMvcTestでテスト対象のControllerを指定します。

   * - (C)
     - ブラウザとして日本語ロケールのChromeを指定して、セットアップメソッドでWebClientを構築します。

   * - (D)
     - 非同期通信(AJAX)のテストを行うためにAjaxControllerを設定しておきます。

   * - (E)
     - WebClientを使用して、テスト対象のHTMLページを取得します。

   * - (F)
     - HTMLページに含まれる項目のアサーションロジックを実装します。

   * - (G)
     - HTMLページに含まれるAJAX処理を実行し、処理実行時間を考慮してWAITした上で、処理実行後に得られる項目を検証します。

   * - (H)
     - リクエストパラメータを設定し、テスト対象のControllerメソッドを呼び出したのち、ModelAndViewを取得します。

   * - (I)
     - BindingResultを取得し、エラーとなっている項目を取得して検証します。

|br|

上記の通り、マイクロサービスを呼び出しをモック化して、取得したデータが正しく画面に表示されるか、入力チェックエラー発生時に正しいメッセージやパラメータが含まれるかを検証できます。
マイクロサービスにおけるControllerの単体テスト同様、ControllerやFormの設定誤りはセキュリティホールに直結しますので、境界値試験など含め、リクエストパラメータの異常系バリエーションを充実させて検証した方が良いでしょう。
また、後述しますが、後続のSeleniumを使用したEndToEndテストは実装・処理コストが大きいため、マイクロサービスの時とは異なり、できるだけController単体試験に寄せて試験を行う方が得策です。

サンプルとして実装したテストケースと検証観点は以下になります。

.. _BackendForFrontendController#getUsers: https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend-for-frontend/src/main/java/org/debugroom/mynavi/sample/continuous/integration/bff/app/web/BackendForFrontendController.java#L74
.. _BackendForFrontendControllerTest#getUsersTest(): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend-for-frontend/src/test/java/org/debugroom/mynavi/sample/continuous/integration/bff/app/web/BackendForFrontendControllerTest.java#L178
.. _BackendForFrontendController#getUser: https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend-for-frontend/src/main/java/org/debugroom/mynavi/sample/continuous/integration/bff/app/web/BackendForFrontendController.java#L69
.. _BackendForFrontendControllerTest#getUserAbnormalTest(): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend-for-frontend/src/test/java/org/debugroom/mynavi/sample/continuous/integration/bff/app/web/BackendForFrontendControllerTest.java#L213
.. _BackendForFrontendController#isUsableLoginId: https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend-for-frontend/src/main/java/org/debugroom/mynavi/sample/continuous/integration/bff/app/web/BackendForFrontendController.java#L81
.. _BackendForFrontendControllerTest#isUsableLoginIdNormalTest(): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend-for-frontend/src/test/java/org/debugroom/mynavi/sample/continuous/integration/bff/app/web/BackendForFrontendControllerTest.java#L228
.. _BackendForFrontendControllerTest#isUsableLoginIdAbnormalTest(): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend-for-frontend/src/test/java/org/debugroom/mynavi/sample/continuous/integration/bff/app/web/BackendForFrontendControllerTest.java#L242
.. _BackendForFrontendController#addUsers: https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend-for-frontend/src/main/java/org/debugroom/mynavi/sample/continuous/integration/bff/app/web/BackendForFrontendController.java#L88
.. _BackendForFrontendControllerTest#addUsersInputParamTest(): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend-for-frontend/src/test/java/org/debugroom/mynavi/sample/continuous/integration/bff/app/web/BackendForFrontendControllerTest.java#L256

|br|

.. list-table::
   :widths: 7, 6, 7

   * - ユースケース
     - 主な処理実装クラス・メソッド |br| テストメソッド
     - 検証観点

   * - [正常系]ユーザ一覧画面を表示する
     - `BackendForFrontendController#getUsers`_  |br| |br|  `BackendForFrontendControllerTest#getUsersTest()`_
     - ・サービス実行結果が正しく画面に表示されるか

   * - [異常系]ユーザ情報を取得する
     - `BackendForFrontendController#getUser`_  |br| |br|  `BackendForFrontendControllerTest#getUserAbnormalTest()`_
     - ・入力チェックやビジネスエラー発生時に正しいメッセージやパラメータを返却するか

   * - [正常系]ログインIDが使用可能か判定する(非同期通信)
     - `BackendForFrontendController#isUsableLoginId`_  |br| |br|  `BackendForFrontendControllerTest#isUsableLoginIdNormalTest()`_
     - ・非同期通信の実行結果が正しく画面に表示されるか

   * - [異常系]ログインIDが使用可能か判定する(非同期通信)
     - `BackendForFrontendController#isUsableLoginId`_  |br| |br|  `BackendForFrontendControllerTest#isUsableLoginIdAbnormalTest()`_
     - ・非同期通信の実行結果が正しく画面に表示されるか

   * - [異常系]複数のユーザを追加する
     - `BackendForFrontendController#addUsers`_  |br| |br|  `BackendForFrontendControllerTest#addUsersInputParamTest()`_
     - ・入力チェックやビジネスエラー発生時に正しいメッセージやパラメータを返却するか

|br|

.. _section-e2e-test-for-webapp-label:

Seleniumを使用したEndToEndテスト
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

BFFアプリケーションの単体テストが一通り終わったタイミングで、マイクロサービスのときと同様、Service⇔Repository、Controller⇔Service⇔Repositoryの結合テストを実施するのも一つの考え方ですが、
Repositoryがバックエンドのマイクロサービスを呼び出すことや、単体テストである程度異常系テストケースの網羅が可能であることから、次のステップとして、結合テストというよりか、
EndToEndテスト(以降、E2Eテスト)としてバックエンドのマイクロサービスの呼び出しを含め検証を行うこととします。

|br|

.. figure:: img/automation_infra_devops_test/webapp-e2e-test-scope.png


|br|

E2Eテストは一般にエンドユーザの操作を想定したユースケースを元に実行結果を検証します。ここではオープンソースのGUI自動化テストツールとして多くの実績があるSeleniumを使って、テストコードを実装します。
なお、WebDriverなどSeleniumで提供されるライブラリは様々な言語で利用可能ですが、Java版ではJUnitコードでのスクリプト記述が可能であり、Springが提供するMockMvcと統合して使用することが可能です。必要に応じて、
`Springの公式ドキュメント MockMvc and WebDriver <https://docs.spring.io/spring/docs/current/spring-framework-reference/testing.html#spring-mvc-test-server-htmlunit-webdriver>`_ も適宜参照ください。
SpringBootアプリケーションのテストでSeleniumを使用するには、以下の通り、Mavenプロジェクトのpom.xmlで、spring-boot-starter-testに加えて、Seleniumのライブラリを定義します。


|br|

.. sourcecode:: xml

   <dependencies>
     <dependency>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter-test</artifactId>
       <scope>test</scope>
     </dependency>
     <dependency>
       <groupId>org.seleniumhq.selenium</groupId>
       <artifactId>selenium-java</artifactId>
     </dependency>
   </dependencies>

|br|

また、ローカル環境では、Seleniumの実行にドライバのインストールが必要です。`Seleniumのインストール手順 <https://selenium-python.readthedocs.io/installation.html>`_ に従って、ドライバをインストールしてください。
以降は/usr/local/binにchromedriverがインストールされた前提で話を進めます。また、このE2Eテストではバックエンドのマイクロサービスを呼び出すため、事前にBackendアプリケーションを起動しておく必要があります。
マイクロサービスのController⇔Sevice⇔Repositoryの結合テスト同様に@SpringBootTestを使って実装した、サンプルのテストコードは以下の通りです。

|br|

.. sourcecode:: java

   package org.debugroom.mynavi.sample.continuous.integration.bff.app.web;

    // omit
   import org.junit.experimental.categories.Category;

   // omit

   import org.springframework.boot.test.context.SpringBootTest;
   import org.springframework.boot.web.server.LocalServerPort;

   import org.openqa.selenium.By;
   import org.openqa.selenium.OutputType;
   import org.openqa.selenium.TakesScreenshot;
   import org.openqa.selenium.WebDriver;
   import org.openqa.selenium.chrome.ChromeDriver;

   // omit

   import org.debugroom.mynavi.sample.continuous.integration.common.apinfra.test.junit.E2ETest;

   // omit

   @RunWith(SpringRunner.class)                                   // …(A)
   @SpringBootTest(classes = {
     TestConfig.EndToEndTestConfig.class,
     BackendForFrontendControllerTest.EndToEndTest.Config.class
   }, webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT) // …(B)
   @Category(E2ETest.class)                                       // …(C)
   public static class EndToEndTest{

       @Configuration
       public static class Config{

           @Autowired
           SeleniumProperties seleniumProperties;                 // …(D)

           @Bean
           @Profile("dev")
           WebDriver webDriver(){
               System.setProperty("webdriver.chrome.driver", seleniumProperties.getChromeDriverPath());
               ChromeOptions options = new ChromeOptions();
               //omit
               return new ChromeDriver(options);                  // …(E)
           }

       }

       @Value("#{servletContext.contextPath}")
       private String contextPath;

       @Autowired
       SeleniumProperties seleniumProperties;

       @LocalServerPort
       private int port;

       @Autowired(required = false)
       WebDriver webDriver;                                       // …(F)

       @Autowired
       PortalPage portalPage;                                     // …(G)

       // omit

       @Test
       public void addUsers_2_AbnormalTest(){
           webDriver.get("http://localhost:" + port + contextPath + "/portal");
           webDriver.findElement(By.id("addFormButton-0")).click();
           portalPage.setAddUserForm1("saburo", "mynavi",
                   "saburo.mynavi1", "100-0000", "Tokyo　Minato",
                   "saburo.mynavi1@debugroom.org");
           portalPage.setAddUserForm2("jiro", "mynavi",
                   "jiro.mynavi1", "300-0000", "Tonde　Saitama",
                   "jiro.mynavi1@debugroom.org");
           webDriver.findElement(By.id("addUsersButton")).click();
           webDriver.getPageSource().contains("使用できないログインIDです。LoginID : jiro.mynavi1");
           File file = ((TakesScreenshot) webDriver).getScreenshotAs(OutputType.FILE);
           file.renameTo(new File(seleniumProperties.getEvidencePath() + "/addUserAbnormalTest_screenshot.png"));
                                                                  // …(H)
       }
      // omit

|br|

.. list-table:: BFFアプリケーションのE2Eテストコードの説明
   :widths: 1, 19

   * - 項番
     - 説明

   * - (A)
     - テストランナーとして、SpringRunnerを指定します。

   * - (B)
     - @SpringBootTestアノテーションには、テスト向け固有の設定クラスを任意に指定し、Webコンテナ(Server)の起動時のポートをランダムで指定しておきます。

   * - (C)
     - テストクラスを種別で分類するためにCategoryアノテーションを付与します。テスト実行にはBackendアプリケーションの起動が前提となりますので、pom.xmlのデフォルトプロファイル内でmaven-surefire-pluginを設定し、Mavenビルド時に、デフォルトではE2ETestインターフェースのカテゴリのテストは実行されないようにしておきます。

   * - (D)
     - Chromeドライバのパスや実行時のスクリーンショット画像を保存するパスなどをプロパティとして定義して利用します。

   * - (E)
     - WebDriverにChromeドライバを設定します。インスタンス生成前にシステム変数webdriver.chrome.driverにChromeドライバのパスを設定しておきます。

   * - (F)
     - WebDriverをインジェクションして利用します。次回以降後述しますが、バックエンドのマイクロサービスの起動が前提となるE2Eテストはビルド時にテストを実施しないよう設定し、WebDriverが除外されてもテストクラスが問題ないようrequiredオプションをfalseに設定しておきます。

   * - (G)
     - テスト対象のHTMLページを、PageObjectパターン※でPageObjectとして実装しておき、インジェクションして利用します。今回サンプルとして実装したPageObjectは `こちら <https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend-for-frontend/src/test/java/org/debugroom/mynavi/sample/continuous/integration/bff/app/web/selenium/PortalPage.java>`_ を参照してください。

   * - (H)
     - WebDriverでテスト対象のページに遷移し、テストパラメータを設定して処理ボタンを押下し、結果の文字列をアサーションします。最後に実行結果の画面をエビデンスとして指定したディレクトリに保存します。

|br|

.. note:: ※PageObjectパターンとは、`Selenium 公式サイトでも推奨されているパターン <https://www.seleniumhq.org/docs/06_test_design_considerations.jsp#page-object-design-pattern>`_ で、HTMLページをPageObjectとして実装し、テストで関係する主なHTML要素を変数として定義して、WebDriverのアクセスを一元的に行い実装効率性を高めるデザインパターンです。

|br|

テスト実行すると、Chromeブラウザが起動し、テストケース操作が自動実行されます。最後に画面のスクリーンショットを保存して、画面表示のレイアウトが崩れていないか等目視確認します。

|br|

.. figure:: img/automation_infra_devops_test/addUserAbnormalTest_screenshot.png


|br|

上記を含め、サンプルとして実装したテストケースと検証観点は以下になります。必要に応じて、ブラウザごとの表示の差異やデータベースへの反映状況なども検証すると良いでしょう。

.. _BackendForFrontendControllerTest#E2E#getUsersTest(): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend-for-frontend/src/test/java/org/debugroom/mynavi/sample/continuous/integration/bff/app/web/BackendForFrontendControllerTest.java#L380
.. _BackendForFrontendControllerTest#E2E#addUsers_1_NormalTest(): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend-for-frontend/src/test/java/org/debugroom/mynavi/sample/continuous/integration/bff/app/web/BackendForFrontendControllerTest.java#L393
.. _BackendForFrontendControllerTest#E2E#addUsers_2_AbnormalTest(): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend-for-frontend/src/test/java/org/debugroom/mynavi/sample/continuous/integration/bff/app/web/BackendForFrontendControllerTest.java#L421

|br|

.. list-table::
   :widths: 7, 6, 7

   * - ユースケース
     - 主な処理実装クラス・メソッド |br| テストメソッド
     - 検証観点

   * - [正常系]ユーザ一覧画面を表示する
     - `BackendForFrontendController#getUsers`_  |br| |br|  `BackendForFrontendControllerTest#E2E#getUsersTest()`_
     - ・ユースケースシナリオ通り操作した時に、正しく画面に結果が表示されるか |br| ・画面表示のレイアウトが崩れていないか

   * - [正常系]複数ユーザを追加する
     - `BackendForFrontendController#addUsers`_  |br| |br|  `BackendForFrontendControllerTest#E2E#addUsers_1_NormalTest()`_
     - ・ユースケースシナリオ通り操作した時に、正しく画面に結果が表示されるか |br| ・画面表示のレイアウトが崩れていないか

   * - [異常系]複数ユーザを追加する
     - `BackendForFrontendController#addUsers`_  |br| |br|  `BackendForFrontendControllerTest#E2E#addUsers_2_AbnormalTest()`_
     - ・ユースケースシナリオ通り操作した時に、エラーメッセージが正しく表示されるか |br| ・画面表示のレイアウトが崩れていないか

|br|

.. warning:: E2Eテストの実行は実装にかかる工数もさることながら、ブラウザの起動やスクリーンキャプチャ処理などが挟まることで、テスト実行時間も相応の時間を要します。実装するテストケースは最小限に絞り、異常系のパターンバリエーションなどは可能な限り、相対的にコストが低い単体テストで検証するようにしてください。

|br|

.. _section-test-strategy-for-webapp-label:

マイクロサービスを呼び出すWebアプリケーションのテスト戦略
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

これまで、Repository、Service、Controllerの単体テストと、Seleniumを使ったE2E自動化テストコード実装を解説してきました。
これまでの説明の再掲にはなりますが、初めから完璧にテストコードを整備しておく必要もありませんし、必要以上のテストコードの実装でかえって開発のアジリティを損なうようでは本末転倒です。
ただ、テストが疎かになるとせっかくの継続的インテグレーションも機能しません。開発のスピードと品質を両立するために、テスト計画やスコープ、検証の観点を明示的に策定しておくことが重要です。

マイクロサービスを呼び出す側のWebアプリケーションの効率的なテスト戦略策定のポイントとしては、以下のような点が挙げられるでしょう。

* バックエンドのマイクロサービスは別途試験されていることから、結合試験を除外し、E2Eテストにフォーカスする。
* E2Eテストは実装工数や処理時間など高コストになりがちなため、できるだけ異常系のバリエーションなどは単体テストで行う。
* 探索的テストを導入し、実装状況に応じてテストケースの重複を極力減らしながらテストコードを作成する
* 機能や処理の重要度に応じて、テスト実施内容に濃淡をつける(ビジネス的にそこまで重要でない処理の参照系はテストしない等)

|br|

次回はこれまで実装したソースコード・テストコードに対し、AWS CodeBuildを使用して、ビルドやテスト実行、SonarQubeとの連携について、実践していきます。

|br|

著者紹介
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

川畑 光平(KAWABATA Kohei) - NTTデータ 課長代理

.. figure:: img/automation_infra_devops_overview/pic_image01.jpg


金融機関システム業務アプリケーション開発・システム基盤担当を経て、現在はソフトウェア開発自動化関連の研究開発・推進に従事。

Red Hat Certified Engineer、Pivotal Certified Spring Professional、AWS Certified Solutions Architect Professional等の資格を持ち、アプリケーション基盤・クラウドなど様々な開発プロジェクト支援にも携わる。

`2019 APN AWS Top Engineers & Ambassadors <https://aws.amazon.com/jp/blogs/psa/japan-apn-ambassador-2019/>`_ 選出。
