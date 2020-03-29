.. include:: ../module.txt

【第5回】マイクロサービスの単体試験コード実装(前編)
------------------------------------------------------------------

.. _section-application-test-for-microservice-label:

マイクロサービスにおける単体テスト
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

前回は、マイクロサービス(Backend)やそれを呼び出すWebアプリケーション(BackendForFrontend:BFF)のパッケージ・コンポーネント構成を示し、テスト観点を例示しました。今回からはテストを実装する際のポイントやテスト戦略を説明していきます。

まずバックエンドで実行されるマイクロサービスの単体テストです。アプリケーションおよびテストのパッケージ・コンポーネント構成は以下としています。

|br|

.. sourcecode:: bash

   [backend]
     └src
       ├main
       │ ├java
       │ │ └org
       │ │   └debugroom
       │ │     └mynavi
       │ │       └sample
       │ │         └continuous
       │ │           └integration
       │ │             └backend
       │ │               └app                                  ... アプリケーション層のパッケージ
       │ │               │ └model                              ... リクエストパラメータのモデルクラスパッケージ
       │ │               │ │ ├Xxxxx.java                       ... 入力チェックルール等が定義されるモデルクラス
       │ │               │ │ └XxxxxMapper.java                 ... ドメイン層のモデルクラスと相互変換するマッパークラス
       │ │               │ └web                                ... MvcConfigでコンポーネントスキャンの対象とするパッケージ
       │ │               │   └BackendRestController.java       ... リクエストハンドリング・ドメインサービス呼び出して、Resourceを返却するコントローラクラス
       │ │               └domain                               ... ドメイン層のパッケージ
       │ │               │ └model
       │ │               │ │ └entity                           ... JPAConfigでスキャン対象とするエンティティクラスパッケージ
       │ │               │ │   └Xxxxx.java                     ... JPAエンティティクラス
       │ │               │ ├repository                         ... JPAConfigでスキャン対象とするレポジトリクラスパッケージ
       │ │               │ │ ├specification                    ... JPAでテーブル結合等の条件を指定するクラスパッケージ
       │ │               │ │ │ └Xxxxx.java                     ... JPAでテーブル結合等の条件を指定するクラス
       │ │               │ │ └XxxxxRepository.java             ... レポジトリインタフェースクラス
       │ │               │ └service                            ... DomainConfigでコンポーネントスキャンの対象とするサービスクラスパッケージ
       │ │               │   ├SampleService.java               ... DBへ基本的なCRUDアクセスを行うサービスインタフェースクラス
       │ │               │   ├SampleServiceImpl.java           ... SampleServiceの実装クラス
       │ │               │   ├SampleOneToOneService.java       ... 1対1の関連をもつテーブルアクセスを行うサービスインタフェースクラス
       │ │               │   ├SampleOneToOneServiceImpl.java   ... SampleOneToOneServiceの実装クラス
       │ │               │   ├SampleOneToManyService.java      ... 1対多の関連をもつテーブルアクセスを行うサービスインタフェースクラス
       │ │               │   ├SampleOneToManyServiceImpl.java  ... SampleOneToOneServiceの実装クラス
       │ │               │   ├SampleManyToManyService.java     ... 多対多の関連をもつテーブルアクセスを行うサービスインタフェースクラス
       │ │               │   └SampleManyToManyServiceImpl.java ... サービス実装クラス
       │ │               └config                               ... 設定クラス用のパッケージ
       │ │                   ├App.java                         ... アプリケーション起動クラス
       │ │                   ├DevConfig.java                   ... 開発環境固有の設定クラス
       │ │                   ├DomainConfig.java                ... ドメイン層に関する設定クラス
       │ │                   ├JPAConfig.java                   ... JPA設定クラス
       │ │                   └MvcConfig.java                   ... アプリケーション層に関する設定クラス
       │ └resources
       │   ├application.yml                                    ... アプリケーション設定ファイル
       │   └application-dev.yml                                ... プロファイル"dev"で有効になるアプリケーション設定ファイル
       test                                                    ... テストパッケージフォルダ
         ├java
         │ └org
         │   └debugroom
         │     └mynavi
         │       └sample
         │         └continuous
         │           └integration
         │             └backend
         │               ├app
         │               │ └web
         │               │   └BackendRestControllerTest.java   ... Controllerのテストクラス
         │               ├domain
         │               │ ├DataJpaTestConfig.java             ... Repositoryのテスト設定クラス
         │               │ ├repository                         ... Repositoryテストパッケージ
         │               │ │ └XxxxRepositoryTest.java          ... Repositoryのテストクラス
         │               │ └service                            ... Serviceテストパッケージ
         │               │   └XxxxServiceTest.java             ... Serviceのテストクラス
         │               └config
         │                 └TestConfig.java                    ... Testの汎用設定クラス
         └resources
           └application.yml                                    ... テスト用のアプリケーション設定ファイル

