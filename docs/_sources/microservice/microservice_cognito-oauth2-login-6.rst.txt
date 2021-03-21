.. include:: ../module.txt

.. _section-aws-microservice-cognito-oauth2-login-6-label:

【第17回】Amazon Cognito + Spring Sercurityを使ったOAuth2 Loginの実装(6)
===============================================================================================

|br|

前回から、以下のイメージのようにOAuth2 Loginをベースとしたアーキテクチャを想定した環境構築を進めています。

|br|

.. figure:: img/cognito-oauth2-login/oauth2-login-flow.png

|br|

前回は、CloudFormationで構築したAmazon Cognitoのアプリクライアントからクライアントシークレットを取得し、
Parameter Storeへ設定するLambdaファンクションを実装しました。今回は引き続き、Cognitoへユーザを登録・サインアップしテータスを更新するLambdaファンクションの実装方法を解説していきます。

なお、実際のソースコードは `GitHub <https://github.com/debugroom/mynavi-sample-aws-microservice/tree/feature_4_cognito-oauth2-login>`_ 上にコミットしています。以降のソースコードでは本質的でない記述を一部省略しているので、実行コードを作成する場合は、必要に応じて適宜GitHub上のソースコードも参照してください。

|br|

Cognitoにユーザを登録するLambdaファンクションの実装
----------------------------------------------------------------------------------

|br|

続いて、Cognitoユーザプールへユーザを追加するLambdaファンクションを実装します。ファンクションクラスの基本的な構成は、前回実装したParameter Storeへクライアントシークレットを設定するものと同じです。

|br|

.. sourcecode:: java

   package org.debugroom.mynavi.sample.aws.microservice.lambda.app.function;

   // omit

   @Slf4j
   public class AddCognitoUserFunction implements Function<Map<String, Object>, Message<String>> { //(A)

       @Autowired
       ServiceProperties serviceProperties; //(B)

       @Autowired
       CloudFormationStackResolver cloudFormationStackResolver; //(C)

       @Autowired
       AWSCognitoIdentityProvider awsCognitoIdentityProvider;

       @Override
       public Message<String> apply(Map<String, Object> stringObjectMap) {
           log.info(this.getClass().getName() +  "  has started!");
           String userPoolId = cloudFormationStackResolver.getExportValue(
             serviceProperties.getCloudFormation().getCognito().getUserPoolId()); //(D)
           ListUsersRequest listUsersRequest = new ListUsersRequest().withUserPoolId(userPoolId);
           ListUsersResult listUsersResult = awsCognitoIdentityProvider.listUsers(
             listUsersRequest);

          int numberOfCognitoUser = listUsersResult.getUsers().size(); //(E)
          List<AttributeType> attributeTypes = new ArrayList<>();
          AdminCreateUserRequest adminCreateUserRequest = new AdminCreateUserRequest()
             .withUserPoolId(userPoolId)
             .withTemporaryPassword("test01");
          attributeTypes.add(new AttributeType().withName("family_name").withValue("mynavi"));
          attributeTypes.add(new AttributeType().withName("given_name").withValue("taro"));
          attributeTypes.add(new AttributeType().withName("custom:isAdmin").withValue("1"));
          if(numberOfCognitoUser != 0){
              adminCreateUserRequest.withUsername("taro.mynavi" + Integer.toString(numberOfCognitoUser));
              attributeTypes.add(new AttributeType()
                 .withName("custom:loginId").withValue("taro.mynavi" + Integer.toString(numberOfCognitoUser)));//(F)
          }else {
              adminCreateUserRequest.withUsername("taro.mynavi");
              attributeTypes.add(new AttributeType().withName("custom:loginId").withValue("taro.mynavi"));
          }
              adminCreateUserRequest.withUserAttributes(attributeTypes);
              AdminCreateUserResult adminCreateUserResult = awsCognitoIdentityProvider.adminCreateUser(
                  adminCreateUserRequest); //(G)
              return MessageBuilder.withPayload("Complete!").build();
          }

   }

