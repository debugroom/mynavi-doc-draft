.. include:: ../module.txt

.. _section-cloud-native-api-gateway-and-lambda-label:

AWSで作るクラウドネイティブアプリケーションの基本
========================================================================================

第1回 API GatewayとLambdaを使ったサーバレスSpringアプリケーション
----------------------------------------------------------------------------------------

|br|

クラウド時代が到来し、ますます広がりを見せつつあるサーバレス開発。初回はAWS LambdaとAPI Gatewayを使ったサーバレス環境下でSpringアプリケーションを構築する方法を解説します。
本稿は以下の前提知識がある開発者を想定しています。

|br|

.. list-table::
   :widths: 3, 7

   * - 対象読者
     - 前提知識

   * - エンタープライズ開発者
     - Java言語及びSpringFrameworkを使った開発に従事したことがある経験者。経験がなければ、|br|
       `こちらのチュートリアル <http://terasolunaorg.github.io/guideline/5.4.1.RELEASE/ja/Tutorial/index.html>`_ を実施することを推奨します。TERASOLUNAはSpringのベストプラクティスをまとめた開発方法論で、このチュートリアルでは、JavaやSpring Frameworkを使った開発に必要な最低限の知識を得ることができます。

   * -
     - GitHubなどのバージョン管理ツールやApache Mavenなどのライブラリ管理ツールを使った開発に従事したことがある経験者。

   * - AWS開発経験者
     - AWSアカウントをもち、コンソール上で各サービスを実行したことがある経験者

|br|

また、動作環境は以下のバージョンで実施しています、将来的には、AWSコンソール上の画面イメージや操作、バージョンアップによりJavaのソースコード内で使用するクラスが異なる可能性があります。

|br|


.. list-table::
   :widths: 5, 5

   * - 動作対象
     - バージョン

   * - Java
     - 1.8

   * - Spring Boot
     - 2.1.2.RELEASE

   * - Spring Cloud Function
     - 2.0.0.RC

|br|


Amazon API GatewayとAWS Lambdaでのサーバレス開発
------------------------------------------------------------------

|br|

通常、アプリケーション開発において、リクエストを受け付けるためのアプリケーションサーバ・それを動かすための実行マシン等が必要になります。
AWSにおいてもEC2やElasticBeansTalkといったサービスが提供されており、その上にアプリケーションをデプロイする方法もありますが、
サーバや実行環境をほぼ意識することなく、最少限度のプログラムを用いて、アプリケーション実行環境を構築するのがサーバレス開発です。

AWSでは、リクエストを受け付ける(APサーバの役割を代替する)API Gatewayと、アプリケーションの実行ランタイムを提供するAWS Lamdaを用いて、標準的なサーバレス開発が実現できます。

|br|

Spring Cloud Functionを使用したサーバレスアプリケーション
-------------------------------------------------------------------------------------------------------

|br|

エンタープライズアプリケーションでよく使用されるJava言語において、今やデファクトスタンダードと言っても過言ではないSpring frameworkですが、
Framework開発元のサブプロジェクトとして、Spring Cloud Functionがあります。その特徴やメリットとして、

* 関数型のビジネスロジック実行をベースとしたフレームワーク
* AWS Lambdaのみならず、Microsoft AzureやApache OpenWhiskといった他のサーバレス環境でも同じプログラミング実装を可能とする(ポータビリティ性がある)。
* SpringFrameworkの機能をフル活用でき、SpringBootをベースとした簡易なアプリケーション設定を利用して構築できる。
* Spring Cloud AWSやSpring Data DynamoDBなどの他のクラウドベースのフレームワークとの統合が可能。

などが挙げられます。今回はこのフレームワークを用いて、AWS Lambda上で実行されるサーバレスアプリケーションを実装してみましょう。
なお、実際に作成したアプリケーションは `GitHub <https://github.com/debugroom/mynavi-sample-aws-lambda>`_ 上にコミットしています。
以降記載されているソースコードで、インポート文など本質的でない記述を省略している部分がありますので、実行コードを作成する場合は、必要に応じて適宜GitHubソースコードも参照してください。

|br|

まず、Mavenプロジェクトのpom.xmlで、以下の通り、必要なライブラリの依存性を定義します。以下の4つのライブラリが必要です。
アプリケーションを実装するためのspring-cloud-function-web、実行環境としてアダプタとなるspring-cloud-function-adapter-aws、
そして、AWSのSDK(SoftwareDevelopersKit)として提供される、aws-lambda-java-eventsとaws-lambda-java-coreです。

