.. include:: ../module.txt

.. _section-cloud-native-sqs-3rd-label:

【第30回】AmazonSQSを使用する実装(3)
----------------------------------------------------------------------------------------

|br|

本連載では、以下のステップで解説を進めていきます。

|br|

#. Amazon SQSの概要とアプリケーション処理パターン、SQSキューの作成
#. Spring Cloud AWSを用いたSQSProducerアプリケーション実装
#. **Spring Batchを用いたバッチアプリケーション実装(1)**
#. Spring Batchを用いたバッチアプリケーション実装(2)
#. Spring Cloud AWSを用いたSQSConsumerアプリケーション実装

|br|

前回はSQSへキューを送信するProducerアプリケーションを実装しました。今回から２回に渡り、SQSメッセージをポーリングして取得するConsumerアプリケーションから実行されるバッチ処理をSpring Batchを使って実装していきます。

|br|

.. figure:: img/aws-sqs/sqs-pattern.png

|br|

.. _section-cloud-native-sqs-batch-using-spring-bacth-implementation-label:

SpringBatchを使用したサンプルアプリケーション
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

今回作成するSpringBatchを使ったアプリケーションは以下の図のようなイメージで構成します。

.. figure:: img/aws-sqs/spring-batch-sample.png


|br|

SpringBatchの基礎的な内容については、ここでは詳細な説明は行いませんが、必要に応じて、`TERASOLUNAガイドライン SpringBatchのアーキテクチャ <https://terasoluna-batch.github.io/guideline/current/ja/single_index.html#Ch02_SpringBatchArch>`_ を参照してください。
なお、上記のガイドラインで扱っているTERASOLUNA Batchでは、SpringBatchを使ったベストプラクティスな実装方法のガイドライン提供に加えて、SpringBatchで提供されていないDBポーリングによる
ジョブの別プロセス非同期実行機能をライブラリとして提供していますが、本連載では、純粋にSpringBatchのみを使って実装します。
また、上記のガイドラインの中では、SpringBootを使ってSpring Batchアプリケーションを実装する方法については述べられていませんが、本連載では、JavaのConfigクラスを使って、SpringBootをベースとした設定方法を扱います。
SpringBatchの処理起動はメインクラスによる実行とSQSのポーリングでメッセージを取得後実行する2パターン作成します。本連載でのテーマであるSQSへのポーリングに関しては次回以降解説する、SpringCloudAWSを使ったConsumerアプリケーションの中で解説を行います。

本連載で実際に作成するSpringBatchアプリケーションは `GitHub <https://github.com/debugroom/mynavi-sample-aws-sqs/tree/master/spring-batch>`_ 上にコミットしています。
以降に記載するソースコードでは、import文など本質的でない記述を省略している部分があるので、実行コードを作成する際は、必要に応じて適宜GitHubにあるソースコードも参照してください。

SpringBootを使ってSpringBatchアプリケーションを作成するには、前回同様、Mavenプロジェクトのpom.xmlでspring-boot-starter-batchのライブラリを追加してください。
また、モデルオブジェクトを簡素化する目的でLombokライブラリを、バッチジョブのリスタートや実行結果の管理のために作られるJobRepository向けのデータベースとしてH2を追加します。

|br|

.. sourcecode:: xml

   <dependencies>
     <dependency>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter-batch</artifactId>
     </dependency>
     <dependency>
       <groupId>org.projectlombok</groupId>
       <artifactId>lombok</artifactId>
       <optional>true</optional>
     </dependency>
     <dependency>
       <groupId>com.h2database</groupId>
       <artifactId>h2</artifactId>
     </dependency>
   </dependencies>

|br|

それでは、SpringBatchアプリケーションの実装に進みます。今回作成するアプリケーションは上記イメージ通り以下のコンポーネント構成です。
バッチ処理を構成する上でよく使用される処理で構成してみます。

.. list-table:: アプリケーション
   :widths: 3, 6, 1

   * - コンポーネント
     - 説明
     - 必須

   * - SpringBatchApplication
     - SpringBootアプリケーションを実行する起動クラス。単体で実行するために作成
     -

   * - BatchAppConfig
     - バッチ処理の実行単位で作成するBean定義クラス
     - ◯

   * - BatchInfraConfig
     - ジョブ管理テーブルへのデータアクセスを定義するクラス
     - ◯

   * - SampleTasklet
     - ジョブの最初のステップで前処理として実行するTaskletクラス
     -

   * - SampleProcessor
     - ジョブのステップで実行するProcessorクラス
     -

   * - SampleWriter
     - ジョブのステップで実行するWriterクラス
     -

   * - SamplePartitioner
     - ProcessorやWriterをマルチスレッドで並列実行する場合の分割単位を定義するクラス
     -

   * - SampleListener
     - ジョブの実行前後で呼び出されるListenerクラス
     -


