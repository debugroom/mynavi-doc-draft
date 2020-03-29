.. include:: ../module.txt

.. _section-cloud-native-api-gateway-and-lambda-1st-label:

【第1回】サーバレス開発(1)Spring Cloud Functionの実装
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

.. _section-serverless-and-spring-cloud-function-overview-label:

サーバレス開発・Spring Cloud Funtionとは
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

通常、アプリケーション開発において、リクエストを受け付けるためのアプリケーションサーバ・それを動かすための実行マシン等が必要になります。
AWSにおいてもEC2やElasticBeansTalkといったサービスが提供されており、その上にアプリケーションをデプロイする方法もありますが、
サーバや実行環境をほぼ意識することなく、最少限度のプログラムを用いて、アプリケーション実行環境を構築するのがサーバレス開発です。
AWSでは、リクエストを受け付ける(APサーバの役割を代替する)API Gatewayと、アプリケーションの実行ランタイムを提供するAWS Lamdaを用いて、標準的なサーバレス開発が実現できます。

一方、エンタープライズアプリケーションでよく使用されるJava言語において、今やデファクトスタンダードと言っても過言ではないSpring frameworkですが、
Framework開発元のサブプロジェクトとして、Spring Cloud Functionがあります。その特徴やメリットとして、

* 関数型のビジネスロジック実行をベースとしたフレームワーク
* AWS Lambdaのみならず、Microsoft AzureやApache OpenWhiskといった他のサーバレス環境でも同じプログラミング実装を可能とする(ポータビリティ性がある)。
* SpringFrameworkの機能をフル活用でき、SpringBootをベースとした簡易なアプリケーション設定を利用して、最小限のコーディングで構築できる。
* Spring Cloud AWSやSpring Data DynamoDBなどの他のクラウドベースのフレームワークとの統合が可能。

などが挙げられます。今回はこのフレームワークを用いて、AWS Lambda上で実行されるサーバレスアプリケーションを実装してみましょう。
なお、実際に作成したアプリケーションは `GitHub <https://github.com/debugroom/mynavi-sample-aws-lambda>`_ 上にコミットしています。
以降記載されているソースコードで、インポート文など本質的でない記述を省略している部分がありますので、実行コードを作成する場合は、必要に応じて適宜GitHubソースコードも参照してください。

第一回は以降、

#. Spring Cloud Functionを使用したサーバレスアプリケーションの実装方法
#. AWS Lambdaの設定
#. Amazon API Gatewayの設定

という、３つのステップに分け、解説を進めていきます。

|br|

.. _section-severless-implementation-using-spring-cloud-function-label:

(1) Spring Cloud Functionを使用したサーバレスアプリケーション実装
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

Spring Cloud Functionを使用するには、まず、Mavenプロジェクトのpom.xmlで、以下の通り、必要なライブラリの依存性を定義します。以下の4つのライブラリが必要です。
アプリケーションを実装するためのspring-cloud-function-web、実行環境としてアダプタとなるspring-cloud-function-adapter-aws、
そして、AWSのSDK(SoftwareDevelopersKit)として提供される、aws-lambda-java-eventsとaws-lambda-java-coreです。

|br|

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

最後に、実際にLambda関数として実行するためのビジネスロジッククラスです。java.util.function.Functionを実装して、
org.springframework.messaging.Message及び、その型パラメータとして、受け取りたいJSONリクエストをモデル化したインプットクラスを指定してください。
この形式で実装しておくと、Spring Cloud FunctionがAPI Gatewayから受け取ったJSONリクエストをPOJOクラスへ変換し、
オーバーライドしたapply()メソッドを実行します。ここでは「Input」というModelクラスを作成しており、
メソッドの引数としてそのオブジェクトを受け取り、そのオブジェクトのパラメータであるString型の「test」という変数の値を
ログ出力しています。処理実行後は単なるString型の文字列「”Complete!”」をJSONレスポンスとして返すような形で実装するサンプルです。

|br|

実案件のアプリケーション開発では、こうした単純なロジックだけではなく、インプットデータをDynamoDBなどに追加する処理ロジックなど追加する要件などが出てきますが、
Springベースのアプリケーションで構築していれば、データベースアクセス用のRepositoryクラスをDIインジェクションするだけで簡単にデータ投入のロジックを実装できます。
その他、SQSへキュー処理を移譲したい場合は、Spring Cloud AWSなどのフレームワークを組み合わせれば良いでしょう。

.. note:: Spring Cloud AWSを用いた詳細な使い方は、NTTが公開している `Macchinetta <https://macchinetta.github.io/cloud-guideline/current/ja/AWSCollaboration/index.html>`_ などを参考にしてください。

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

JARファイル形式で、アプリケーションが作成できれば完成です。次回は作成したアプリケーションをAWS Lambdaへ設定して見ます。

|br|

著者紹介
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

川畑 光平(KAWABATA Kohei)

.. figure:: img/aws-lambda-and-api-gateway/pic_image01.jpg


某システムインテグレータにて、金融機関システム業務アプリケーション開発・システム基盤担当を経て、現在はソフトウェア開発自動化関連の研究開発・推進に従事。

Red Hat Certified Engineer、Pivotal Certified Spring Professional、AWS Certified Solutions Architect Professional等の資格を持ち、アプリケーション基盤・クラウドなど様々な開発プロジェクト支援にも携わる。
