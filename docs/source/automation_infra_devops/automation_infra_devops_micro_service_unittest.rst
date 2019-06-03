.. include:: ../module.txt

.. _section-automation-infra-devops-microservice-unittest-label:

基盤・デプロイ自動化実践
==================================================================

マイクロサービスアーキテクチャの基盤・デプロイ自動化
------------------------------------------------------------------

|br|

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
       │ │               │ │ └XxxxxRepository.java             ... レポジトリインターフェースクラス
       │ │               │ └service                            ... DomainConfigでコンポーネントスキャンの対象とするサービスクラスパッケージ
       │ │               │   ├SampleService.java               ... DBへ基本的なCRUDアクセスを行うサービスインターフェースクラス
       │ │               │   ├SampleServiceImpl.java           ... SampleServiceの実装クラス
       │ │               │   ├SampleOneToOneService.java       ... 1対1の関連をもつテーブルアクセスを行うサービスインターフェースクラス
       │ │               │   ├SampleOneToOneServiceImpl.java   ... SampleOneToOneServiceの実装クラス
       │ │               │   ├SampleOneToManyService.java      ... 1対多の関連をもつテーブルアクセスを行うサービスインターフェースクラス
       │ │               │   ├SampleOneToManyServiceImpl.java  ... SampleOneToOneServiceの実装クラス
       │ │               │   ├SampleManyToManyService.java     ... 多対多の関連をもつテーブルアクセスを行うサービスインターフェースクラス
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
   :scale: 100%

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

   import static org.hamcrest.CoreMatchers.*;
   import static org.junit.Assert.*;

   import org.junit.Before;
   import org.junit.Test;
   import org.junit.runner.RunWith;
   import org.springframework.beans.factory.annotation.Autowired;
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

Repositoryで定義したインターフェースのメソッドに対するテストを実装し、期待結果を検証することで、
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

.. _section-serivce-test-for-microservice-label:

Serviceの単体テスト実装
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

Serviceは `TERASOLUNAのガイドライン Serviceの実装 <http://terasolunaorg.github.io/guideline/5.5.1.RELEASE/ja/ImplementationAtEachLayer/DomainLayer.html#service>`_ でも述べている通り、
ビジネス処理のトランザクション境界であり、ロジックの中核となるコンポーネントです。Service内でデータベースにアクセスする場合は、
Serviceの実装クラスに@AutowiredでインジェクションしたRepositoryを介して行いますが、単体テストを行う場合は、
モックなどでデータアクセス部分をスタブ化してからテストコードを実装します。Serviceは可能な限りPOJOで実装されるのが望ましいですが、
ビジネスロジック中に頻出するビジネス例外発生時のメッセージなどはSpringが提供するMessageSourceを使って取得するのが一般的であり、
こうした部分までをスタブ化するとセットアップが大変なので、実際の処理と同様、SpringのDIコンテナから取得できるようにしておくほうが望ましいです。

そのため、ServiceのテストではSpringBootアプリケーション起動時と同様に、DIコンテナとともに実行に必要なコンポーネントをオートコンフィグレーションする@SpringBootTestアノテーションを用いて、
MessageSourceなどのコンポーネントはSpringのDIコンテナから取得できるようにしておき、Repositoryなど手動実装がメインの部分はモック化します。
また、@SpringBootTestアノテーションのclasses属性に指定したテストクラスは、そのテストクラスと同一パッケージにある
@Configurationアノテーションが付与された設定クラスを読み込むので、src/test配下に、src/main/配下と同じconfigパッケージを作成し、
そこに配置したテスト用のConfigクラスを設定することで、アプリケーション起動時と同様、src/main/配下のconfigパッケージ配下にある設定クラスを読み込むようになります。
コードの例は以下のようになります。

|br|