|br|

.. list-table::
   :widths: 1, 9

   * - 項番
     - 説明

   * - A
     - java.util.function.Functionを実装します。Input型としてハンドラクラスで生成したMapオブジェクトクラスを、Output型としてorg.springframework.messaging.Messageを指定します。

   * - B
     - CloudFormationのスタックから取得するためのエクスポート名をプロパティクラスに保持します。実際のエクスポート名はapplicaiton.ymlに記載します。

   * - C
     - CloudFormationのスタックからOutput要素で出力した値を取得するユーティリティクラスをインジェクションします。実装は `こちら <https://github.com/debugroom/mynavi-sample-aws-microservice/blob/master/common/src/main/java/org/debugroom/mynavi/sample/aws/microservice/common/apinfra/cloud/aws/CloudFormationStackResolver.java>`_ ですが、CloudFormationのSDKクライアントを使ってエクスポート値を取得します。

   * - D
     - ユーザプールIDを(C)を使って取得します。

   * - E
     - ユーザプールに既に登録されているユーザの数を取得します。

   * - F
     - Eの数に応じて、ログインIDや、ユーザIDに相当するusernameが重複しないように設定します。

   * - G
     - awsCognitoIdentityProvider.adminCreateUser()メソッドを実行して、ユーザを登録します。


|br|

Cognitoユーザのサインアップステータスを変更するLambdaファンクションの実装
----------------------------------------------------------------------------------

|br|
続いて、前節で実装したCognitoユーザ作成後にサインアップステータスを変更するLambdaファンクションを実装します。前節と同様、基本的なLambdaファンクション実装の構成は同じです。

|br|

