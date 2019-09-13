.. include:: ../module.txt

.. _section-cloud-native-sqs-label-4:

AWSで作るクラウドネイティブアプリケーションの基本
========================================================================================

.. _section-cloud-native-sqs-4th-label:

Amazon SQSを使ったSpringアプリケーション(4)
----------------------------------------------------------------------------------------

|br|

本連載では、以下のステップで解説を進めていきます。

|br|

#. Amazon SQSの概要とアプリケーション処理パターン、SQSキューの作成
#. Spring Cloud AWSを用いたSQSProducerアプリケーション実装
#. Spring Batchを用いたバッチアプリケーション実装(1)
#. **Spring Batchを用いたバッチアプリケーション実装(2)**
#. Spring Cloud AWSを用いたSQSConsumerアプリケーション実装

|br|

前回はSpring Batchを使ったバッチの設定を実装しました。今回は引き続きバッチの処理を実装していきます。


.. _section-cloud-native-sqs-batch-using-spring-bacth-process-implementation-label:

SpringBatchの処理実装
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

|br|

今回作成するSpringBatchを使ったアプリケーションは以下の図のようなイメージで構成しています。

|br|

.. figure:: img/aws-sqs/spring-batch-sample.png

|br|

本連載で実際に作成するSpringBatchアプリケーションは `GitHub <https://github.com/debugroom/mynavi-sample-aws-sqs/tree/master/spring-batch>`_ 上にコミットしています。
以降に記載するソースコードでは、import文など本質的でない記述を省略している部分があるので、実行コードを作成する際は、必要に応じて適宜GitHubにあるソースコードも参照してください。

前回に引き続き、実装していくクラスを説明します。ジョブの最初のステップで実行されるSampleTaskletでは、
org.springframework.batch.core.step.tasklet.Taskletを実装して、execute()メソッド内にバッチ処理を実装します。
また、バッチ起動時に渡した引数をChunkContextを通じて、StepExecutionから取得し、ログ出力する例をサンプルとして実装しています。
加えて、後続の処理にパラメータとして、インプットファイルのパスをJobExecutionContextに設定する実装例です。

戻り値としてはorg.springframework.batch.repeat.RepeatStatusに適切なステータスコードを設定しておきます。

|br|

.. sourcecode:: java

   package org.debugroom.mynavi.sample.aws.sqs.app.batch.step;

   import org.springframework.batch.core.StepContribution;
   import org.springframework.batch.core.StepExecution;
   import org.springframework.batch.core.scope.context.ChunkContext;
   import org.springframework.batch.core.step.tasklet.Tasklet;
   import org.springframework.batch.item.ExecutionContext;
   import org.springframework.batch.repeat.RepeatStatus;

   import lombok.extern.slf4j.Slf4j;

   @Slf4j
   public class SampleTasklet implements Tasklet {

       @Override
       public RepeatStatus execute(StepContribution stepContribution,
                                ChunkContext chunkContext) throws Exception {

           StepExecution stepExecution = chunkContext.getStepContext().getStepExecution();
           String param = stepExecution.getJobParameters().getString("param");
           log.info(this.getClass().getName() + "#execute() starteds. input param : " + param);
           ExecutionContext jobExecutionContext = stepExecution.getJobExecution().getExecutionContext();
           jobExecutionContext.put("paramBySampleTasklet", "/test.txt");
           return RepeatStatus.FINISHED;
       }
   }

|br|

前回定義したジョブの処理フロー上では、続いて、ファイルの読み込みを行うFlatFileItemReaderが実行されます。ここで実行される処理は設定クラスBatchAppConfigで定義したsampleFlatFileItemReader()メソッドの処理となります。
上記のSampleTaskletでJobExecutionContextに設定したパラメータ(Readerで読み込むファイルのパス)を受け取り、ファイルの読み込み設定や、ファイルのデリミター(区切り文字)、モデルクラスマッピング設定を行います。
Springから提供されているorg.springframework.batch.item.file.FlatFileItemReaderクラスに、型パラメータとしてモデルクラスを設定するだけで単純なファイルの読み込み実装を簡略化できる(Readerクラスをわざわざ実装しなくてもよい)のが特徴です。
なお、ここでDelimiterに設定している名前"stepParam"はマッピングするモデルクラスの変数名になります。