.. sourcecode:: xml

   <dependencies>
       <dependency>
           <groupId>org.springframework.cloud</groupId>
           <artifactId>spring-cloud-function-web</artifactId>
       </dependency>
       <dependency>
           <groupId>org.springframework.cloud</groupId>
           <artifactId>spring-cloud-function-adapter-aws</artifactId>
       </dependency>
       <dependency>
           <groupId>com.amazonaws</groupId>
           <artifactId>aws-lambda-java-events</artifactId>
       </dependency>
       <dependency>
           <groupId>com.amazonaws</groupId>
           <artifactId>aws-lambda-java-core</artifactId>
       </dependency>
   </dependencies>

|br|

これらのライブラリを用いて、サーバレス環境で実行するために必要なアプリケーションを作成します。最低限必要なクラスは、以下３種類です。

#. SpringBootアプリケーションとして実行するメイン実行クラス
#. リクエストを受け取り、実行するためのHandlerクラス
#. Handlerから受け取ったリクエストを元に実行するビジネスロジッククラス

|br|

では、それぞれのクラスの実装について解説していきましょう。

|br|

1. SpringBootアプリケーションとして実行するメイン実行クラス

.. sourcecode:: java

   package org.debugroom.mynavi.sample.aws.lambda;

   import org.springframework.boot.SpringApplication;
   import org.springframework.boot.autoconfigure.SpringBootApplication;

   @SpringBootApplication
   public class App {

       public static void main(String[] args) {
          SpringApplication.run(App.class, args);
       }

   }

まず最初に、アプリケーションをSpring Frameworkで動かすために必要となる起動クラスです。
SpringAppcaliton.run()メソッドを実行するmain()メソッドを持つ実行クラスに、＠SpringBootAppcalitonアノテーションを付与するだけで、
SpringBootによりSpringFramework実行に必要な設定が自動実行されていきます。
DB接続設定の追加など必要に応じてカスタマイズできますが、最低限、このクラスを作成しておくことが必要です。

|br|
|br|

2. リクエストを受け取り、実行するためのHandlerクラス

.. sourcecode:: java

   package org.debugroom.mynavi.sample.aws.lambda;

   import org.springframework.cloud.function.adapter.aws.SpringBootApiGatewayRequestHandler;

   public class ApiGatewayEventHandler extends SpringBootApiGatewayRequestHandler {
   }

続いて必要になるのが、APIからリクエストを受け取り、イベントとしてLambda関数を実行するためのハンドラクラスです。org.springframework.cloud.function.adapter.aws.SpringBootApiGatewayRequestHandlerを継承しているだけで、
上記で作成しているクラスは何も実装していませんが、それはSpringBootApiGatewayRequesthandlerが最低限の機能を実行するのに必要な処理を実装していて、
それを継承することで特に追加実装する必要がないためです。
他にも汎用的なリクエストを受け取るSpringBootRequestHandlerや、Kinesisからのイベントを契機としてリクエストを受け取るためのSpringBootKinesisEventHandlerなどがあります。
今回は、API Gatewayからリクエストを受け取るために必要なSpringBootApiGatewayRequesthandlerを設定しておきましょう。

|br|
|br|

3. Handlerから受け取ったリクエストを元に実行するビジネスロジッククラス

.. sourcecode:: java

   package org.debugroom.mynavi.sample.aws.lambda.functions;

   import java.util.HashMap;
   import java.util.Map;
   import java.util.function.Function;

   import lombok.extern.slf4j.Slf4j;

   import org.springframework.messaging.Message;
   import org.springframework.messaging.MessageHeaders;

   @Slf4j
   public class ApiGatewayEventFunction implements Function<Message<Input>, Message<String>> {

       @Override
       public Message<String> apply(Message<Input> inputMessage) {
           log.info("Message : " + inputMessage.getPayload().getTest());
           return new Message<String>() {
               @Override
               public String getPayload() {
                   return "Complete!";
               }

               @Override
               public MessageHeaders getHeaders() {
                   Map<String, Object> headers = new HashMap<>();
                   return new MessageHeaders(headers);
               }
           };
       }
    }

|br|