|br|

.. _section-cloud-native-sqs-batch-using-spring-bacth-setting-implementation-label:

SpringBatchの設定クラス実装
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

それでは、実装していくクラスを説明します。まず、最初にSpringBoot起動クラス及び、各種設定クラスです。
@SpringBootApplicaitonアノテーションが付与された起動クラスは、同一パッケージにある@Configurationアノテーションが付与された設定クラス及び、
設定クラス内で@ComponentScanされたパッケージにあるクラスを読み取ります。ここでは、起動クラスと別パッケージにある設定クラスを読み取るために、
@Importをを使用します。読み取るクラスは以下の2つ作成します。

* Batch処理を定義する設定クラス：BatchAppConfigクラス
* ジョブ実行に必要なリソースを定義するクラス：BatchInfraConfigクラス

設定クラスは必ずしも複数である必要はなく一つにまとめても動作上問題ありませんが、クラス名と役割を対応づけて作成していた方が、
後々設定内容を混乱することなく、クラス名から識別できてベターです。なお、コンポーネントスキャンを使わずに直接設定クラスを読み込む理由として、
通常バッチは単体の処理で完結するため、必要のないジョブコンポーネントの読み込みを回避して、パフォーマンスを向上させるためです。
また、下記の実装はBatchアプリケーションをローカル単体でテスト実行できるような形で実装したものになります(次回以降解説するConsumerアプリケーションではこちらのクラスから起動しません)。

|br|

.. sourcecode:: java

   package org.debugroom.mynavi.sample.aws.sqs.app.batch.launcher;

   import org.debugroom.mynavi.sample.aws.sqs.config.BatchAppConfig;
   import org.springframework.boot.SpringApplication;
   import org.springframework.boot.WebApplicationType;
   import org.springframework.boot.autoconfigure.SpringBootApplication;
   import org.springframework.boot.builder.SpringApplicationBuilder;
   import org.springframework.context.annotation.Import;

   @Import(BatchAppConfig.class)
   @SpringBootApplication
   public class SpringBatchApplication {

       public static void main(String[] args) {
           String inputParam = "param=1";
           new SpringApplicationBuilder(SpringBatchApplication.class)
                   .web(WebApplicationType.NONE)
                   .run(new String[]{inputParam});
        }
   }

|br|

起動クラスから読み込むBatchの処理定義を記載したBatchAppConfigクラスは以下の通りです。こうした設定クラスはバッチジョブ単位に作成し、起動クラスとジョブを1：1で組み合わせて、@Importで読み込んで実行するとよいでしょう。

|br|