|br|

.. sourcecode:: java

   package org.debugroom.mynavi.sample.aws.sqs.config;

   // omit

   import org.springframework.batch.item.file.FlatFileItemReader;

   @EnableBatchProcessing
   public class BatchAppConfig extends DefaultBatchConfigurer {

   // omit

       @Bean
       @StepScope
       @Value("#{jobExecutionContext['paramBySampleTasklet']}")
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

           return flatFileItemReader;
       }
    }

|br|

SpringBatchが読み込んだファイルのデータをモデルクラスにマッピングした後、後続のProcessor、Writerと実行されていく形になりますが、
これらの処理は前回設定した通り、パーティショナーを通して並列実行されます。パーティショナーを使用した処理については `TERASOLUNA バッチガイドライン Partitioning Step <https://terasoluna-batch.github.io/guideline/5.0.0.RELEASE/ja/Ch08_ParallelAndMultiple.html#Ch08_ParallelAndMultiple_Partitioning>`_ も合わせて参考にしてください。

パーティショナーでは、org.springframework.batch.core.partition.support.Partitionerを実装し、partition(int gridSize)メソッド内で、並列実行させたいスレッドの数に応じてパーティションIDを作成して、
ID名とIDの値をExecutionContext#putString()に設定し、スレッドを識別する文字列とExecutionContextをペアでMapに設定して戻り値として返却する実装を行います。

|br|

.. sourcecode:: java

   package org.debugroom.mynavi.sample.aws.sqs.app.batch.partitioner;

   // omit
   import org.springframework.batch.core.partition.support.Partitioner;
   import org.springframework.batch.item.ExecutionContext;

   @AllArgsConstructor
   @NoArgsConstructor
   @Builder
   @Data
   public class SamplePartitioner implements Partitioner {

      private String param;

      @Override
      public Map<String, ExecutionContext> partition(int gridSize) {
          Map<String, ExecutionContext> executionContextMap = new HashMap<>();
          try(InputStream inputStream = getClass().getResourceAsStream(param);
                  Reader reader = new InputStreamReader(inputStream);
                  BufferedReader bufferedReader = new BufferedReader(reader);){
              String readLine;
              int index = 0;
              while ((readLine = bufferedReader.readLine()) != null){
                  ExecutionContext executionContext = new ExecutionContext();
                  executionContext.putString("partitionId", readLine);
                  executionContextMap.put("partition" + index, executionContext);
                  index++;
              }
           }catch(IOException e){
               e.printStackTrace();
           }
           return executionContextMap;
       }
   }

|br|

後続のProcessorはorg.springframework.batch.item.ItemProcessorを実装し、型パラメータとしてIOとなるモデルクラスを指定します。
クラス内でオーバーライドしたprocess()メソッドが実行されますが、パーティショナーで作成したパーティンションキーの数に応じたスレッド数で多重非同期実行されます。
ProcessorクラスではインプットとしてマッピングされたモデルクラスSampleが入力クラスとして渡されます。ProcessorクラスでインジェクションされたStepExecutionを通じて、パーティションキーを取得し、
モデルの変数が、パーティションキーとして設定したIDと一致する場合にのみ、ログ出力処理を行うという形でサンプル処理実装しています。

|br|

.. sourcecode:: java

   package org.debugroom.mynavi.sample.aws.sqs.app.batch.step;

   // omit
   import java.util.Objects;

   import org.springframework.batch.core.StepExecution;
   import org.springframework.batch.item.ExecutionContext;
   import org.springframework.batch.item.ItemProcessor;
   import org.springframework.beans.factory.annotation.Value;

   @Slf4j
   public class SampleProcessor implements ItemProcessor<Sample, Sample> {

       @Value("#{stepExecution}")
       private StepExecution stepExecution;

       @Override
       public Sample process(Sample sample) throws Exception {
           ExecutionContext stepExecutionContext = stepExecution.getExecutionContext();
           ExecutionContext jobExecutionContext = stepExecution.getJobExecution().getExecutionContext();
           if(Objects.equals(sample.getStepParam(), stepExecutionContext.get("partitionId"))){
               log.info(this.getClass().getName()
                 + " started. sample.stepParam:" + sample.getStepParam()
                 + " stepExecution.partitionId:" + stepExecutionContext.getString("partitionId"));
           }
           return sample;
       }
   }