最後に、実際にLambda関数として実行するためのビジネスロジッククラスです。java.util.function.Functionを実装して、org.springframework.messaging.Message及び、その型パラメータとして、
受け取りたいJSONリクエストをモデル化したインプットクラスを指定してください。このクラスでは受け取ったJSONリクエストをPOJOクラスへ変換し(ここではInputというModelクラスを作成しています)、
そのクラス中に定義されているString型のtestという変数の値をログ出力しています。処理実行後は単なるString型の文字列"Complete!"をJSONレスポンスとして返すような形で実装するサンプルとしています。

|br|

エンタープライズ案件であれば、こうした単純なロジックだけではなく、インプットデータをDynamoDBなどに追加する処理ロジックなど追加する要件などが出てきますが、
Springベースのアプリケーションで構築していれば、データベースアクセス用のRepositoryクラスをDIインジェクションするだけで簡単にデータ投入のロジックを実装できます。
その他、SQSへキュー処理を移譲したい場合は、Spring Cloud AWSなどのフレームワークを組み合わせれば良いでしょう。
Spring Cloud AWSを用いた詳細な使い方は `Macchinetta <https://macchinetta.github.io/cloud-guideline/current/ja/AWSCollaboration/index.html>`_ を参考にしてください。

また、関数型のクラスは、1で実装したアプリケーションクラス内で直接＠Beanアノテーションで定義しておくか、以下のようにapplication.ymlといった設定ファイルで、
spring.cloud.function.scan.packagesプロパティにコンポーネントスキャンするパッケージを指定しておけば、自動でSpringBootが読み取ってくれます。

|br|

.. sourcecode:: xml

   spring:
     cloud:
       function:
         scan:
           packages: org.debugroom.mynavi.sample.aws.lambda.functions

|br|

さて、最低限の実装が終わったら、いよいよAWS Lambdaへこの実行クラスをJAR形式にしてアップロードしてデプロイします。
JARファイルを作成するには、通常では、Mavenでゴールをpackageに設定し、mvn packageコマンドを実行すれば良いですが、
AWS Lambdaへアップロードするには、これまで作成した３つのクラスに加えて、実行に必要な全ての依存JARライブラリを１つにまとめてアップロードする必要があります。

そのためにpom.xmlに以下の定義も合わせて追加するようにしてください。
maven-shade-pluginの定義等を追加しておけば、Mavenのビルド時に必要なJARファイルを全て含めた形で１つのJAR形式のファイルを作成できます。
mvn packageコマンドを実行すると、プロジェクト名に下記のshareClassifierNameタグに定義した名称が付与された、
全てのJARファイルを包含した大きめのサイズのJARファイルがtargetディレクトリ上に出来上がるはずです。

.. sourcecode:: xml

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-deploy-plugin</artifactId>
                <configuration>
                    <skip>true</skip>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <dependencies>
                    <dependency>
                        <groupId>org.springframework.boot.experimental</groupId>
                        <artifactId>spring-boot-thin-layout</artifactId>
                    </dependency>
                </dependencies>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <configuration>
                    <createDependencyReducedPom>false</createDependencyReducedPom>
                    <shadedArtifactAttached>true</shadedArtifactAttached>
                    <shadedClassifierName>aws</shadedClassifierName>
                </configuration>
            </plugin>
        </plugins>
    </build>


|br|
|br|

AWS Lambdaの設定
-------------------------------------------------------------------------------------------------------

|br|

作成したサーバレスアプリケーションをAWS Lambda Functionとして設定します。AWSコンソールのLambdaサービスを選択し、
「関数の作成」ボタンを押下してください。

|br|

.. figure:: img/aws-lambda-and-api-gateway/management-console-lambda-create-function-1.png
   :scale: 100%

|br|

次に、「１から作成」メニューを選択し、以下の通り設定を行います。

* 名前：任意の名前を設定
* ランタイム：Java8を選択
* ロール：初めてLambdaを使用する場合でロールを初めて作成する場合はカスタムロールの作成を選択してください。

|br|

.. figure:: img/aws-lambda-and-api-gateway/management-console-lambda-create-function-2.png
   :scale: 100%

|br|

カスタムロールの作成を選択すると、IAMロール画面へのウィンドウが開き、Lambda関数の実行やCloudWatchのログ書き込み権限など最低限の権限を設定したIAMロールが自動で表示されます。
「新しいIAMロールの作成」を選んだまま、任意のロール名を入力し、「許可ボタン」を押してください。

|br|

.. figure:: img/aws-lambda-and-api-gateway/management-console-lambda-create-function-3.png
   :scale: 100%

