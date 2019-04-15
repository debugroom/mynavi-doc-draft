.. include:: ../module.txt

.. _section-cloud-native-rds-label-1-3:

AWSで作るクラウドネイティブアプリケーションの基本
========================================================================================

.. _section-cloud-native-rds-3rd-label:

Amazon RDSにアクセスするSpringアプリケーション(3)
----------------------------------------------------------------------------------------

|br|

クラウド上のリレーショナルデータベースとして、AWSで利用可能なAmazon RDS。今回はPostgreSQLをベースとするAmazon RDSへアクセスする
Springアプリケーションの実装方法についてわかりやすく解説します。なお、データベースアクセスの実装にはSpring Data JPAおよび、
Spring Cloud AWSを用いて実装します。

|br|

#. Amazon RDSの概要とデータベース構築
#. RDSへのテーブル構築とSpring Data JPAおよびSpring Cloud AWSを用いたRDSアクセスアプリケーションの実装(1)
#. **RDSへのテーブル構築とSpring Data JPAおよびSpring Cloud AWSを用いたRDSアクセスアプリケーションの実装(2)**

|br|

.. _section-cloud-native-spring-data-jpa-and-spring-cloud-aws-implementation-2nd-label:

Spring Data JPAおよびSpring Cloud AWSを用いたRDSアクセスアプリケーションの実装(2)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

前回に引き続き、アプリケーションの処理実装に進みます。まずは前回作成した以下のテーブルに対応したエンティティクラスを実装します。

|br|

.. figure:: img/aws-rds/sample_database.png
   :scale: 100%

|br|

テーブルに対応したJavaクラスを作成し、JPA標準のjavax.persistenceパッケージのアノテーションを付与します。代表的なアノテーションの概要は以下の通りです。

|br|

.. list-table:: javax.persistenceアノテーションの概要
   :header-rows: 1
   :widths: 30,20,50

   * - アノテーション
     - 設定
     - 概要
   * - @Entity
     - クラス
     - エンティティとして表現するクラスに付与する。
   * - @Table
     - クラス
     - エンティティとマッピングする物理テーブル名を指定する。
   * - @Embeddable
     - クラス
     - 主に複合IDとして表現する組み込みクラスに付与する。
   * - @Id
     - フィールド |br| メソッド
     - エンティティクラスのプライマリキーのフィールドに指定する。
   * - @EmbeddedId
     - フィールド |br| メソッド
     - エンティティクラスの複合IDプライマリーキーのフィールドに指定する。
   * - @Column
     - フィールド |br| メソッド
     - フィールドとデータベースのカラム名のマッピングを定義する。
   * - @Basic
     - フィールド |br| メソッド
     - LazyFetchオブションなど指定する。
   * - @JoinColumns
     - フィールド |br| メソッド
     - @JoinColumnを複数同時に記述する場合に利用する。
   * - @JoinColumn
     - フィールド |br| メソッド
     - 結合のための外部キーカラム及び外部参照制約オプションを指定する。
   * - @PraimaryKeyJoinColums
     - フィールド |br| メソッド
     - @PrimaryKeyJoinColumnを複数同時に記述する場合に利用する。
   * - @PraimaryKeyJoinColumn
     - フィールド |br| メソッド
     - 他のテーブルと結合する場合に、外部キーとして扱われるカラム
   * - @OneToOne
     - フィールド |br| メソッド
     - OneToOneリレーションシップであることを示す場合に付与する。
   * - @OneToMany
     - フィールド |br| メソッド
     - OneToManyリレーションシップであることを示す場合に付与する。
   * - @ManyToOne
     - フィールド |br| メソッド
     - ManyToOneリレーションシップであることを示す場合に付与する。
   * - @AttributeOverrides
     - フィールド |br| メソッド
     - @AttributeOverrideを複数記述する場合に利用する。
   * - @AttributeOverride
     - フィールド |br| メソッド
     - マッピング情報をオーバーライドする。
   * - @Transient
     - フィールド |br| メソッド
     - 永続化しないエンティティクラス、マップドスーパークラス、または組み込みクラスのフィールドであることを示す。
   * - @Temporal
     - フィールド |br| メソッド
     - 時刻を示す型(java.util.Date及び、java.util.Calendar)を持つフィールドに付与する。
   * - @GeneratedValue
     - フィールド |br| メソッド
     - プライマリーキーカラムにユニークな値を自動生成、付与する方法を指定する。
   * - @NamedQuery
     - クラス
     - JPQLの名前付きクエリを指定する。

|br|