|br|

Processorの後続で実行されるWriterはorg.springframework.batch.item.ItemWriterを実装し、型パラメータとしてインプットクラスを指定します。
ここでも同様、パーティショナーで作成したパーティションキーの数に応じたスレッドで多重非同期実行されます。
同じくパーティションキーと一致したモデルクラスのみ実行する形でログ出力処理を実装しています。

|br|

.. sourcecode:: java

   package org.debugroom.mynavi.sample.aws.sqs.app.batch.step;

   // omit
   import java.util.Objects;

   import org.springframework.batch.core.StepExecution;
   import org.springframework.batch.item.ExecutionContext;
   import org.springframework.batch.item.ItemWriter;

   import org.springframework.beans.factory.annotation.Value;

   import lombok.extern.slf4j.Slf4j;

   @Slf4j
   public class SampleWriter implements ItemWriter<Sample> {

       @Value("#{stepExecution}")
       private StepExecution stepExecution;

       @Override
       public void write(List<? extends Sample> samples) throws Exception {
           ExecutionContext stepExecutionContext = stepExecution.getExecutionContext();
           log.info(this.getClass().getName() + " started.");
           samples.stream()
             .filter(sample -> Objects.equals(((Sample) sample).getStepParam(), stepExecutionContext.get("partitionId")))
             .forEach(sample -> {
                 log.info(this.getClass().getName() + " sample.stepParam:" + ((Sample) sample).getStepParam());
             });
           stepExecutionContext.put("status", "complete!");
       }
   }

|br|

.. note:: 並列実行した結果を元に最後に1つのファイルへ書き出すといった場合は、多重実行されたスレッドが全て完了を待ち合わせ、最終処理を行うといった形で実装する必要があります。

|br|

ジョブの終了前後で何か処理実行が必要であれば、JobExecutionListenerSupportを継承し、beforeJob()メソッドやafterJob()メソッドで適宜必要な処理を行うと良いでしょう。

|br|

.. sourcecode:: java

   package org.debugroom.mynavi.sample.aws.sqs.app.batch.listener;

   import lombok.extern.slf4j.Slf4j;
   import org.springframework.batch.core.JobExecution;
   import org.springframework.batch.core.listener.JobExecutionListenerSupport;

   @Slf4j
   public class SampleListener extends JobExecutionListenerSupport {

       @Override
       public void afterJob(JobExecution jobExecution) {
           log.info(this.getClass().getName() + "#afterJob started.");
       }

       @Override
       public void beforeJob(JobExecution jobExecution) {
           log.info(this.getClass().getName() + "#beforeJob started.");
       }

   }

|br|

実装が終わったら、早速メインクラスを実行してバッチジョブを実行させてみましょう。

|br|