|br|

その後、Lambda作成画面に戻り、「関数の作成」ボタンを押下すると、Lambda関数が作成されます。

|br|

.. figure:: img/aws-lambda-and-api-gateway/management-console-lambda-create-function-4.png
   :scale: 100%

|br|

そのまま下にスクロールすると、関数のコードや実行条件に関する詳細な設定項目が表示されます。関数コード・環境変数の設定で以下の要領に則って作成します。

.. list-table:: 関数のコードと環境変数での入力の要領
   :widths: 3, 7

   * - 入力項目
     - 説明

   * - コードエントリタイプ
     - Lambda関数として実行するアプリケーションをアップロードする方式を決定します。|br| ファイルを直接アップロードするか、S3経由でアップロードするかになりますが、|br| JavaではJAR形式のファイルをアップロードする形になります。
       |br| 10MBをオーバーしていてもアップロードできないことはありませんが、ファイルサイズが大幅に大きくなる場合は |br| S3へアップロードする方法を指定してください。

   * - ランタイム
     - Lambda関数として実行するアプリケーションのランタイムを指定します。 |br| 今回はJava8を指定しておいてください。

   * - ハンドラ
     - 今回作成した3種類のクラスのうち、HandlerクラスをFQCNで設定します。

   * - 関数パッケージ
     - コードエントリタイプでzip、jar形式を指定していた状態で、押下すると |br| アップロードダイアログが表示されます。作成したアプリケーションのJARファイルを指定してください。|br| S3経由のアップロードを選択していた場合は、オブジェクトキーを指定します。

   * - 環境変数
     - Spring Cloud Functionでは、実行する関数を環境変数FUNCTION_NAMEで指定した |br| Bean名で取得します。
       ここでは、今回作成した3種類のクラスのうち、FunctionクラスのBean名を指定します。|br|
       クラス名をキャメルケース(最初の英大文字を小文字にする形で)指定してください。

|br|

最低限設定が必要なものは上記になります、実行条件はデフォルトのままで問題ありませんので、入力後は画面上部にある保存ボタンを押下してください。

.. note:: 重たい処理などはあらかじめ実行メモリを大きめにとっておいた方が良い場合があります。HelloWorldだけのシンプルなアプリケーションでも最低100MBはメモリ消費します。

|br|

.. figure:: img/aws-lambda-and-api-gateway/management-console-lambda-create-function-5.png
   :scale: 100%

|br|

.. warning:: 保村して「テスト」ボタンを実行しても、ここではエラーになります。後述しますが、Lambdaプロキシ統合オプションに基づくリクエストが必要になるので、ここでエラーになっても気にしないでください。

|br|

Amazon API Gatewayの設定
-------------------------------------------------------------------------------------------------------

|br|

続いて、リクエストを受け付けて、設定したAWS Lambdaを実行するためのAPI Gatewayの設定を行います。AWSコンソールの
API Gatewayサービスメニューから、以下を入力して、「APIの作成」を押下してください。

* Protocol : REST
* 新しいAPIの作成：新しいAPI
* API名：任意の名前を入力
* エンドポイントタイプ：リージョン

|br|

.. figure:: img/aws-lambda-and-api-gateway/management-console-api-gateway-create-api-1.png
   :scale: 100%

|br|

API作成後、「アクション」ボタンから「リソースの作成」を選択します。

|br|

.. figure:: img/aws-lambda-and-api-gateway/management-console-api-gateway-create-api-2.png
   :scale: 100%

|br|

リソース名とリソースパスを設定し、「リソースの作成」ボタンを押下します。

|br|

.. figure:: img/aws-lambda-and-api-gateway/management-console-api-gateway-create-api-3.png
   :scale: 100%

|br|

作成したリソースを選択し、再び「アクション」メニューからメソッドの作成を選択し、POSTメソッドを選択、チェックボタンを押下します。

.. warning:: GETメソッドを指定すると、SpringBootApiGatewayRequestHandlerでは、JSONリクエストボディがなくなるため、エラーになります。2019.1月現在、この問題は `ISSUE <https://github.com/spring-cloud/spring-cloud-function/issues/189>`_ として起票されています

|br|

.. figure:: img/aws-lambda-and-api-gateway/management-console-api-gateway-create-api-4.png
   :scale: 100%

|br|

以下の設定内容に従って、POSTメソッドのセットアップを行い、「保存」ボタンを押下します。