.. sourcecode:: java

   package org.debugroom.mynavi.sample.aws.sqs.config;

   // omit

   import org.springframework.batch.core.Job;
   import org.springframework.batch.core.JobExecutionListener;
   import org.springframework.batch.core.Step;
   import org.springframework.batch.core.configuration.annotation.DefaultBatchConfigurer;
   import org.springframework.batch.core.configuration.annotation.EnableBatchProcessing;
   import org.springframework.batch.core.configuration.annotation.JobBuilderFactory;
   import org.springframework.batch.core.configuration.annotation.StepBuilderFactory;
   import org.springframework.batch.core.configuration.annotation.StepScope;
   import org.springframework.batch.core.partition.support.Partitioner;
   import org.springframework.batch.core.step.tasklet.Tasklet;
   import org.springframework.batch.item.ItemProcessor;
   import org.springframework.batch.item.ItemWriter;
   import org.springframework.batch.item.file.FlatFileItemReader;
   import org.springframework.batch.item.file.mapping.BeanWrapperFieldSetMapper;
   import org.springframework.batch.item.file.mapping.DefaultLineMapper;
   import org.springframework.batch.item.file.transform.DelimitedLineTokenizer;
   import org.springframework.beans.factory.annotation.Autowired;
   import org.springframework.beans.factory.annotation.Qualifier;
   import org.springframework.beans.factory.annotation.Value;
   import org.springframework.context.annotation.Bean;
   import org.springframework.context.annotation.Configuration;
   import org.springframework.context.annotation.Import;
   import org.springframework.core.io.DefaultResourceLoader;
   import org.springframework.core.task.SimpleAsyncTaskExecutor;
   import org.springframework.core.task.TaskExecutor;

   @Import(BatchInfraConfig.class)                                               // …(A)
   @Configuration
   @EnableBatchProcessing                                                        // …(B)
   public class BatchAppConfig extends DefaultBatchConfigurer {                  // …(C)

       @Autowired
       JobBuilderFactory jobBuilderFactory;                                      // …(D)

       @Autowired
       StepBuilderFactory stepBuilderFactory;                                    // …(E)

       @Bean
       public Job job(@Qualifier("step1") Step step1, @Qualifier("step2") Step step2){
           return jobBuilderFactory.get("job")
             .listener(jobExecutionListener())
             .start(step1)
             .next(partionStep())
             .build();                                                           // …(F)
       }

       @Bean
       public Step step1(){
           return stepBuilderFactory
             .get("step1")
             .tasklet(sampleTasklet())
             .build();                                                           // …(G)
       }

       @Bean
       protected Step step2(){
           return stepBuilderFactory.get("step2")
             .<Sample, Sample>chunk(10)
             .reader(sampleFlatFileItemReader(null))
             .processor(sampleProcessor())
             .writer(sampleWriter())
             .build();                                                           // …(H)
       }

       @Bean
       protected Step partionStep(){
           return stepBuilderFactory.get("partitionStep")
             .partitioner(step2())
             .partitioner("step2", partitioner(null))
             .taskExecutor(taskExecutor())
             .build();                                                           // …(I)
       }

       @Bean
       @StepScope
       @Value("#{jobExecutionContext['paramBySampleTasklet']}")                  // …(J)
       public FlatFileItemReader<Sample> sampleFlatFileItemReader(String paramBySampleTasklet){

           FlatFileItemReader<Sample> flatFileItemReader = new FlatFileItemReader<>();

           flatFileItemReader.setResource(new DefaultResourceLoader().getResource(paramBySampleTasklet));

           DefaultLineMapper<Sample> defaultLineMapper = new DefaultLineMapper<>();
           DelimitedLineTokenizer delimitedLineTokenizer = new DelimitedLineTokenizer();
           delimitedLineTokenizer.setNames("stepParam");
           defaultLineMapper.setLineTokenizer(delimitedLineTokenizer);

           BeanWrapperFieldSetMapper<Sample> beanWrapperFieldSetMapper = new BeanWrapperFieldSetMapper<>();
           beanWrapperFieldSetMapper.setTargetType(Sample.class);
           defaultLineMapper.setFieldSetMapper(beanWrapperFieldSetMapper);

           flatFileItemReader.setLineMapper(defaultLineMapper);

           return flatFileItemReader;                                            // …(K)
       }

       @Bean
       @StepScope
       protected ItemProcessor<Sample, Sample> sampleProcessor(){
           return new SampleProcessor();                                         // …(L)
       }

       @Bean
       @StepScope
       protected ItemWriter<Sample> sampleWriter(){
           return new SampleWriter();                                            // …(M)
       }

       @Bean
       protected Tasklet sampleTasklet(){
           return new SampleTasklet();                                           // …(N)
       }

       @Bean
       protected JobExecutionListener jobExecutionListener(){
           return new SampleListener();                                          // …(O)
       }

       @Bean
       public TaskExecutor taskExecutor(){
           SimpleAsyncTaskExecutor simpleAsyncTaskExecutor = new SimpleAsyncTaskExecutor();
           simpleAsyncTaskExecutor.setConcurrencyLimit(10);
           return simpleAsyncTaskExecutor;                                       // …(P)
       }

       @Bean
       @StepScope
       @Value("#{jobExecutionContext['paramBySampleTasklet']}")
       public Partitioner partitioner(String paramBySampleTasklet){
           return new SamplePartitioner(paramBySampleTasklet);                   // …(Q)
       }

       @Override
       @Autowired
       public void setDataSource(@Qualifier("batchDataSource") DataSource dataSource){
           super.setDataSource(dataSource);                                      // …(R)
       }
   }

|br|

設定クラスコードの説明は以下の通りです。

|br|