|br|

一般にソフトウェアの単体テストでは、

* 開発組織やプロジェクト
* プログラミング言語
* テストのスコープとする処理・機能、プログラム単位
* Webアプリケーションやモバイルなどアプリケーション特性

などの違いにより、その定義・検証観点は異なります。Java、Springにおける単体テストでは、テスト対象のクラスが依存するコンポーネントをモックやスタブで置き換えてテストを行えるよう、様々な機能が提供されています。
SpringBootを使ってアプリケーションを実装する場合は、主にController、Service、Repositoryという単位で単体コンポーネントと考え、
以下のイメージの通り、モックやスタブの設定を行い、表に示すような観点でテストすることを推奨します。

.. _命名規約によるSQLクエリの自動組立: http://terasolunaorg.github.io/guideline/5.5.1.RELEASE/ja/ArchitectureInDetail/DataAccessDetail/DataAccessJpa.html#how-to-specify-query-mathodname-label

|br|

.. figure:: img/automation_infra_devops_test/microservice-unittest-scope.png


|br|

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

|br|

また、以降、SpringBootを使ってテストコード実装を進めていきますが、プロジェクトのpom.xmlにspring-boot-starter-testのライブラリを含めておいてください。

|br|

.. sourcecode:: xml

   <dependencies>
     <dependency>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter-test</artifactId>
       <scope>test</scope>
     </dependency>
   </dependencies>

|br|

.. _section-repository-test-for-microservice-label:

Repositoryの単体テスト実装
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

RepositoryはEricEvansのドメイン駆動設計で有名となった、データを永続化するコンポーネントの表現です。
J2EEパターンで言うところの「DataAccessObject(DAO)」に相当しますが、主な違いとしてはRepositoryはよりビジネスドメインのルール・制約を含んでおり、
より低いレベルのAPIをもつデータベースアクセスコンポーネントにない、よりビジネス的な意味合いを含むコンポーネントです。
save()メソッドがRepositoryの持つAPIであり、Insert()メソッドがDAOが持つAPIと考えるとわかりやすいでしょう。
AddressやEmailをもつUserを永続化する場合、AddressやEmailを設定したUserに対し、repository#save(User)を一度呼べば済むのか、
emailDao#insert(email)、addressDao#insert(address)、userDao#insert(User)の３セットコールするのか、実装ではより顕著な差が現れます。
また、永続化する先のデータストアがファイルなのか、RDBなのか、NoSQLデータベースなのか、他システムのデータストアかもRepositoryは問いません。
(ファイルを永続化する場合にinsertは表現として少々不適当)。つまり、Repositoryはデータ永続化のためのより抽象的なコンポーネントとして表現されます。

このマイクロサービスでは永続化先をRDBに設定しているため、上記の表に記載したテスト観点を設定していますが、Repositoryクラスの永続化先や実装ライブラリに応じて、
適切な観点でテストを実施するようにしてください。以降、RDBとSpringDataJPAを用いたSpringBoot実装のテストについて述べます。

|br|

Springではテスト実行環境を自動構築するいくつかのアノテーションを提供しています。SpringBootを使用したアプリケーションのテスト向けに提供されている@SpringBootTestアノテーションを使って
Repositoryのテストを実行することも可能ですが、テストで使用しないコンポーネントを含めてDIコンテナを構築するなど起動時間のオーバーヘッドがあるため、
実行速度の観点からJPAのRepositoryの単体テストでは、@DataJpaTestを使用することを推奨します。

|br|

.. note:: @DataJpaTest以外にも@JdbcTestや@DataRedisTest、@DataMongoTest、@MyBaitsTestなど、データストアやORマッパーライブラリのに応じて同様の機能をもつアノテーションがサードパーティ含め提供されています。

|br|