.. sourcecode:: java

   package org.debugroom.mynavi.sample.aws.microservice.lambda.app.function;

   import java.nio.charset.StandardCharsets;
   import java.util.Base64;

   // omit

   import javax.crypto.Mac;
   import javax.crypto.spec.SecretKeySpec;

   @Slf4j
   public class ChangeCognitoUserStatusFunction implements Function<Map<String, Object>, Message<String>> {

       @Autowired
       ServiceProperties serviceProperties;

       @Autowired
       CloudFormationStackResolver cloudFormationStackResolver;

       @Autowired
       AWSCognitoIdentityProvider awsCognitoIdentityProvider;

       @Autowired
       AWSSimpleSystemsManagement awsSimpleSystemsManagement;

       @Override
       public Message<String> apply(Map<String, Object> stringObjectMap) {
           log.info(this.getClass().getName() +  "  has started!");

           String userPoolId = cloudFormationStackResolver.getExportValue(
             serviceProperties.getCloudFormation().getCognito().getUserPoolId());
           String appClientId = cloudFormationStackResolver.getExportValue(
             serviceProperties.getCloudFormation().getCognito().getAppClientId());
           String clientSecret = getParameterFromParameterStore(
             serviceProperties.getSystemsManagerParameterStore().getCognito().getAppClientSecret(), true);
           ListUsersRequest listUsersRequest = new ListUsersRequest().withUserPoolId(userPoolId);
           ListUsersResult listUsersResult = awsCognitoIdentityProvider.listUsers(
             listUsersRequest); //(A)

           listUsersResult.getUsers().stream()
             .filter(userType -> Objects.equals(userType.getUserStatus(), "FORCE_CHANGE_PASSWORD")) //(B)
             .forEach(userType -> {
                 AdminInitiateAuthRequest adminInitiateAuthRequest
                         = adminInitiateAuthRequest(userType.getUsername(), userPoolId, appClientId, clientSecret); //(C)
                 AdminInitiateAuthResult  adminInitiateAuthResult =
                         awsCognitoIdentityProvider.adminInitiateAuth(adminInitiateAuthRequest); //(D)

                 if(Objects.equals(adminInitiateAuthResult.getChallengeName(), "NEW_PASSWORD_REQUIRED")){ //(E)
                     Map<String, String> challengeResponses = new HashMap<>();
                     challengeResponses.put("USERNAME", userType.getUsername());
                     challengeResponses.put("NEW_PASSWORD", "test02"); //(F)
                     challengeResponses.put("userAttributes.given_name",
                             userType.getAttributes().stream().filter(attributeType ->
                                 Objects.equals(attributeType.getName(), "given_name")).findFirst().get().getValue()); //(G)
                     challengeResponses.put("SECRET_HASH", calculateSecretHash(appClientId, clientSecret, userType.getUsername())); //(H)

                     AdminRespondToAuthChallengeRequest adminRespondToAuthChallengeRequest =
                             new AdminRespondToAuthChallengeRequest()
                             .withChallengeName(adminInitiateAuthResult.getChallengeName())
                             .withUserPoolId(userPoolId)
                             .withClientId(appClientId)
                             .withSession(adminInitiateAuthResult.getSession()) //(I)
                             .withChallengeResponses(challengeResponses); //(J)

                     AdminRespondToAuthChallengeResult adminRespondToAuthChallengeResult =
                             awsCognitoIdentityProvider.adminRespondToAuthChallenge(adminRespondToAuthChallengeRequest); //(K)
                 }

             });

           return MessageBuilder.withPayload("Complete!").build();
       }

       private AdminInitiateAuthRequest adminInitiateAuthRequest(
         String userName, String userPoolId, String appClientId, String clientSecret){
           AdminInitiateAuthRequest adminInitiateAuthRequest = new AdminInitiateAuthRequest();
           adminInitiateAuthRequest.setAuthFlow(AuthFlowType.ADMIN_USER_PASSWORD_AUTH); //(L)
           adminInitiateAuthRequest.setUserPoolId(userPoolId);
           adminInitiateAuthRequest.setClientId(appClientId);
           Map<String, String> authParameters = new HashMap<>();
           authParameters.put("USERNAME", userName);
           authParameters.put("PASSWORD", "test01");
           authParameters.put("SECRET_HASH", calculateSecretHash(appClientId, clientSecret, userName)); //(M)
           adminInitiateAuthRequest.setAuthParameters(authParameters);
           return adminInitiateAuthRequest;
       }

       private static String calculateSecretHash(String userPoolClientId,
           String userPoolClientSecret, String userName) { //(N)
           final String HMAC_SHA256_ALGORITHM = "HmacSHA256";

           SecretKeySpec signingKey = new SecretKeySpec(
             userPoolClientSecret.getBytes(StandardCharsets.UTF_8),
             HMAC_SHA256_ALGORITHM);
           try {
               Mac mac = Mac.getInstance(HMAC_SHA256_ALGORITHM);
               mac.init(signingKey);
               mac.update(userName.getBytes(StandardCharsets.UTF_8));
               byte[] rawHmac = mac.doFinal(userPoolClientId.getBytes(StandardCharsets.UTF_8));
               return Base64.getEncoder().encodeToString(rawHmac);
           } catch (Exception e) {
               throw new RuntimeException("Error while calculating ");
           }
       }

       private String getParameterFromParameterStore(String paramName, boolean isEncripted){
           GetParameterRequest request = new GetParameterRequest();
           request.setName(paramName);
           request.setWithDecryption(isEncripted);
           return awsSimpleSystemsManagement.getParameter(request).getParameter().getValue();
       }

   }

|br|