.. list-table:: SpringBatchのサンプル設定クラスコードの説明
   :widths: 1, 19

   * - 項番
     - 説明

   * - (A)
     - BatchInfraConfig設定クラスをインポートします。

   * - (B)
     - @EnableBatchProcessingを付与することで、Batch設定クラスとして扱われます。この設定によりJobRepositoryといったSpringBatchの主要なコンポーネントが自動で構築されていくようになります。

   * - (C)
     - 今回のサンプルではリスタートや実行結果の管理のために作られるJobRepository用のDataSourceをインメモリDBへ置き換えるために使用します。DefaultBatchConfigurerを継承することにより、当設定クラス内でデータソースを置き換えるようオーバーライドします(Rを参照)。

   * - (D)
     - ジョブ内でステップ処理フローを定義するためにJobBuildFactoryをインジェクションします。

   * - (E)
     - ステップ内で実行する処理を定義するためにStepBuildFactoryをインジェクションします。

   * - (F)
     - ジョブのステップ処理フローを定義します。ここでは、"job"というジョブ名に、リスナーとタスクレットを実行する"step1"、Readerおよびパーティショナーを組み込んだProcessor、Writerを実行する"step2"を定義しています。

   * - (G)
     - ステップ処理を定義します。ここでは、"step1"というstep名に、SampleTaskletクラスを定義しています。

   * - (H)
     - ステップ処理を定義します。ここでは、"step2"というstep名に、IOモデルクラス、チャンクサイズ、Reader、Processor、Writerクラスを定義しています。

   * - (I)
     - "step2"をパーティション化して実行するステップを定義します。ここでは、Partitionerクラスおよび、各ステップを実行するTaskExecutorを定義しています。

   * - (J)
     - Step間で共有するパラメータを設定します。ここではJobExcutionContext内に"paramBySampleTasklet"というパラメータをSampleTaskletクラスで設定し、FlatFileItemReaderの実行引数として受け取る形で実装しています。

   * - (K)
     - (H)で設定した通り、SpringBatchから提供されているFlatFileItemReaderで型パラメータとして、モデルクラスとなるSampleクラスを指定してStepScopeで定義します。ここで実装している内容は次回改めて解説を行います。

   * - (L)
     - (H)で設定した通り、ProcessorとしてSampleProcessorクラスをBean定義します。

   * - (M)
     - (H)で設定した通り、WriterとしてSampleWriterクラスをBean定義します。

   * - (N)
     - (G)で設定した通り、TaskletとしてSampleTaskletクラスをBean定義します。

   * - (O)
     - (F)で設定した通り、リスナーとしてSampleListenerクラスをBean定義します。

   * - (P)
     - (I)で設定した通り、パーティショナーを使って並列実行を行う場合のTaskExecutorクラスとして、SimpleAsyncTaskExecutorをBean定義します。

   * - (Q)
     - (I)で設定した通り、Partitionerクラスとして、SamplePartitionerクラスをBean定義します。JobExcutionContext内に"paramBySampleTasklet"というパラメータ(パーティションキーの元となる読み込むファイルのパス)をコンストラクタインジェクションしてBean生成します。

   * - (R)
     - (C)で設定した通り、リスタートや実行結果の管理のために作られるJobRepository用のDataSourceをインメモリDBへ置き換える形で設定します。

|br|

また、BatchInfraConfigクラスは複数のジョブ設定クラスから使用される内容を定義します。ここでは、JobRepositoryのためのデータソースをインメモリDBとしてH2を定義します。

|br|

.. sourcecode:: java

   package org.debugroom.mynavi.sample.aws.sqs.config;

   import javax.sql.DataSource;

   import org.springframework.context.annotation.Bean;
   import org.springframework.context.annotation.Configuration;
   import org.springframework.jdbc.datasource.embedded.EmbeddedDatabaseBuilder;
   import org.springframework.jdbc.datasource.embedded.EmbeddedDatabaseType;

   @Configuration
   public class BatchInfraConfig {

       @Bean(name="batchDataSource")
       public DataSource jobRepositoryEmbeddedDataSource(){
           return new EmbeddedDatabaseBuilder()
             .setType(EmbeddedDatabaseType.H2)
             .addScript("classpath:/org/springframework/batch/core/schema-h2.sql")
             .build();
       }
   }

|br|

これで、Batchアプリケーションの設定クラスが作成できました。次回は、TaskletやProcessorなどのバッチ処理クラスを実装してみます。

|br|

著者紹介
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

川畑 光平(KAWABATA Kohei)

.. figure:: img/aws-lambda-and-api-gateway/pic_image01.jpg


某システムインテグレータにて、金融機関システム業務アプリケーション開発・システム基盤担当を経て、現在はソフトウェア開発自動化関連の研究開発・推進に従事。

Red Hat Certified Engineer、Pivotal Certified Spring Professional、AWS Certified Solutions Architect Professional等の資格を持ち、アプリケーション基盤・クラウドなど様々な開発プロジェクト支援にも携わる。

`2019 APN AWS Top Engineers & Ambassadors <https://aws.amazon.com/jp/blogs/psa/japan-apn-ambassador-2019/>`_ 選出。

本連載記事の内容に対するご意見・ご質問は `Facebook <https://www.facebook.com/kohei.kawabata.5>`_ まで。