.. sourcecode:: java

   import static org.hamcrest.core.Is.is;
   import static org.hamcrest.core.IsNull.nullValue;
   import static org.mockito.Mockito.when;
   import static org.junit.Assert.*;

   // omit

   import org.springframework.beans.factory.annotation.Autowired;
   import org.springframework.boot.test.context.SpringBootTest;
   import org.springframework.boot.test.mock.mockito.MockBean;
   import org.springframework.context.annotation.Bean;
   import org.springframework.context.annotation.Configuration;

   // omit

   import org.debugroom.mynavi.sample.continuous.integration.backend.config.TestConfig;

   // omit

   @RunWith(Enclosed.class)
   public class SampleServiceImplTest {

       @RunWith(SpringRunner.class)
       @SpringBootTest(classes = {
            TestConfig.UnitTestConfig.class,
            SampleServiceImplTest.UnitTest.Config.class,
       }, webEnvironment = SpringBootTest.WebEnvironment.NONE)
       public static class UnitTest{

         @Configuration
         public static class Config{
            @Bean
            SampleService sampleService(){
                return new SampleServiceImpl();
            }
         }

         @MockBean
         UserRepository userRepositoryMock;

         // omit

         @Autowired
         SampleService sampleService;

         @Rule
         public ExpectedException expectedException = ExpectedException.none();

         @Before
         public void setUp(){
            //omit
            when(userRepositoryMock.findById(new Long(1))).thenReturn(Optional.empty());
            //omit
         }

         // omit

         @Test
         public void findOneAbnormalTest() throws BusinessException{
             expectedException.expect(BusinessException.class);
             expectedException.expectMessage("指定されたユーザは存在しないか、IDが誤っています。 UserID : 1");
             User user = sampleService.findOne(
               User.builder().userId(new Long(1)).build());
         }

         // omit
   }

|br|

.. _SampleService#findOne(User user): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/main/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/service/SampleServiceImpl.java#L35
.. _SampleServiceImplTest#findOneAbnormalTest(): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/test/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/service/SampleServiceImplTest.java#L177
.. _SampleService#add(User user): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/main/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/service/SampleServiceImpl.java#L52
.. _SampleServiceImplTest#addTestNormalCase(): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/test/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/service/SampleServiceImplTest.java#L185
.. _SampleServiceImplTest#addTestAbnormalCase(): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/test/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/service/SampleServiceImplTest.java#L215
.. _SampleService#update(User user): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/main/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/service/SampleServiceImpl.java#L98
.. _SampleServiceImplTest#updateTestNormalCase(): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/test/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/service/SampleServiceImplTest.java#L226
.. _SampleServiceImplTest#updateTestAbnormalCase1(): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/test/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/service/SampleServiceImplTest.java#L261
.. _SampleServiceImplTest#updateTestAbnormalCase2(): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/test/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/service/SampleServiceImplTest.java#L272
.. _SampleService#delete(User user): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/main/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/service/SampleServiceImpl.java#L145
.. _SampleServiceImplTest#deleteTestAbnormalCase(): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/test/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/service/SampleServiceImplTest.java#L288
.. _SampleService#findUserHave(String loginId): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/main/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/service/SampleServiceImpl.java#L159
.. _SampleServiceImplTest#findUserHaveLoginAbnormalTest(): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/test/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/service/SampleServiceImplTest.java#L298
.. _SampleOneToOneService#findAddressOf(User user): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/main/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/service/SampleOneToOneServiceImpl.java#L33
.. _SampleOneToOneServiceImplTest#findAddressOfAbnormalTest(): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/test/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/service/SampleOneToOneServiceImplTest.java#L109
.. _SampleOneToOneService#update(Address address): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/main/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/service/SampleOneToOneServiceImpl.java#L57
.. _SampleOneToOneServiceImplTest#updateNormalTest(): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/test/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/service/SampleOneToOneServiceImplTest.java#L117
.. _SampleOneToOneServiceImplTest#updateAbnormalTest(): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/test/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/service/SampleOneToOneServiceImplTest.java#L129
.. _SampleOneToManyService#getEmailsOf(User user): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/main/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/service/SampleOneToManyServiceImpl.java#L38
.. _SampleOneToManyServiceImplTest#getEmailsOfAbnormalTest(): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/test/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/service/SampleOneToManyServiceImplTest.java#L118
.. _SampleOneToManyService#add(Email email): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/main/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/service/SampleOneToManyServiceImpl.java#L55
.. _SampleOneToManyServiceImplTest#addAbnormalTest(): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/test/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/service/SampleOneToManyServiceImplTest.java#L126
.. _SampleOneToManyService#update(Email email): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/main/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/service/SampleOneToManyServiceImpl.java#L74
.. _SampleOneToManyServiceImplTest#updateAbnormalTest(): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/test/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/service/SampleOneToManyServiceImplTest.java#L134
.. _SampleOneToManyService#delete(Email email): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/main/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/service/SampleOneToManyServiceImpl.java#L92
.. _SampleOneToManyServiceImplTest#deleteAbnormalTest1(): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/test/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/service/SampleOneToManyServiceImplTest.java#L142
.. _SampleOneToManyServiceImplTest#deleteAbnormalTest2(): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/test/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/service/SampleOneToManyServiceImplTest.java#L150
.. _SampleOneToManyServiceImplTest#deleteAbnormalTest3(): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/test/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/service/SampleOneToManyServiceImplTest.java#L158
.. _SampleOneToManyService#deleteAllEmail(User user): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/main/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/service/SampleOneToManyServiceImpl.java#L122
.. _SampleOneToManyServiceImplTest#deleteAllEmailAbnormalTest(): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/test/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/service/SampleOneToManyServiceImplTest.java#L166
.. _SampleManyToManyService#addUserTo(Group group, User addUser): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/main/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/service/SampleManyToManyServiceImpl.java#L58
.. _SampleManyToManyServiceImplTest#addUserToGroupAbnormalTest1(): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/test/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/service/SampleManyToManyServiceImplTest.java#L105
.. _SampleManyToManyServiceImplTest#addUserToGroupAbnormalTest2(): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/test/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/service/SampleManyToManyServiceImplTest.java#L114
.. _SampleManyToManyService#deleteUserFrom(Group group, User deleteUser): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/main/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/service/SampleManyToManyServiceImpl.java#L86
.. _SampleManyToManyServiceImplTest#deleteUserFromGroupAbnormalTest1(): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/test/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/service/SampleManyToManyServiceImplTest.java#L123
.. _SampleManyToManyServiceImplTest#deleteUserFromGroupAbnormalTest2(): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/test/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/service/SampleManyToManyServiceImplTest.java#L132
.. _SampleManyToManyService#delete(Group group): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/test/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/service/SampleManyToManyServiceImplTest.java#L141
.. _SampleManyToManyServiceImplTest#deleteGroupAbnormalTest(): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/test/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/service/SampleManyToManyServiceImplTest.java#L141
.. _SampleManyToManyService#delete(User user): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/main/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/service/SampleManyToManyServiceImpl.java#L123
.. _SampleManyToManyServiceImplTest#deleteUserAbnormalTest(): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/test/java/org/debugroom/mynavi/sample/continuous/integration/backend/domain/service/SampleManyToManyServiceImplTest.java#L149


こうしたテスト実装により、Sevice実行時のアウトプットオブジェクトの妥当性やビジネス例外発生の妥当性、ビジネス例外のメッセージなどを
検証できます。単純にRepositoryを呼び出してデータを返すだけのメソッドは、試験内容が重複するので、結合試験で確認するものとし、
サンプルで作成したテストケースは、Serviceの異常系処理を中心に、以下のようなユースケース・検証観点をもとに実装しています。

|br|

.. list-table::
   :widths: 7, 6, 7

   * - ユースケース
     - 主な処理実装クラス・メソッド |br| テストメソッド
     - 検証観点

   * - [異常系]ユーザ情報を元にユーザを検索する
     - `SampleService#findOne(User user)`_ |br| `SampleServiceImplTest#findOneAbnormalTest()`_
     - ・Service実行の結果、正しくビジネス例外が返されるか |br| ・例外に正しくメッセージが設定されるか

   * - [正常系]ユーザを追加する
     - `SampleService#add(User user)`_ |br| `SampleServiceImplTest#addTestNormalCase()`_
     - ・Service実行の結果、正しくアウトプットが返されるか

   * - [異常系]ユーザを追加する
     - `SampleService#add(User user)`_ |br| `SampleServiceImplTest#addTestAbnormalCase()`_
     - ・Service実行の結果、正しくビジネス例外が返されるか

   * - [正常系]ユーザを更新する
     - `SampleService#update(User user)`_ |br| `SampleServiceImplTest#updateTestNormalCase()`_
     - ・Service実行の結果、正しくアウトプットが返されるか

   * - [異常系]ユーザを更新する
     - `SampleService#update(User user)`_ |br| `SampleServiceImplTest#updateTestAbnormalCase1()`_
     - ・Service実行の結果、正しくビジネス例外が返されるか |br| ・例外に正しくメッセージが設定されるか

   * - [異常系]ユーザを更新する
     - `SampleService#update(User user)`_ |br| `SampleServiceImplTest#updateTestAbnormalCase2()`_
     - ・Service実行の結果、正しくビジネス例外が返されるか |br| ・例外に正しくメッセージが設定されるか

   * - [異常系]ユーザを削除する
     - `SampleService#delete(User user)`_ |br| `SampleServiceImplTest#deleteTestAbnormalCase()`_
     - ・Service実行の結果、正しくビジネス例外が返されるか |br| ・例外に正しくメッセージが設定されるか

   * - [異常系]指定されたログインIDをもつユーザを検索する。
     - `SampleService#findUserHave(String loginId)`_ |br| `SampleServiceImplTest#findUserHaveLoginAbnormalTest()`_
     - ・Service実行の結果、正しくビジネス例外が返されるか |br| ・例外に正しくメッセージが設定されるか

   * - [異常系]指定されたユーザが持つアドレスを取得する。
     - `SampleOneToOneService#findAddressOf(User user)`_ |br| `SampleOneToOneServiceImplTest#findAddressOfAbnormalTest()`_
     - ・Service実行の結果、正しくビジネス例外が返されるか |br| ・例外に正しくメッセージが設定されるか

   * - [正常系]住所を更新する
     - `SampleOneToOneService#update(Address address)`_ |br| `SampleOneToOneServiceImplTest#updateNormalTest()`_
     - ・Service実行の結果、正しくアウトプットが返されるか

   * - [異常系]住所を更新する
     - `SampleOneToOneService#update(Address address)`_ |br| `SampleOneToOneServiceImplTest#updateAbnormalTest()`_
     - ・Service実行の結果、正しくビジネス例外が返されるか |br| ・例外に正しくメッセージが設定されるか

   * - [異常系]指定されたユーザが持つEmailを取得する
     - `SampleOneToManyService#getEmailsOf(User user)`_ |br| `SampleOneToManyServiceImplTest#getEmailsOfAbnormalTest()`_
     - ・Service実行の結果、正しくビジネス例外が返されるか |br| ・例外に正しくメッセージが設定されるか

   * - [異常系]Emailを追加する
     - `SampleOneToManyService#add(Email email)`_ |br| `SampleOneToManyServiceImplTest#addAbnormalTest()`_
     - ・Service実行の結果、正しくビジネス例外が返されるか |br| ・例外に正しくメッセージが設定されるか

   * - [異常系]Emailを更新する
     - `SampleOneToManyService#update(Email email)`_ |br| `SampleOneToManyServiceImplTest#updateAbnormalTest()`_
     - ・Service実行の結果、正しくビジネス例外が返されるか |br| ・例外に正しくメッセージが設定されるか

   * - [異常系]Emailを削除する
     - `SampleOneToManyService#delete(Email email)`_ |br| `SampleOneToManyServiceImplTest#deleteAbnormalTest1()`_
     - ・Service実行の結果、正しくビジネス例外が返されるか |br| ・例外に正しくメッセージが設定されるか

   * - [異常系]Emailを削除する
     - `SampleOneToManyService#delete(Email email)`_ |br| `SampleOneToManyServiceImplTest#deleteAbnormalTest2()`_
     - ・Service実行の結果、正しくビジネス例外が返されるか |br| ・例外に正しくメッセージが設定されるか

   * - [異常系]Emailを削除する
     - `SampleOneToManyService#delete(Email email)`_ |br| `SampleOneToManyServiceImplTest#deleteAbnormalTest3()`_
     - ・Service実行の結果、正しくビジネス例外が返されるか |br| ・例外に正しくメッセージが設定されるか

   * - [異常系]Emailを全て削除する
     - `SampleOneToManyService#deleteAllEmail(User user)`_ |br| `SampleOneToManyServiceImplTest#deleteAllEmailAbnormalTest()`_
     - ・Service実行の結果、正しくビジネス例外が返されるか |br| ・例外に正しくメッセージが設定されるか

   * - [異常系]ユーザをグループに追加する
     - `SampleManyToManyService#addUserTo(Group group, User addUser)`_ |br| `SampleManyToManyServiceImplTest#addUserToGroupAbnormalTest1()`_
     - ・Service実行の結果、正しくビジネス例外が返されるか |br| ・例外に正しくメッセージが設定されるか

   * - [異常系]ユーザをグループに追加する
     - `SampleManyToManyService#addUserTo(Group group, User addUser)`_ |br| `SampleManyToManyServiceImplTest#addUserToGroupAbnormalTest2()`_
     - ・Service実行の結果、正しくビジネス例外が返されるか |br| ・例外に正しくメッセージが設定されるか

   * - [異常系]ユーザをグループから削除する
     - `SampleManyToManyService#deleteUserFrom(Group group, User deleteUser)`_ |br| `SampleManyToManyServiceImplTest#addUserToGroupAbnormalTest1()`_
     - ・Service実行の結果、正しくビジネス例外が返されるか |br| ・例外に正しくメッセージが設定されるか

   * - [異常系]ユーザをグループから削除する
     - `SampleManyToManyService#deleteUserFrom(Group group, User deleteUser)`_ |br| `SampleManyToManyServiceImplTest#addUserToGroupAbnormalTest2()`_
     - ・Service実行の結果、正しくビジネス例外が返されるか |br| ・例外に正しくメッセージが設定されるか

   * - [異常系]グループを削除する
     - `SampleManyToManyService#delete(Group group)`_ |br| `SampleManyToManyServiceImplTest#deleteGroupAbnormalTest()`_
     - ・Service実行の結果、正しくビジネス例外が返されるか |br| ・例外に正しくメッセージが設定されるか

   * - [異常系]ユーザを削除する
     - `SampleManyToManyService#delete(User user)`_ |br| `SampleManyToManyServiceImplTest#deleteUserAbnormalTest()`_
     - ・Service実行の結果、正しくビジネス例外が返されるか |br| ・例外に正しくメッセージが設定されるか

|br|

ソースコードとテストケースを突き合わせると分かる通り、Serviceのテストではカバレッジ率が上昇するほどテストケースの網羅率が上がることがわかります。
逆に、前節で示したRepositoryや、次節で紹介するControllerのテストにおけるカバレッジはテスト品質を表す指標としては意味がないので注意しましょう。

|br|

.. _section-controller-test-for-microservice-label:

Controllerの単体テスト実装
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

本アプリケーションでは、マイクロサービスをRESTfulなアプリケーションとして作成しています。RESTfulなアプリケーションをSpringで実装する場合の考え方や実装する方法については、
`TERASOLUNAのガイドライン RESTful Web Service <https://terasolunaorg.github.io/guideline/5.5.1.RELEASE/ja/ArchitectureInDetail/WebServiceDetail/REST.html>`_ を適宜参照してください。
SpringMVCにおけるControllerはRestControllerとして作成し、基本的に、正常応答時はHTTPステータス200でリソースのJSONレスポンスを、
ビジネス例外(業務的に想定される例外)発生時はHTTPステータス400(BadRequest)で、原因やパラメータを設定したBusinessExceptionのJSONレスポンスを、
システム例外発生時はHTTPステータスで500(InternalServerError)で原因やパラメータを設定したSystemExceptionのJSONレスポンスを返却するような仕様としています。
例外のハンドリングは@ControllerAdviceを付与した `CommonExceptionHandler <https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/common/src/main/java/org/debugroom/mynavi/sample/continuous/integration/common/apinfra/exception/CommonExceptionHandler.java>`_ クラスで実装していますが、本アプリケーションでは純粋にSpringのみの依存としたいため、
例外ハンドラに加え、BusinessExceptionやSystemExceptionはTERASOLUNAが提供している共通ライブラリは使用せず、個別にAP基盤部品として作成しています。

RestControllerの単体テストでは、Serviceの単体テストと同じく@SpringBootTestを使ってテスト用のコンテナを起動し、処理の妥当性を検証することも可能ですが、
DIコンテナ生成に伴うオーバヘッドを軽減するために、よりテスト環境構築をライトに構築できる@WebMvcTestを使ってテストケースを実装します。上記に示したイメージ通り、MockMvcがドライバのような位置付けで
テスト対象のControllerクラスを呼び出し、Sevice以下のコンポーネントはMockとしてスタブ化した上で、Controllerクラスの実装の妥当性を検証します。
サンプルのテストコードは以下のようになります。

|br|

.. sourcecode:: java

   package org.debugroom.mynavi.sample.continuous.integration.backend.app.web;

   import static org.hamcrest.core.Is.is;
   import static org.junit.Assert.*;

   import com.fasterxml.jackson.databind.ObjectMapper;
   import org.junit.experimental.runners.Enclosed;
   import org.junit.runner.RunWith;
   import org.springframework.test.context.junit4.SpringRunner;
   import org.springframework.http.HttpStatus;
   import org.springframework.test.web.servlet.MockMvc;
   import org.springframework.test.web.servlet.MvcResult;
   import org.springframework.test.web.servlet.request.MockMvcRequestBuilders;
   import org.springframework.test.web.servlet.result.MockMvcResultMatchers;

   import org.debugroom.mynavi.sample.continuous.integration.common.apinfra.exception.BusinessExceptionResponse;
   import org.debugroom.mynavi.sample.continuous.integration.common.apinfra.exception.ErrorResponse;

   // omit

   @RunWith(Enclosed.class)
   public class BackendControllerTest {

       @RunWith(SpringRunner.class)
       @WebMvcTest(controllers = BackendContoller.class)
       public static class UnitTest{

           @Autowired
           ObjectMapper objectMapper;

           @Autowired
           MockMvc mockMvc;

           @MockBean
           SampleService sampleService;

           //omit

           @Before
           public void setUp() throws Exception{
              //omit
              User mockUser2 = User.builder()
                    .userId(1).firstName("hanako").familyName("mynavi").loginId("hanako.mynavi")
                    .isLogin(false).addressByUserId(mockAddress2).emailsByUserId(Arrays.asList(new Email[]{mockEmail3, mockEmail4}))
                 .build();
              Mockito.when(sampleService.findOne(mockUser2)).thenReturn(mockUser2);
              Mockito.when(sampleService.findOne(User.builder().userId(3).build()))
                       .thenThrow(new BusinessException("E0001", "", new Long[]{3L}));
              //omit
           }

           @Test
           public void getUserNormalTest() throws Exception{
               // omit
               UserResource userResource = UserResource.builder()
                        .userId(1).firstName("hanako").familyName("mynavi").loginId("hanako.mynavi")
                        .address(addressResource).emailList(Arrays.asList(new EmailResource[]{emailResource1, emailResource2}))
                   .build();
               mockMvc.perform(MockMvcRequestBuilders
                   .get("/api/v1/users/{userId}", 1)
                   .contentType(MediaType.APPLICATION_JSON))
                   .andExpect(MockMvcResultMatchers.status().is(HttpStatus.OK.value()))
                   .andExpect(MockMvcResultMatchers.content().string(
                        objectMapper.writeValueAsString(userResource)));
          }

           @Test
           public void getUserAbnormalTest() throws Exception{
               MvcResult mvcResult = mockMvc.perform(MockMvcRequestBuilders.get(
                  "/api/v1/users/{userId}", "3"))
                  .andExpect(MockMvcResultMatchers.status().is(HttpStatus.BAD_REQUEST.value()))
                  .andReturn();

               BusinessExceptionResponse businessExceptionResponse = (BusinessExceptionResponse)
                  objectMapper.readValue(mvcResult.getResponse().getContentAsString(),
                  ErrorResponse.class);

               assertThat(businessExceptionResponse.getBusinessException().getCode(), is("E0001"));
               assertThat(businessExceptionResponse.getBusinessException().getArgs(), is(new Integer[]{3}));
           }

           //omit

       }
   }

|br|

正常系、異常系ともに戻り値はJSON文字列になりますので、レスポンスとなるリソースやException(ここでは抽象的なエラーレスポンスインターフェースクラスとして `ErrorResponse <https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/common/src/main/java/org/debugroom/mynavi/sample/continuous/integration/common/apinfra/exception/ErrorResponse.java>`_ )を
JacsonのObjectMapperでシリアライズして期待値と一致するかを検証しています。

|br|

.. note:: ErrorResponseではcom.fasterxml.jackson.annotation.JsonSubTypesやcom.fasterxml.jackson.annotation.JsonTypeInfoを使用してデシリアライズ時に動的にクラスを切り替えれるようにしています。

|br|

ドメインオブジェクト⇔リソースマッピングなどレイヤ間を跨ぐ変換処理実装の妥当性も合わせて検証できるようにService呼び出しの入出力時にマッピング処理を実装しておくと、わざわざMapperを単体テストするより効率的に済みます。
サンプルで作成したテストケースのユースケース・検証観点は以下の通りです。

|br|

.. _BackendController#getUser(Long userId): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/main/java/org/debugroom/mynavi/sample/continuous/integration/backend/app/web/BackendContoller.java#L58
.. _UserMapper: https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/main/java/org/debugroom/mynavi/sample/continuous/integration/backend/app/model/UserMapper.java
.. _BackendControllerTest#getUserNormalTest(): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/test/java/org/debugroom/mynavi/sample/continuous/integration/backend/app/web/BackendControllerTest.java#L151
.. _BackendControllerTest#getUserAbnormalTest(): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/test/java/org/debugroom/mynavi/sample/continuous/integration/backend/app/web/BackendControllerTest.java#L184
.. _BackendController#addUser(User user): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/main/java/org/debugroom/mynavi/sample/continuous/integration/backend/app/web/BackendContoller.java#L65
.. _User: https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/main/java/org/debugroom/mynavi/sample/continuous/integration/backend/app/model/User.java
.. _BackendControllerTest#addUserInputParamNormalTest(): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/test/java/org/debugroom/mynavi/sample/continuous/integration/backend/app/web/BackendControllerTest.java#L199
.. _BackendControllerTest#addUserInputParamAbnormalTest(): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/test/java/org/debugroom/mynavi/sample/continuous/integration/backend/app/web/BackendControllerTest.java#L263
.. _BackendController#updateUser(User user): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/main/java/org/debugroom/mynavi/sample/continuous/integration/backend/app/web/BackendContoller.java#L74
.. _BackendControllerTest#updateUserInputParamNormalTest(): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/test/java/org/debugroom/mynavi/sample/continuous/integration/backend/app/web/BackendControllerTest.java#L340
.. _BackendController#findUserOfLoginId(User user): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/main/java/org/debugroom/mynavi/sample/continuous/integration/backend/app/web/BackendContoller.java#L88
.. _BackendControllerTest#findUserByLoginIdInputParamAbnormalTest(): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/test/java/org/debugroom/mynavi/sample/continuous/integration/backend/app/web/BackendControllerTest.java#L403
.. _Address: https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/main/java/org/debugroom/mynavi/sample/continuous/integration/backend/app/model/Address.java
.. _AddressMapper: https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/main/java/org/debugroom/mynavi/sample/continuous/integration/backend/app/model/AddressMapper.java
.. _BackendController#updateAddress(Address address): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/main/java/org/debugroom/mynavi/sample/continuous/integration/backend/app/web/BackendContoller.java#L118
.. _BackendControllerTest#updateAddressInputParamAbnormalTest(): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/test/java/org/debugroom/mynavi/sample/continuous/integration/backend/app/web/BackendControllerTest.java#L417
.. _Email: https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/main/java/org/debugroom/mynavi/sample/continuous/integration/backend/app/model/Email.java
.. _EmailMapper: https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/main/java/org/debugroom/mynavi/sample/continuous/integration/backend/app/model/EmailMapper.java
.. _BackendController#findUserHavingEmail(Email email): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/main/java/org/debugroom/mynavi/sample/continuous/integration/backend/app/web/BackendContoller.java#L132
.. _BackendControllerTest#findUserHavingEmailInputParamAbnormalTest(): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/test/java/org/debugroom/mynavi/sample/continuous/integration/backend/app/web/BackendControllerTest.java#L452
.. _BackendController#addEmail(Email email): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/main/java/org/debugroom/mynavi/sample/continuous/integration/backend/app/web/BackendContoller.java#L139
.. _BackendControllerTest#addEmailInputParamAbnormalTest(): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/test/java/org/debugroom/mynavi/sample/continuous/integration/backend/app/web/BackendControllerTest.java#L471
.. _BackendController#updateEmail(Email email): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/main/java/org/debugroom/mynavi/sample/continuous/integration/backend/app/web/BackendContoller.java#L146
.. _BackendControllerTest#updateEmailInputParamAbnormalTest(): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/test/java/org/debugroom/mynavi/sample/continuous/integration/backend/app/web/BackendControllerTest.java#L495
.. _BackendController#deleteEmail(Email email): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/main/java/org/debugroom/mynavi/sample/continuous/integration/backend/app/web/BackendContoller.java#L153
.. _BackendControllerTest#deleteEmailInputParamAbnormalTest(): https://github.com/debugroom/mynavi-sample-continuous-integration/blob/master/backend/src/test/java/org/debugroom/mynavi/sample/continuous/integration/backend/app/web/BackendControllerTest.java#L531

.. list-table::
   :widths: 7, 6, 7

   * - ユースケース
     - 主な処理実装クラス・メソッド |br| テストメソッド
     - 検証観点

   * - [正常系]ユーザリソースを取得する
     - `BackendController#getUser(Long userId)`_ |br| `UserMapper`_ |br| |br| `BackendControllerTest#getUserNormalTest()`_
     - ・指定したHTTPメソッドやURLで正しくリクエストハンドリングされるか |br| ・リクエストパラメータやパス変数が正しくマッピングされるか |br| ・レイヤ間のモデルオブジェクト変換は正しくマッピングされるか

   * - [異常系]ユーザリソースを取得する
     - `BackendController#getUser(Long userId)`_ |br| `BackendControllerTest#getUserAbnormalTest()`_
     - ・入力チェックエラーやビジネスエラー発生時に正しいHTTPステータスを返却するか |br| ・入力チェックエラーやビジネスエラー発生時に正しいメッセージやパラメータを返却するか

   * - [正常系]ユーザリソースを追加する
     - `BackendController#addUser(User user)`_ |br| `User`_ |br| |br| `BackendControllerTest#addUserInputParamNormalTest()`_
     - ・指定したHTTPメソッドやURLで正しくリクエストハンドリングされるか |br| ・リクエストパラメータやパス変数が正しくマッピングされるか |br| ・入力チェックが正しく行われているか

   * - [異常系]ユーザリソースを追加する
     - `BackendController#addUser(User user)`_ |br| `User`_ |br| |br| `BackendControllerTest#addUserInputParamAbnormalTest()`_
     - ・入力チェックエラーやビジネスエラー発生時に正しいHTTPステータスを返却するか |br| ・入力チェックエラーやビジネスエラー発生時に正しいメッセージやパラメータを返却するか

   * - [正常系]ユーザリソースを更新する
     - `BackendController#updateUser(User user)`_ |br| `User`_ |br| |br| `BackendControllerTest#updateUserInputParamNormalTest()`_
     - ・指定したHTTPメソッドやURLで正しくリクエストハンドリングされるか |br| ・リクエストパラメータやパス変数が正しくマッピングされるか

   * - [異常系]指定されたログインIDをもつユーザリソースを取得する
     - `BackendController#findUserOfLoginId(User user)`_ |br| `User`_ |br| |br| `BackendControllerTest#findUserByloginIdInputParamAbnormalTest()`_
     - ・入力チェックエラーやビジネスエラー発生時に正しいHTTPステータスを返却するか |br| ・入力チェックエラーやビジネスエラー発生時に正しいメッセージやパラメータを返却するか

   * - [異常系]住所リソースを更新する
     - `BackendController#updateAddress(Address address)`_ |br| `Address`_ |br| `AddressMapper`_ |br| |br| `BackendControllerTest#updateAddressInputParamAbnormalTest()`_
     - ・入力チェックエラーやビジネスエラー発生時に正しいHTTPステータスを返却するか |br| ・入力チェックエラーやビジネスエラー発生時に正しいメッセージやパラメータを返却するか |br| ・レイヤ間のモデルオブジェクト変換は正しくマッピングされるか


   * - [異常系]指定されたEmailをもつユーザリソースを取得する
     - `BackendController#findUserHavingEmail(Email email)`_ |br| `Email`_ |br| `EmailMapper`_ |br| |br| `BackendControllerTest#findUserHavingEmailInputParamabnormalTest()`_
     - ・入力チェックエラーやビジネスエラー発生時に正しいHTTPステータスを返却するか |br| ・入力チェックエラーやビジネスエラー発生時に正しいメッセージやパラメータを返却するか |br| ・レイヤ間のモデルオブジェクト変換は正しくマッピングされるか


   * - [異常系]メールリソースを追加する
     - `BackendController#addEmail(Email email)`_ |br| `Email`_ |br| |br| `BackendControllerTest#addEmailInputParamAbnormalTest()`_
     - ・入力チェックエラーやビジネスエラー発生時に正しいHTTPステータスを返却するか |br| ・入力チェックエラーやビジネスエラー発生時に正しいメッセージやパラメータを返却するか

   * - [異常系]メールリソースを更新する
     - `BackendController#updateEmail(Email email)`_ |br| `Email`_ |br| |br| `BackendControllerTest#updateEmailInputParamAbnormalTest()`_
     - ・入力チェックエラーやビジネスエラー発生時に正しいHTTPステータスを返却するか |br| ・入力チェックエラーやビジネスエラー発生時に正しいメッセージやパラメータを返却するか

   * - [異常系]メールリソースを削除する
     - `BackendController#deleteEmail(Email email)`_ |br| `Email`_ |br| |br| `BackendControllerTest#deleteEmailInputParamAbnormalTest()`_
     - ・入力チェックエラーやビジネスエラー発生時に正しいHTTPステータスを返却するか |br| ・入力チェックエラーやビジネスエラー発生時に正しいメッセージやパラメータを返却するか

|br|

特にControllerのテスト対象は、ソースコードとテストコードを見て分かる通り、リクエストのマッピングの妥当性だけではなく、リクエストパラメータのバリデーション定義が期待通り動作するかや、チェックが正しいタイミング(ユースケース)で実行されるかなど検証内容が複雑かつ多岐に渡ります。
単純にデータを取得するだけの正常系のユースケースは後々の結合試験で確認できますので、Controllerの単体テストでは、境界値試験など含め、リクエストパラメータの異常系バリエーションを充実させて検証した方が良いでしょう。
Controllerの設定誤りはセキュリティホールに直結しますので、各実装が少なくとも一度はテストパスすることを推奨します。

|br|

.. note:: SpringMVCにおける入力チェックの基本は `TERASOLUNAのガイドライン 入力チェック <http://terasolunaorg.github.io/guideline/5.5.1.RELEASE/ja/ArchitectureInDetail/WebApplicationDetail/Validation.html>`_ や、 `RESTful Web Serviceにおける入力エラー例外のハンドリング実装 <http://terasolunaorg.github.io/guideline/5.5.1.RELEASE/ja/ArchitectureInDetail/WebServiceDetail/REST.html#resthowtouseexceptionhandlingforvalidationerror>`_ を適宜参照してください。

|br|

.. _section-unit-test-strategy-for-microservice-label:

マイクロサービスにおける単体テスト戦略と品質評価
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

これまで、Repository、Service、Controllerの単体テストコード実装を解説してきました。単体テストのコードだけでも実アプリケーションのコードよりもはるかにボリュームが多く、
あらゆる異常系のテストを網羅しようとするとかなり大変なことがお分りいただけたのではないでしょうか。前回の説明の再掲にはなりますが、繰り返しのテストが発生しがちなマイクロサービスですが、
初めから完璧にテストコードを整備しておく必要もありませんし、必要以上のテストコードの実装でかえって開発のアジリティを損なうようでは本末転倒です。
ただ、テストが疎かになるとせっかくの継続的インテグレーションも機能しません。開発のスピードと品質を両立するために、テスト計画やスコープ、検証の観点を明示的に策定しておくことが重要です。

効率的な単体テスト戦略策定のポイントとしては、

* ServiceやRepositoryにおける単純なデータ取得の正常系テストなど、結合試験でも重複して登場するテストケースは単体テストから除外する。
* データベース更新結果など結合試験で効率的に検証できるテストケースは単体テストから除外する。
* Controllerの設定誤りなどはセキュリティホールに直結するため、異常系のバリエーションを充実させたり、実装が少なくとも一度はテストパスさせる。
* 探索的テストを導入し、実装状況に応じてテストケースの重複を極力減らしながらテストコードを作成する
* 機能や処理の重要度に応じて、テスト実施内容に濃淡をつける(ビジネス的にそこまで重要でない処理の参照系はテストしない等)

テスト品質は、これまでも見てきた通り、Serviceを覗き、単純なカバレッジのみでは評価できないので、ユースケース数に対するテストケースの割合、テストケースの定性的評価などを加えつつ評価するとよいでしょう。

|br|

次回は引き続き、SpringBootを使った結合試験のテストコードを実装し、解説していきます。

著者紹介
------------------------------------------------------------------

川畑 光平(KAWABATA Kohei) - NTTデータ 課長代理

.. figure:: img/automation_infra_devops_overview/pic_image01.jpg
   :scale: 100%

金融機関システム業務アプリケーション開発・システム基盤担当を経て、現在はソフトウェア開発自動化関連の研究開発・推進に従事。

Red Hat Certified Engineer、Pivotal Certified Spring Professional、AWS Certified Solutions Architect Professional等の資格を持ち、アプリケーション基盤・クラウドなど様々な開発プロジェクト支援にも携わる。