* 統合タイプ：Lambda関数
* Lambdaプロキシ統合の使用：チェックする
* Lambdaリージョン：Lambdaを作成したリージョン
* Lamnda関数：前節で作成したLambda関数名を選択する。
* デフォルトタイムアウトの使用：チェックする


|br|

.. warning:: Lambdaプロキシ統合をチェックしないと、Lambdaファンション側にSpring Cloud Functionが期待するデータが来ないのでエラーとなります。忘れないようにチェックしておきましょう。

.. note:: Lambdaプロキシ統合をチェックしないとエラーになる理由は、今回ハンドラクラスで継承したSpringBootApiGatewayRequestHandlerが、型パラメータとしてcom.amazonaws.services.lambda.runtime.events.APIGatewayProxyRequestEventを指定しているためです。Lambdaプロキシ統合でなければ、上記の型パラメータクラスに必要な情報が渡って来ないため、NullPointerExceptionでエラーになります。

|br|

.. figure:: img/aws-lambda-and-api-gateway/management-console-api-gateway-create-api-5.png
   :scale: 100%

|br|

設定したLambda関数が正しく動くか動作テストを実施します。「テスト」リンクを押下します。

|br|

.. figure:: img/aws-lambda-and-api-gateway/management-console-api-gateway-create-api-6.png
   :scale: 100%

|br|

以下の項目を入力し、「テスト」ボタンを押下します。レスポンス本文に「Complete!」と表示されればアップロードしたサーバレスアプリケーションが正常に実行された結果です。

* リクエスト本文：{"test":"TestMessage"}

|br|

.. figure:: img/aws-lambda-and-api-gateway/management-console-api-gateway-create-api-7.png
   :scale: 100%

|br|

Lambda関数が正常に実行できることは確認できましたが、外からAPIをコールするためには、APIをデプロイすることが必要です。「アクション」メニューから「APIのデプロイ」を選択します。

|br|

.. figure:: img/aws-lambda-and-api-gateway/management-console-api-gateway-create-api-8.png
   :scale: 100%

|br|

デプロイされるステージとして、「新しいステージ」を選択し、任意の名前を入力し、「デプロイ」ボタンを押下します。ステージ名はエンドポイントURLにも反映されるため、「prod」や「dev」などにするのが一般的です。

|br|

.. figure:: img/aws-lambda-and-api-gateway/management-console-api-gateway-create-api-9.png
   :scale: 100%

|br|

APIがデプロイされると、エンドポイントのURLが表示されます。このURLにリソース名を加えたものがリソースURLになります。

|br|

.. figure:: img/aws-lambda-and-api-gateway/management-console-api-gateway-create-api-10.png
   :scale: 100%

|br|

実際に外部のクライアントから、以下のようなコマンドで実行結果を確認してみましょう。

.. sourcecode:: bash

   curl -d '{"test":"TestMessage"}' https://xxxxxxxx.execute-api.ap-northeast-1.amazonaws.com/dev/sample

|br|

CloudWatch Logsには、実行したLambda関数の結果が表示されます。

|br|

.. figure:: img/aws-lambda-and-api-gateway/management-console-api-gateway-create-api-11.png
   :scale: 100%

|br|

.. note:: Lambda関数の初回起動時はSpringFrameworkの起動でAPIリクエストを送信してから実行までに少々時間がかかります。２回目以降は実行環境が再利用されるので、即時処理されますが、実行環境一定時間経つと破棄されるので、そうした制約を考慮しておきましょう。

|br|

まとめ
------------------------------------------------------------------

AWS LambdaやAmazon API Gateway、Spring Cloud Functionを利用することにより、ごくごく少量のコーディングで、サーバレスアプリケーションを実行することができます。
このサーバレス開発のスタックはとても簡易で、かつ拡張性も高く、エンタープライズ開発でも十分有用です。

著者紹介
------------------------------------------------------------------

川畑 光平(KAWABATA Kohei)

.. figure:: img/aws-lambda-and-api-gateway/pic_image01.jpg
   :scale: 100%

某システムインテグレータにて、金融機関システム業務アプリケーション開発・システム基盤担当を経て、現在はソフトウェア開発自動化関連の研究開発・推進に従事。

Red Hat Certified Engineer、Pivotal Certified Spring Professional、AWS Certified Solutions Architect Professional等の資格を持ち、アプリケーション基盤・クラウドなど様々な開発プロジェクト支援にも携わる。