＠DataJpaTestの使用方法としては、JUnitテストランナーとしてorg.springframework.test.context.junit4.SpringRunnerを指定したテストクラスに、
org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTestアノテーションを付与します。この設定により、テスト実行環境として、
pom.xmlで依存性を定義したH2やHSQLなどインメモリDBが構築され、テスト環境向けのEntityMangerであるTestEntityManagerがテスト用DIコンテナに追加されるようになります。
テストコード上では、@AutowiredでTestEntityManagerを取得し、@Beforeを付与したテストのセットアップメソッドを用意して、前もって準備しておきたいテストデータをtestEntityManager#persist()で
インメモリDBへ事前保存し、@Testメソッドでテスト検証コードを記載する形で利用します。

|br|

.. sourcecode:: xml
   :caption: pom.xmlの依存性定義

   <dependency>
     <groupId>org.hsqldb</groupId>
     <artifactId>hsqldb</artifactId>
     <scope>runtime</scope>
   </dependency>

|br|

.. sourcecode:: java

   package org.debugroom.mynavi.sample.continuous.integration.backend.domain.repository;

   // omit

   import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
   import org.springframework.boot.test.autoconfigure.orm.jpa.TestEntityManager;
   import org.springframework.test.context.junit4.SpringRunner;

   // omit

   @RunWith(SpringRunner.class)
   @DataJpaTest
   public class UserRepositoryTest {

       @Autowired
       TestEntityManager testEntityManager;

       @Autowired
       UserRepository userRepository;

       @Before
       public void before(){
           // omit
           testEntityManager.persist(
                   User.builder()
                           .userId(userIdA)
                           .firstName("taro")
                           .familyName("mynavi")
                           .loginId("taro.mynavi")
                           .addressByUserId(address1)
                           .membershipsByUserId(
                                   Arrays.asList(new Membership[]{
                                           membership1, membership3}))
                           .ver(0)
                           .lastUpdatedAt(DateUtil.now())
                   .build());
      }

      // omit

      @Test
      public void testFindByLoginIdNormalCase(){
          Optional<User> optionalUser = userRepository.findByLoginId("taro.mynavi");
          User user = optionalUser.get();
          assertThat(user.getUserId(), is(0L));
          assertThat(user.getFirstName(), is("taro"));
      }

      // omit
   }

|br|

.. warning:: @DataJpaTestアノテーションに限らずですが、@SpringBootTestをはじめ、SpringBootで提供されているテスト用のアノテーションはテストクラスと同じ、もしくはその上位にあるパッケージに@SpringBootApplicaitonが付与された起動クラスが必要になります。
   SpringBoot起動クラスがsrc/main上で、テストクラスと同一もしくはその上位にあるパッケージにあれば問題ありませんが、本アプリケーションのように
   テストクラスのパッケージの上位ルート上でもないconfigパッケージにある場合は、テスト用のパッケージに@SpringBootApplicaitonが付与されたクラスを作成しておきましょう。

|br|

Repositoryで定義したインタフェースのメソッドに対するテストを実装し、期待結果を検証することで、
テーブル定義とエンティティクラスの整合性や、エンティティクラスへのデータマッピング、SQLクエリの実行可否など、
Repositoryやエンティティクラスの定義、SQL定義の実装の妥当性を検証可能です。テーブルの結合によるデータ取得やRDBの集計関数を使ったデータアクセスなども
合わせて検証が可能なので、データ取得に関するエラーはこの単体テストで検出できるようにしておきましょう。
ただし、データベースの更新については、データベースの反映結果を取得して個別にアサーションを記載するとアサーションコード量が膨大になり大変です。
次回以降で解説する結合試験でDBUnitを用いて、テーブルデータをまとめて検証した方が簡易なため、ここでは検証対象には含めないでおきます。
今回サンプルでは以下のようなユースケース・検証観点でテストコードを実装しています。

|br|