上記の表を参考に、各テーブルごとに対応したエンティティクラスを作成します。一例としてユーザエンティティクラスの実装例を一部抜粋して載せますが、各テーブルごとのエンティティクラスの実装の詳細は 、
`エンティティクラスのパッケージ <https://github.com/debugroom/mynavi-sample-aws-rds/tree/master/src/main/java/org/debugroom/mynavi/sample/aws/rds/domain/model/entity>`_ を参照してください。

|br|

.. sourcecode:: java

   package org.debugroom.mynavi.sample.aws.rds.domain.model.entity;

   import lombok.AllArgsConstructor;
   import lombok.Builder;
   import lombok.NoArgsConstructor;

   import javax.persistence.*;
   import java.sql.Timestamp;
   import java.util.Collection;
   import java.util.Objects;

   @AllArgsConstructor
   @NoArgsConstructor
   @Builder
   @Entity
   @Table(name = "usr", schema = "public", catalog = "sample_database")
   public class User {

       private long userId;
       private String firstName;
       private String familyName;
       private String loginId;
       private Boolean isLogin;
       private Integer ver;
       private Timestamp lastUpdatedAt;
       private Address addressByUserId;
       private Collection<Email> emailsByUserId;
       private Collection<Membership> membershipsByUserId;

       // omit

       @Id
       @Column(name = "user_id", nullable = false)
       public long getUserId() {
           return userId;
       }

       @Basic
       @Column(name = "first_name", nullable = true, length = -1)
       public String getFirstName() {
           return firstName;
       }

       @Basic
       @Column(name = "family_name", nullable = true, length = -1)
       public String getFamilyName() {
           return familyName;
       }

       @Basic
       @Column(name = "login_id", nullable = true, length = -1)
       public String getLoginId() {
           return loginId;
       }

       @Basic
       @Column(name = "is_login", nullable = true)
       public Boolean getLogin() {
           return isLogin;
       }

       @Basic
       @Column(name = "ver", nullable = true)
       @Version
       public Integer getVer() {
           return ver;
       }

       @Basic
       @Column(name = "last_updated_at", nullable = true)
       public Timestamp getLastUpdatedAt() {
           return lastUpdatedAt;
       }

       @OneToOne(mappedBy = "usrByUserId", cascade=CascadeType.ALL)
       public Address getAddressByUserId() {
           return addressByUserId;
       }

       @OneToMany(mappedBy = "usrByUserId", cascade=CascadeType.ALL)
       public Collection<Email> getEmailsByUserId() {
           return emailsByUserId;
       }

       @OneToMany(mappedBy = "usrByUserId", cascade = CascadeType.ALL)
       public Collection<Membership> getMembershipsByUserId() {
           return membershipsByUserId;
       }
    }


|br|

続いて、ユーザテーブルへアクセスするコンポーネントであるUserRepositoryインターフェースは、以下の様な要領で実装しておく必要があります。

* org.springframework.data.jpa.repository.JpaRepositoryを継承する
* テーブルをモデル化したクラスとキーをJpaRepositoryの型パラメータに設定する。

各テーブルごとのレポジトリクラスの実装は 、
`レポジトリクラスのパッケージ <https://github.com/debugroom/mynavi-sample-aws-rds/tree/master/src/main/java/org/debugroom/mynavi/sample/aws/rds/domain/repository>`_ を参照してください。

|br|

.. sourcecode:: java

   package org.debugroom.mynavi.sample.aws.rds.domain.repository;

   import org.springframework.data.jpa.repository.JpaRepository;
   import org.debugroom.mynavi.sample.aws.rds.domain.model.entity.User;

   public interface UserRepository extends JpaRepository<User, Long> {
   }

|br|

.. note:: 特に実装クラスも作成せずインターフェースだけで、findAllやsaveメソッドが実行できる理由は、Spring Data JPAが、GenericDAOパターンに基づく実装クラスを提供しているためです。
   GenericDAOパターンとはJavaの型パラメータを利用した実装で、CRUD処理をJavaのジェネリクス機構を使って実装しておき、テーブルの種類に応じた、
   型パラメータクラスを設定することで、汎用的なCRUD共通処理を実装したDAO(DatabaseAccessObject)を作成しておくパターンです。
   Spring Data JPAに限らず、Spring Data XXXなど類似一連のプロダクトでは、同様にインターフェースを作成するだけで、基本的なCRUDはほぼ実装せずにデータベースアクセス処理を行うことが可能です。

|br|