.. sourcecode:: none

   // omit

   .   ____          _            __ _ _
   /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
   ( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
   \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
   '  |____| .__|_| |_|_| |_\__, | / / / /
    =========|_|==============|___/=/_/_/_/
   :: Spring Boot ::        (v2.1.7.RELEASE)

   // omit

   2019-09-11 01:58:11.973  INFO 66174 --- [           main] o.d.m.s.a.s.a.b.l.SpringBatchApplication : Starting SpringBatchApplication on kawabatakouheinoiMac.local with PID 66174 (/Users/xxxxx/debugroom/mynavi-sample-aws-sqs/spring-batch/target/classes started by kawabatakouhei in /Users/xxxxxx/debugroom/mynavi-sample-aws-sqs)
   2019-09-11 01:58:11.978  INFO 66174 --- [           main] o.d.m.s.a.s.a.b.l.SpringBatchApplication : No active profile set, falling back to default profiles: default
   2019-09-11 01:58:21.388  INFO 66174 --- [           main] o.s.j.d.e.EmbeddedDatabaseFactory        : Starting embedded database: url='jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=false', username='sa'
   2019-09-11 01:58:22.770  WARN 66174 --- [           main] o.s.b.c.c.a.DefaultBatchConfigurer       : No transaction manager was provided, using a DataSourceTransactionManager
   2019-09-11 01:58:22.823  INFO 66174 --- [           main] o.s.b.c.r.s.JobRepositoryFactoryBean     : No database type set, using meta data indicating: H2
   2019-09-11 01:58:24.657  INFO 66174 --- [           main] o.s.b.c.l.support.SimpleJobLauncher      : No TaskExecutor has been set, defaulting to synchronous executor.
   2019-09-11 01:58:24.936  WARN 66174 --- [           main] o.s.b.c.l.AbstractListenerFactoryBean    : org.springframework.batch.item.ItemWriter is an interface. The implementing class will not be queried for annotation based listener configurations. If using @StepScope on a @Bean method, be sure to return the implementing class so listener annotations can be used.
   2019-09-11 01:58:24.937  WARN 66174 --- [           main] o.s.b.c.l.AbstractListenerFactoryBean    : org.springframework.batch.item.ItemProcessor is an interface. The implementing class will not be queried for annotation based listener configurations. If using @StepScope on a @Bean method, be sure to return the implementing class so listener annotations can be used.
   2019-09-11 01:58:25.840  INFO 66174 --- [           main] o.d.m.s.a.s.a.b.l.SpringBatchApplication : Started SpringBatchApplication in 19.962 seconds (JVM running for 23.551)
   2019-09-11 01:58:25.843  INFO 66174 --- [           main] o.s.b.a.b.JobLauncherCommandLineRunner   : Running default command line with: [param=1]
   2019-09-11 01:58:26.322  INFO 66174 --- [           main] o.s.b.c.l.support.SimpleJobLauncher      : Job: [SimpleJob: [name=job]] launched with the following parameters: [{param=1}]
   2019-09-11 01:58:26.360  INFO 66174 --- [           main] o.d.m.s.a.s.a.b.listener.SampleListener  : org.debugroom.mynavi.sample.aws.sqs.app.batch.listener.SampleListener#beforeJob started.
   2019-09-11 01:58:26.397  INFO 66174 --- [           main] o.s.batch.core.job.SimpleStepHandler     : Executing step: [step1]
   2019-09-11 01:58:26.445  INFO 66174 --- [           main] o.d.m.s.a.s.a.batch.step.SampleTasklet   : org.debugroom.mynavi.sample.aws.sqs.app.batch.step.SampleTasklet#execute() starteds. input param : 1
   2019-09-11 01:58:26.468  INFO 66174 --- [           main] o.s.batch.core.job.SimpleStepHandler     : Executing step: [partitionStep]
   2019-09-11 01:58:26.871  INFO 66174 --- [cTaskExecutor-1] o.d.m.s.a.s.a.b.step.SampleProcessor     : org.debugroom.mynavi.sample.aws.sqs.app.batch.step.SampleProcessor started. sample.stepParam:test2 stepExecution.partitionId:test2
   2019-09-11 01:58:26.873  INFO 66174 --- [cTaskExecutor-4] o.d.m.s.a.s.a.b.step.SampleProcessor     : org.debugroom.mynavi.sample.aws.sqs.app.batch.step.SampleProcessor started. sample.stepParam:test3 stepExecution.partitionId:test3
   2019-09-11 01:58:26.873  INFO 66174 --- [cTaskExecutor-3] o.d.m.s.a.s.a.b.step.SampleProcessor     : org.debugroom.mynavi.sample.aws.sqs.app.batch.step.SampleProcessor started. sample.stepParam:test1 stepExecution.partitionId:test1
   2019-09-11 01:58:26.875  INFO 66174 --- [cTaskExecutor-5] o.d.m.s.a.s.a.b.step.SampleProcessor     : org.debugroom.mynavi.sample.aws.sqs.app.batch.step.SampleProcessor started. sample.stepParam:test5 stepExecution.partitionId:test5
   2019-09-11 01:58:26.875  INFO 66174 --- [cTaskExecutor-2] o.d.m.s.a.s.a.b.step.SampleProcessor     : org.debugroom.mynavi.sample.aws.sqs.app.batch.step.SampleProcessor started. sample.stepParam:test4 stepExecution.partitionId:test4
   2019-09-11 01:58:26.878  INFO 66174 --- [cTaskExecutor-5] o.d.m.s.a.s.app.batch.step.SampleWriter  : org.debugroom.mynavi.sample.aws.sqs.app.batch.step.SampleWriter started.
   2019-09-11 01:58:26.879  INFO 66174 --- [cTaskExecutor-3] o.d.m.s.a.s.app.batch.step.SampleWriter  : org.debugroom.mynavi.sample.aws.sqs.app.batch.step.SampleWriter started.
   2019-09-11 01:58:26.880  INFO 66174 --- [cTaskExecutor-4] o.d.m.s.a.s.app.batch.step.SampleWriter  : org.debugroom.mynavi.sample.aws.sqs.app.batch.step.SampleWriter started.
   2019-09-11 01:58:26.880  INFO 66174 --- [cTaskExecutor-3] o.d.m.s.a.s.app.batch.step.SampleWriter  : org.debugroom.mynavi.sample.aws.sqs.app.batch.step.SampleWriter sample.stepParam:test1
   2019-09-11 01:58:26.880  INFO 66174 --- [cTaskExecutor-4] o.d.m.s.a.s.app.batch.step.SampleWriter  : org.debugroom.mynavi.sample.aws.sqs.app.batch.step.SampleWriter sample.stepParam:test3
   2019-09-11 01:58:26.880  INFO 66174 --- [cTaskExecutor-1] o.d.m.s.a.s.app.batch.step.SampleWriter  : org.debugroom.mynavi.sample.aws.sqs.app.batch.step.SampleWriter started.
   2019-09-11 01:58:26.880  INFO 66174 --- [cTaskExecutor-1] o.d.m.s.a.s.app.batch.step.SampleWriter  : org.debugroom.mynavi.sample.aws.sqs.app.batch.step.SampleWriter sample.stepParam:test2
   2019-09-11 01:58:26.880  INFO 66174 --- [cTaskExecutor-5] o.d.m.s.a.s.app.batch.step.SampleWriter  : org.debugroom.mynavi.sample.aws.sqs.app.batch.step.SampleWriter sample.stepParam:test5
   2019-09-11 01:58:26.883  INFO 66174 --- [cTaskExecutor-2] o.d.m.s.a.s.app.batch.step.SampleWriter  : org.debugroom.mynavi.sample.aws.sqs.app.batch.step.SampleWriter started.
   2019-09-11 01:58:26.883  INFO 66174 --- [cTaskExecutor-2] o.d.m.s.a.s.app.batch.step.SampleWriter  : org.debugroom.mynavi.sample.aws.sqs.app.batch.step.SampleWriter sample.stepParam:test4
   2019-09-11 01:58:26.903  INFO 66174 --- [           main] o.d.m.s.a.s.a.b.listener.SampleListener  : org.debugroom.mynavi.sample.aws.sqs.app.batch.listener.SampleListener#afterJob started.
   2019-09-11 01:58:26.906  INFO 66174 --- [           main] o.s.b.c.l.support.SimpleJobLauncher      : Job: [SimpleJob: [name=job]] completed with the following parameters: [{param=1}] and the following status: [COMPLETED]
   2019-09-11 01:58:26.910  INFO 66174 --- [       Thread-6] o.s.j.d.e.EmbeddedDatabaseFactory        : Shutting down embedded database: url='jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=false'

|br|

これで、SpringBootを使ってバッチジョブが実装できました。次回はこのバッチジョブを、SQSキューを受け取って実装するアプリケーションを作成してみます。

|br|

著者紹介
------------------------------------------------------------------

川畑 光平(KAWABATA Kohei)

.. figure:: img/aws-lambda-and-api-gateway/pic_image01.jpg


某システムインテグレータにて、金融機関システム業務アプリケーション開発・システム基盤担当を経て、現在はソフトウェア開発自動化関連の研究開発・推進に従事。

Red Hat Certified Engineer、Pivotal Certified Spring Professional、AWS Certified Solutions Architect Professional等の資格を持ち、アプリケーション基盤・クラウドなど様々な開発プロジェクト支援にも携わる。

`2019 APN AWS Top Engineers & Ambassadors <https://aws.amazon.com/jp/blogs/psa/japan-apn-ambassador-2019/>`_ 選出。

本連載記事の内容に対するご意見・ご質問は `Facebook <https://www.facebook.com/kohei.kawabata.5>`_ まで。