.. _UserRepository#findByLoginId: https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/main/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/repository/UserRepository.java#L14
.. _User(Entity): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/main/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/model/entity/User.java
.. _UserRepositoryTest#testFindByLoginIdNormalCase(): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/test/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/repository/UserRepositoryTest.java#L124
.. _UserRepositoryTest#testFindByLoginIdAbnormalCase(): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/test/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/repository/UserRepositoryTest.java#L132
.. _UserRepository#existsByLoginId: https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/main/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/repository/UserRepository.java#L15
.. _UserRepositoryTest#testExistsByLoginIdNormalCase(): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/test/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/repository/UserRepositoryTest.java#L138
.. _UserRepositoryTest#testExistsByLoginIdAbnormalCase(): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/test/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/repository/UserRepositoryTest.java#L143
.. _FindUsersHavingAddressOfZipCode: https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/main/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/repository/specification/FindUsersHavingAddressOfZipCode.java
.. _UserRepositoryTest#testFindUsersHavingAddressOfZipCodeNormalCase(): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/test/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/repository/UserRepositoryTest.java#L148
.. _FindUsersNotHavingAddressOfZipCode: https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/main/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/repository/specification/FindUsersNotHavingAddressOfZipCode.java
.. _UserRepositoryTest#testFindUsersNotHavingAddressOfZipCodeNormalCase(): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/test/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/repository/UserRepositoryTest.java#L156
.. _FindUsersByGroup: https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/main/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/repository/specification/FindUsersByGroup.java
.. _UserRepositoryTest#testFindUsersByGroupNormalCase1(): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/test/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/repository/UserRepositoryTest.java#L164
.. _UserRepositoryTest#testFindUsersByGroupNormalCase2(): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/test/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/repository/UserRepositoryTest.java#L176
.. _FindUsersByNotGroup: https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/main/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/repository/specification/FindUsersByNotGroup.java
.. _UserRepositoryTest#testFindUsersByNotGroupNormalCase1(): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/test/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/repository/UserRepositoryTest.java#L189
.. _UserRepositoryTest#testFindUsersByNotGroupNormalCase2(): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/test/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/repository/UserRepositoryTest.java#L202
.. _UserRepository#getMaxUserId: https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/main/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/repository/UserRepository.java#L18
.. _UserRepositoryTest#testGetMaxUserIdNormalCase(): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/test/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/repository/UserRepositoryTest.java#L215
.. _Email(Entity): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/main/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/model/entity/Email.java
.. _EmailRepository#findByEmail: https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/main/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/repository/EmailRepository.java#L10
.. _EmailRepositoryTest#testFindByEmailNormalCase(): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/test/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/repository/EmailRepositoryTest.java#L56
.. _Group(Entity): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/main/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/model/entity/Group.java
.. _GroupRepository#findByGroupName: https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/main/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/repository/GroupRepository.java#L11
.. _GroupRepositoryTest#testFindByGroupNameNormalCase(): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/test/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/repository/GroupRepositoryTest.java#L92
.. _FindGroupsByUserId: https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/main/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/repository/specification/FindGroupsByUserId.java
.. _GroupRepositoryTest#testFindGroupsByUserIdNormalCase(): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/test/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/repository/GroupRepositoryTest.java#L98