最後にトランザクション境界が設定されるServiceクラスです。@Transactionalアノテーションと@Serviceアノテーションを付与したService実装クラスを作成します。
こちらも一部の抜粋になりますが、この処理では各テーブルにデータを保存する処理を記述しています。
詳細は `サービスクラス <https://github.com/debugroom/mynavi-sample-aws-rds/blob/master/src/main/java/org/debugroom/mynavi/sample/aws/rds/domain/service/SampleServiceImpl.java>`_ を参照してください。


.. sourcecode:: java

   package org.debugroom.mynavi.sample.aws.rds.domain.service;

   import org.springframework.beans.factory.annotation.Autowired;
   import org.springframework.stereotype.Service;
   import org.springframework.transaction.annotation.Transactional;

   import org.debugroom.mynavi.sample.aws.rds.domain.model.entity.*;
   import org.debugroom.mynavi.sample.aws.rds.domain.repository.*;

   @Transactional
   @Service
   public class SampleServiceImpl implements SampleService{

       @Autowired
       UserRepository userRepository;

       @Autowired
       GroupRepository groupRepository;

       @Override
       public void init(){
           userRepository.deleteAll();
           groupRepository.deleteAll();
       }

       @Override
       public void setData() {
           init();
           // omit
           userRepository.saveAll(Arrays.asList(new User[]{user1, user2}));
       }

.. note:: ユーザとグループのエンティティクラスには関連テーブルの属性にCascadeType.ALLを指定しているため、関連テーブルを含めデータの追加、削除が行われます。

|br|

アプリケーション起動クラスを実行した後、PSQLクライアントからRDSにアクセスすると、各テーブルにデータが追加されていることが確認できます。

.. sourcecode:: bash

   sample_database=> select * from usr;
    user_id | first_name | family_name |    login_id    | is_login | ver |     last_updated_at
   ---------+------------+-------------+----------------+----------+-----+-------------------------
          0 | taro       | mynavi      | taro-account   | f        |   0 | 2019-04-11 01:31:26.064
          1 | hanako     | mynavi      | hanako-account | f        |   0 | 2019-04-11 01:31:26.064
   (2 rows)


   sample_database=> select * from grp;
    group_id |  group_name   | ver |     last_updated_at
   ----------+---------------+-----+-------------------------
           0 | mynavi-group1 |   0 | 2019-04-11 01:31:25.996
           1 | mynavi-group2 |   0 | 2019-04-11 01:31:25.996
   (2 rows)

   sample_database=> select * from membership ;
    user_id | group_id | ver |     last_updated_at
   ---------+----------+-----+-------------------------
          0 |        0 |   0 | 2019-04-11 01:31:26.064
          0 |        1 |   0 | 2019-04-11 01:31:26.064
          1 |        0 |   0 | 2019-04-11 01:31:26.064
   (3 rows)

   sample_database=> select * from address;
    user_id | zip_code |     address     | ver |     last_updated_at
   ---------+----------+-----------------+-----+-------------------------
          0 | 100-0000 | Tokyo Chiyodaku |   0 | 2019-04-11 01:31:26.064
          1 | 100-0000 | Tokyo Chuouku   |   0 | 2019-04-11 01:31:26.064
   (2 rows)

   sample_database=> select * from email;
    user_id | email_no |     email      | ver |     last_updated_at
   ---------+----------+----------------+-----+-------------------------
          0 |        0 | test@test.com  |   0 | 2019-04-11 01:31:26.064
          0 |        1 | test1@test.com |   0 | 2019-04-11 01:31:26.064
          1 |        0 | test2@test.com |   0 | 2019-04-11 01:31:26.064
          1 |        1 | test3@test.com |   0 | 2019-04-11 01:31:26.064
   (4 rows)

|br|

ここまで3回にわたり、AWS RDSを構築し、データベース・テーブルにアクセスするSpringアプリケーションをSpring Data JPAおよびSpring Cloud AWSを使って構築してきました。
RDSへSpringアプリケーションで接続する場合、各フレームワークが提供する機能やデータアクセスの仕組みを押さえた上で実装しておくことが重要です。

|br|

著者紹介
------------------------------------------------------------------

川畑 光平(KAWABATA Kohei)

.. figure:: img/aws-lambda-and-api-gateway/pic_image01.jpg
   :scale: 100%

某システムインテグレータにて、金融機関システム業務アプリケーション開発・システム基盤担当を経て、現在はソフトウェア開発自動化関連の研究開発・推進に従事。

Red Hat Certified Engineer、Pivotal Certified Spring Professional、AWS Certified Solutions Architect Professional等の資格を持ち、アプリケーション基盤・クラウドなど様々な開発プロジェクト支援にも携わる。

本連載記事の内容に対するご意見・ご質問は `Facebook <https://www.facebook.com/kohei.kawabata.5>`_ まで。