.. list-table::
   :widths: 1, 9

   * - 項番
     - 説明

   * - A
     - ユーザプールID、アプリクライアントIDをCloudFormationのスタック情報から、クライアントシークレットをSystems Manager Parameter Storeから取得し、ユーザプールに存在するユーザの一覧を取得します。

   * - B
     - 取得したユーザの中で、ステータスが「FORCE_CHANGE_PASSWORD」である初期ユーザを抽出します。

   * - C
     - Cognitoユーザをサーバ側でサインアップ処理を完了する場合、`ユーザプールの認証フロー サーバ側の認証フロー <https://docs.aws.amazon.com/ja_jp/cognito/latest/developerguide/amazon-cognito-user-pools-authentication-flow.html#amazon-cognito-user-pools-server-side-authentication-flow>`_ に従って、ユーザを認証するために、AdminInitiateAuthRequestを生成します。

   * - D
     - AdminInitiateAuth APIを呼び出し、AdminRespondToAuthChallengeRequestの生成に必要なセッションパラメータを含むAdminInitiateAuthResultを取得します。

   * - E
     - AdminInitiateAuthResultのチャレンジ名が「NEW_PASSWORD_REQUIRED」だった場合に、チャレンジレスポンスを生成し、AdminRespondToAuthChallenge APIを呼び出します。

   * - F
     - 新たなパスワードを「test02」に変更します。

   * - G
     - チャレンジレスポンスに「userAttributes.given_name」を含めておきます。これはユーザプールで自身が必須定義した内容に応じて必要となるパラメータです。

   * - H
     - チャレンジレスポンスに「SECRET_HASH」を含めておく必要があります。SECRET_HASHの実装の詳細は(N)を参照してください。

   * - I
     - AdminRespondToAuthChallengeRequestにAdminInitiateAuthResultが持つ「セッションパラメータ」が必要になります。詳細は、`AdminRespondToAuthChallenge Session <https://docs.aws.amazon.com/ja_jp/cognito-user-identity-pools/latest/APIReference/API_AdminRespondToAuthChallenge.html#API_AdminRespondToAuthChallenge_RequestSyntax>`_ も参照してください。

   * - J
     - AdminRespondToAuthChallengeRequestにチャレンジレスポンスを設定します。

   * - K
     - AdminRespondToAuthChalleng APIを呼び出して、ユーザのステータスを変更します。

   * - L
     - AdminInitiateAuthRequestでは、認証フローを「ADMIN_USER_PASSWORD_AUTH」で設定します。

   * - M
     - AdminInitiateAuthRequestにおいても、「SECRET_HASH」を含めておきます。SECRET_HASHの実装の詳細は(N)を参照してください

   * - N
     -  Amazon CognitoのユーザプールAPIは、ユーザプールIDやアプリクライアントID、クライアントシークレットを元にSecretHash値が必要なものがあります。SecretHash値が必要なAPI及びその計算方法は `ユーザーアカウントのサインアップと確認 SecretHash 値の計算 <https://docs.aws.amazon.com/ja_jp/cognito/latest/developerguide/signing-up-users-in-your-app.html>`_ を参照してください。

|br|

今回は、Cognitoユーザプールにユーザを追加し、サインアップステータスを変更するLambdaファンクションを実装しました。
次回は、前回実装したクライアントシークレットをParameter Storeへ登録するLambdaファンクションを含めて、カスタムリソースとして実行するCloudFormationテンプレートを作成し、実行してみます。

|br|

著者紹介
------------------------------------------------------------------

川畑 光平(KAWABATA Kohei) - NTTデータ エグゼクティブ ITスペシャリスト ソフトウェアアーキテクト・デジタルテクノロジーストラテジスト(クラウド)

.. figure:: img/overview/aws_361383_075.jpeg

金融機関システム業務アプリケーション開発・システム基盤担当、ソフトウェア開発自動化関連の研究開発を経て、デジタル技術関連の研究開発・推進に従事。

Red Hat Certified Engineer、Pivotal Certified Spring Professional、AWS Certified Solutions Architect Professional等の資格を持ち、アプリケーション基盤・クラウドなど様々な開発プロジェクト支援にも携わる。

`AWS Top Engineers & Ambassadors <https://aws.amazon.com/jp/partners/ambassadors/?cards-body.sort-by=item.additionalFields.ambassadorName&cards-body.sort-order=asc&cards-body.q=kawabata&cards-body.q_operator=AND>`_ 選出。

本連載記事の内容に対するご意見・ご質問は `Facebook <https://www.facebook.com/kohei.kawabata.5>`_ まで。