.. list-table::
   :widths: 7, 6, 7

   * - ユースケース
     - 主な処理実装クラス・メソッド |br| テストメソッド
     - 検証観点

   * - [正常系]ログインIDを元にユーザを検索する
     - `UserRepository#findByLoginId`_ |br| `User(Entity)`_ |br| |br| `UserRepositoryTest#testFindByLoginIdNormalCase()`_
     -  ・エンティティクラスがテーブル定義と一致しているか |br| ・O/Rマッピング設定が妥当か |br| ・ `命名規約によるSQLクエリの自動組立`_ が正しく実行されるか

   * - [異常系]ログインIDを元にユーザを検索する
     - `UserRepository#findByLoginId`_ |br| `UserRepositoryTest#testFindByLoginIdAbnormalCase()`_
     - ・該当しないデータが発生した場合に期待された戻り値が返されるか

   * - [正常系]ログインIDを元にユーザが存在するか確認する
     - `UserRepository#existsByLoginId`_ |br| `UserRepositoryTest#testExistsByLoginIdNormalCase()`_
     - ・ `命名規約によるSQLクエリの自動組立`_ が正しく実行されるか

   * - [異常系]ログインIDを元にユーザが存在するか確認する
     - `UserRepository#existsByLoginId`_ |br| `UserRepositoryTest#testExistsByLoginIdAbnormalCase()`_
     - ・該当しないデータが発生した場合に期待された戻り値が返されるか

   * - [正常系]指定された郵便番号の住所をもつユーザを検索する
     - `FindUsersHavingAddressOfZipCode`_ |br| `UserRepositoryTest#testFindUsersHavingAddressOfZipCodeNormalCase()`_
     - ・テーブルの結合条件が妥当か

   * - [正常系]指定された郵便番号の住所をもたないユーザを検索する
     - `FindUsersNotHavingAddressOfZipCode`_ |br| `UserRepositoryTest#testFindUsersNotHavingAddressOfZipCodeNormalCase()`_
     - ・テーブルの結合条件が妥当か

   * - [正常系]指定されたグループ名のグループに所属するユーザを検索する
     - `FindUsersByGroup`_ |br| `UserRepositoryTest#testFindUsersByGroupNormalCase1()`_
     - ・テーブルの結合条件が妥当か

   * - [正常系]指定されたグループIDのグループに所属するユーザを検索する
     - `FindUsersByGroup`_ |br| `UserRepositoryTest#testFindUsersByGroupNormalCase2()`_
     - ・テーブルの結合条件が妥当か

   * - [正常系]指定されたグループ名のグループに所属しないユーザを検索する
     - `FindUsersByNotGroup`_ |br| `UserRepositoryTest#testFindUsersByNotGroupNormalCase1()`_
     - ・テーブルの結合条件が妥当か

   * - [正常系]指定されたグループIDのグループに所属しないユーザを検索する
     - `FindUsersByNotGroup`_ |br| `UserRepositoryTest#testFindUsersByNotGroupNormalCase2()`_
     - ・テーブルの結合条件が妥当か

   * - [正常系]userIdでMaxの値を持つIDを検索する
     - `UserRepository#getMaxUserId`_ |br| `UserRepositoryTest#testGetMaxUserIdNormalCase()`_
     - ・記載したSQLクエリや集合関数が正しく実行されるか

   * - [正常系]指定されたメールアドレスをもつメールを検索する。
     - `EmailRepository#findByEmail`_ |br| `Email(Entity)`_ |br| |br| `EmailRepositoryTest#testFindByEmailNormalCase()`_
     - ・エンティティクラスがテーブル定義と一致しているか |br| ・O/Rマッピング設定が妥当か |br| ・ `命名規約によるSQLクエリの自動組立`_ が正しく実行されるか

   * - [正常系]指定されたグループ名をもつグループを検索する。
     - `GroupRepository#findByGroupName`_ |br| `Group(Entity)`_ |br| |br| `GroupRepositoryTest#testFindByGroupNameNormalCase()`_
     - ・エンティティクラスがテーブル定義と一致しているか |br| ・O/Rマッピング設定が妥当か |br| ・ `命名規約によるSQLクエリの自動組立`_ が正しく実行されるか |br|

   * - [正常系]指定されたユーザIDをもつユーザが所属するグループを検索する。
     - `FindGroupsByUserId`_ |br| `GroupRepositoryTest#testFindGroupsByUserIdNormalCase()`_
     - ・テーブルの結合条件が妥当か

|br|

.. note:: 結合条件を指定したSpecificationクラスに実装している `JPAのメタモデルクラスはIDEのジェネレータ機能を使用して自動生成 <https://docs.jboss.org/hibernate/orm/5.0/topical/html/metamodelgen/MetamodelGenerator.html>`_ しています。
          本アプリケーションでは、`IntelliJの公式ページ <https://intellij-support.jetbrains.com/hc/en-us/community/posts/206842215-Hibernate-JPA-2-Metamodel-generation>`_ や
          `Hibernateが提供するGenerator <https://docs.jboss.org/hibernate/orm/5.0/topical/html/metamodelgen/MetamodelGenerator.html#chapter-usage>`_ の手順にならって設定していますが、
          IntelliJでは、メタモデルクラスのデフォルトの出力先がtargetフォルダになっているため、GitHubソースコード上には掲載されてないのでご注意ください(IntelliJでもEclipseでもメタモデルクラスの出力機能があるので、その設定を行えば出力されるようになります)。

|br|

次回は引き続き、Service、Controllerのテストコードを実装し、解説していきます。

著者紹介
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

川畑 光平(KAWABATA Kohei) - NTTデータ 課長代理

.. figure:: img/automation_infra_devops_overview/pic_image01.jpg


金融機関システム業務アプリケーション開発・システム基盤担当を経て、現在はソフトウェア開発自動化関連の研究開発・推進に従事。

Red Hat Certified Engineer、Pivotal Certified Spring Professional、AWS Certified Solutions Architect Professional等の資格を持ち、アプリケーション基盤・クラウドなど様々な開発プロジェクト支援にも携わる。

`2019 APN AWS Top Engineers & Ambassadors <https://aws.amazon.com/jp/blogs/psa/japan-apn-ambassador-2019/>`_ 選出。
