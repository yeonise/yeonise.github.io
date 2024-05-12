---
layout: post
title: "Spring Batch"
subtitle: Spring Batch 5.0 체험기
date: 2024-05-09 09:00:00 +0900
tags: [server, spring]
---

<br />

<img src="/assets/screenshot/0509-1.png" />

# 1. application.yml 설정

``` yaml
spring:
  batch:
    job:
      name: ${job.name:NONE}  # job.name이 있으면 job.name 값을 할당하고, 없으면 NONE을 할당한다.
```

위 설정을 이용하면 Spring Batch가 실행될 때, job.name의 값과 일치하는 Job만 실행할 수 있다.  
`NONE` 이 할당되면 어떠한 배치도 실행되지 않는다.  
`--job.name=helloWorldJob` 과 같이 Program arguments에 추가하면 된다.

<br />

# 2. Job 생성

Job은 Step으로 구성된 작업 단위이다.  
Step은 실제 Batch 작업을 수행하는 역할을 한다.  
Step을 Bean으로 등록하고 이를 Job을 Bean으로 등록할 때 사용한다.  
Step 하위에는 ItemReader, ItemProcessor, ItemWriter가 존재한다.  
그러나 읽기 또는 쓰기 작업이 없는 경우, Tasklet을 사용하여 단순한 Batch를 생성할 수 있다.

<br />

**helloWorldJob 구현 예시**

(참고로 강의에서 사용한 `JobBuilderFactory` 는 `deprecated` 되었다.)  

``` java
@Configuration
public class HelloWorldJobConfig {

    @Bean
    public Job helloWorldJob(JobRepository jobRepository, Step helloWorldStep) {
      return new JobBuilder("helloWorldJob", jobRepository)
        .incrementer(new RunIdIncrementer())
        .start(helloWorldStep)
        .build();
    }
      
    @Bean
    @JobScope
    public Step helloWorldStep(JobRepository jobRepository, PlatformTransactionManager platformTransactionManager, Tasklet helloWorldTasklet) {
      return new StepBuilder("helloWorldStep", jobRepository)
        .tasklet(helloWorldTasklet, platformTransactionManager)
        .build();
    }

    @Bean
    @StepScope
    public Tasklet helloWorldTasklet() {
      return new Tasklet() {
        @Override
        public RepeatStatus execute(StepContribution contribution, ChunkContext chunkContext) throws Exception {
          System.out.println("Hello World Spring batch");

          return RepeatStatus.FINISHED;
        }
      };
    }

}
```

> 여기서 `incrementer()` 는 Job 실행 시마다 고유한 JobInstanceId를 생성하여 동일한 Job이 한 번만 실행되도록 보장하는 역할을 한다.  

<br />

애플리케이션을 실행하면 다음과 같은 결과를 확인할 수 있다. ( `--job.name=helloWorldJob` )  

<img src="/assets/screenshot/0509-2.png" />

<br />

# 3. 파라미터 받기 및 검증

Program arguments를 통해 파라미터를 전달하고 받은 파라미터를 검증할 수 있다.  

<br />

**validatedParamJob 구현 예시**

``` java
@Configuration
public class ValidatedParamJobConfig {

  @Bean
  public Job validatedParamJob(JobRepository jobRepository, Step validatedParamStep) {
    return new JobBuilder("validatedParamJob", jobRepository)
      .incrementer(new RunIdIncrementer())
      .validator(new FileParamValidator())  // 추가한 부분
      .start(validatedParamStep)
      .build();
  }

  @Bean
  @JobScope
  public Step validatedParamStep(JobRepository jobRepository, PlatformTransactionManager platformTransactionManager, Tasklet validatedParamTasklet) {
    return new StepBuilder("validatedParamStep", jobRepository)
      .tasklet(validatedParamTasklet, platformTransactionManager)
      .build();
  }

  @Bean
  @StepScope
  public Tasklet validatedParamTasklet(@Value("#{jobParameters[fileName]}") String fileName) {
    return new Tasklet() {
      @Override
      public RepeatStatus execute(StepContribution contribution, ChunkContext chunkContext) throws Exception {
        System.out.println(fileName);
        System.out.println("Validated Param Tasklet");

        return RepeatStatus.FINISHED;
      }
    };
  }

}
```

> `#{jobParameters[fileName]}` 표현식을 이용하여 파라미터 값을 가져올 수 있다.

<br />

**FileParamValidator 구현 예시**

``` java
public class FileParamValidator implements JobParametersValidator {

  @Override
  public void validate(JobParameters parameters) throws JobParametersInvalidException {
    String fileName = parameters.getString("fileName");

    if (!StringUtils.endsWithIgnoreCase(fileName, "csv")) {
      throw  new JobParametersInvalidException("This is not csv file.");
    }
  }

}
```

<br />

애플리케이션을 실행하면 다음과 같은 결과를 확인할 수 있다. ( `--job.name=validatedParamJob -fileName=test.csv` )  

검증을 통과한 경우 파일 이름이 정상적으로 출력된다.

<img src="/assets/screenshot/0509-3.png" />

검증을 통과하지 못한 경우에는 예외가 발생한다.

<img src="/assets/screenshot/0509-4.png" />

<br />

# 4. 리스너 구현

Job이 실행되기 전과 후에 함수를 호출하거나 로그를 쌓는 작업을 수행할 수 있다.  

<br />

**jobListenerJob 구현 예시**

```java
@Configuration
public class JobListenerConfig {

  @Bean
  public Job jobListenerJob(JobRepository jobRepository, Step jobListenerStep) {
    return new JobBuilder("jobListenerJob", jobRepository)
      .incrementer(new RunIdIncrementer())
      .listener(new JobLoggerListener())  // 추가한 부분
      .start(jobListenerStep)
      .build();
  }

  @Bean
  @JobScope
  public Step jobListenerStep(JobRepository jobRepository, PlatformTransactionManager platformTransactionManager, Tasklet jobListenerTasklet) {
    return new StepBuilder("jobListenerStep", jobRepository)
      .tasklet(jobListenerTasklet, platformTransactionManager)
      .build();
  }

  @Bean
  @StepScope
  public Tasklet jobListenerTasklet() {
    return new Tasklet() {
      @Override
      public RepeatStatus execute(StepContribution contribution, ChunkContext chunkContext) throws Exception {
        System.out.println("This is Job Listener Tasklet");

        return RepeatStatus.FINISHED;
      }
    };
  }

}
```

<br />

**JobLoggerListener 구현 예시**

``` java
@Slf4j
public class JobLoggerListener implements JobExecutionListener {

  private static String BEFORE_MESSAGE = "{} is Running";
  private static String AFTER_MESSAGE = "{} is Done. (Status: {})";
  private static String JOB_FAILURE_MESSAGE = "{} is FAILED.";

  @Override
  public void beforeJob(JobExecution jobExecution) {
    log.info(BEFORE_MESSAGE, jobExecution.getJobInstance().getJobName());
  }

  @Override
  public void afterJob(JobExecution jobExecution) {
    log.info(AFTER_MESSAGE, jobExecution.getJobInstance().getJobName(), jobExecution.getStatus());

    if (jobExecution.getStatus() == BatchStatus.FAILED) {
      // send email...

      log.info(JOB_FAILURE_MESSAGE, jobExecution.getJobInstance().getJobName());
    }
  }

}
```

<br />

애플리케이션을 실행하면 다음과 같이 로그가 출력되는 것을 확인할 수 있다. ( `--job.name=jobListenerJob` )  

<img src="/assets/screenshot/0509-5.png" />

`jobListenerTasklet()` 에서 예외가 발생하도록 코드를 수정한 뒤, 애플리케이션을 실행하면 `afterJob()` 에서 작성한 `JOB_FAILURE_MESSAGE` 가 추가적으로 출력된다.

<img src="/assets/screenshot/0509-6.png" />

<br />

# 5. DB 데이터 읽고 쓰기

다음은 주문 테이블(orders)에 있는 데이터를 읽어서 정산 테이블(accounts)로 이관하는 예제이다.

> trMigration**Job** 하위에 trMigration**Step**이 존재한다.  
> trMigration**Step** 하위에 trOrders**Reader**, trOrder**Processor**, trOrder**Writer**이 존재한다.

(참고로 강의에서는 Accounts의 `@Id` 에 `@GeneratedValue(strategy = GenerationType.IDENTITY)` 를 적용했는데, trOrderWriter가 insert 작업을 총 데이터의 절반만 수행하는 문제가 발생했다. 제거하니 정상적으로 동작한다.)

<br />

**trMigrationJob 구현 예시**

``` java
@Configuration
@RequiredArgsConstructor
public class TrMigrationConfig {

  private final OrdersRepository ordersRepository;
  private final AccountsRepository accountsRepository;

  @Bean
  public Job trMigrationJob(JobRepository jobRepository, Step trMigrationStep) {
    return new JobBuilder("trMigrationJob", jobRepository)
      .incrementer(new RunIdIncrementer())
      .start(trMigrationStep)
      .build();
  }

  @Bean
  @JobScope
  public Step trMigrationStep(JobRepository jobRepository, PlatformTransactionManager platformTransactionManager,
    ItemReader<Orders> trOrdersReader,
    ItemProcessor<Orders, Accounts> trOrderProcessor,
    ItemWriter<Accounts> trOrderWriter) {

    return new StepBuilder("trMigrationStep", jobRepository)
      .<Orders, Accounts>chunk(5, platformTransactionManager)
      .reader(trOrdersReader)
      .processor(trOrderProcessor)
      .writer(trOrderWriter)  // chunk size 충족 시 write 작업을 수행한다.
      .build();
  }

  @Bean
  @StepScope
  public RepositoryItemReader<Orders> trOrdersReader() {
    return new RepositoryItemReaderBuilder<Orders>()
      .name("trOrdersReader")
      .repository(ordersRepository)
      .methodName("findAll")
      .pageSize(5)  // 한 번에 읽는 데이터의 개수를 설정한다.
      .arguments(List.of())
      .sorts(Collections.singletonMap("id", Sort.Direction.DESC))
      .build();
  }

  @Bean
  @StepScope
  public ItemProcessor<Orders, Accounts> trOrderProcessor() {
    return new ItemProcessor<Orders, Accounts>() {
      @Override
      public Accounts process(Orders item) throws Exception {
        return new Accounts(item); 
      }
    };
  }

  @Bean
  @StepScope
  public ItemWriter<Accounts> trOrderWriter() {
    return new RepositoryItemWriterBuilder<Accounts>()
      .repository(accountsRepository)
      .methodName("save")
      .build();
  }

}
```   

<br />

애플리케이션을 실행하면 다음과 같이 쿼리가 출력되는 것을 확인할 수 있다. ( `--job.name=trMigrationJob` )  

<img src="/assets/screenshot/0509-7.png" />

비어있던 accounts 테이블로 orders의 데이터가 정상적으로 이관되었다.  

<img src="/assets/screenshot/0509-8.png" />

<br />

# 6. 파일 읽고 쓰기

`player.csv` 파일을 읽어서 `player-output.csv` 파일을 쓰는 예제이다.  

`player.csv` 파일의 내용은 다음과 같다.  

```
ID,lastName,firstName,position,birthYear,debutYear
AbduKa00,Abdul-Jabbar,Karim,rb,1974,1996
AbduRa00,Abdullah,Rabih,rb,1975,1999
AberWa00,Abercrombie,Walter,rb,1959,1982
AbraDa00,Abramowicz,Danny,wr,1945,1967
AdamBo00,Adams,Bob,te,1946,1969
AdamCh00,Adams,Charlie,wr,1979,2003
```

<br />

파일을 읽을 때 사용할 FieldSetMapper를 구현한다.  

**FieldSetMapper 구현 예시**

``` java
public class PlayerFieldSetMapper implements FieldSetMapper<Player> {

  public Player mapFieldSet(FieldSet fieldSet) {
    Player player = new Player();

    player.setID(fieldSet.readString(0));  // ID
    player.setLastName(fieldSet.readString(1));  // lastName
    player.setFirstName(fieldSet.readString(2));  // firstName
    player.setPosition(fieldSet.readString(3));  // position
    player.setBirthYear(fieldSet.readInt(4));  // birthYear
    player.setDebutYear(fieldSet.readInt(5));  // debutYear

    return player;
  }

}
```

<br />

FlatFileItemReader와 FlatFileItemWriter를 이용하여 Step을 구현하고 Job을 등록한다.  

**fileReadWriteJob 구현 예시**

``` java
@Configuration
public class FileReadWriteConfig {

  @Bean
  public Job fileReadWriteJob(JobRepository jobRepository, Step fileReadWriteStep) {
    return new JobBuilder("fileReadWriteJob", jobRepository)
      .incrementer(new RunIdIncrementer())
      .start(fileReadWriteStep)
      .build();
  }

  @Bean
  @JobScope
  public Step fileReadWriteStep(JobRepository jobRepository, PlatformTransactionManager transactionManager,
    ItemReader<Player> playerItemReader,
    ItemProcessor<Player, PlayerYears> playerItemProcessor,
    ItemWriter<PlayerYears> playerItemWriter) {

    return new StepBuilder("fileReadWriteStep", jobRepository)
      .<Player, PlayerYears>chunk(5, transactionManager)
      .reader(playerItemReader)
      .processor(playerItemProcessor)
      .writer(playerItemWriter)
      .build();
  }

  @Bean
  @StepScope
  public FlatFileItemReader<Player> playerItemReader() {
    return new FlatFileItemReaderBuilder<Player>()
      .name("playerItemReader")
      .resource(new FileSystemResource("player.csv"))  // 읽어올 파일 설정
      .lineTokenizer(new DelimitedLineTokenizer())  // comma로 구분
      .fieldSetMapper(new PlayerFieldSetMapper())  // Mapper 적용
      .linesToSkip(1)
      .build();
  }

  @Bean
  @StepScope
  public ItemProcessor<Player, PlayerYears> playerItemProcessor() {
    return new ItemProcessor<Player, PlayerYears>() {
      @Override
      public PlayerYears process(Player item) throws Exception {
        return new PlayerYears(item);  // yearsExperience 필드가 추가된 객체로 가공
      }
    };
  }

  @Bean
  @StepScope
  public FlatFileItemWriter<PlayerYears> playerItemWriter() {
    BeanWrapperFieldExtractor<PlayerYears> fieldExtractor = new BeanWrapperFieldExtractor<>();  // 어떤 필드를 사용할 것인가
    fieldExtractor.setNames(new String[]{"ID", "lastName", "position", "yearsExperience"});
    fieldExtractor.afterPropertiesSet();

    DelimitedLineAggregator<PlayerYears> lineAggregator = new DelimitedLineAggregator<>();
    lineAggregator.setDelimiter(",");  // 어떤 구분자를 사용할 것인가
    lineAggregator.setFieldExtractor(fieldExtractor);  // 어떤 필드를 사용할 것인가

    FileSystemResource outputResource = new FileSystemResource("player-output.csv");  // 결과물

    return new FlatFileItemWriterBuilder<PlayerYears>()
      .name("playerItemWriter")
      .resource(outputResource)
      .lineAggregator(lineAggregator)
      .build();
  }

}
```

<br />

애플리케이션을 실행하면 `player-output.csv` 파일이 생성된다. ( `--job.name=fileReadWriteJob` )  

<img src="/assets/screenshot/0512-1.png" />

<br />

# 7. 여러 개의 Step을 이용한 실행 상태에 따른 분기 처리 

지금까지는 하나의 Job에 하나의 Step을 사용해서 구현했다.  
하나의 Job에 여러 개의 Step을 사용하는 방법은 다음과 같다.  

<br />

**multiStepJob 구현 예시**

``` java
@Configuration
public class MultiStepJobConfig {

  @Bean
  public Job multiStepJob(JobRepository jobRepository, Step step1, Step step2, Step step3) {
    return new JobBuilder("multiStepJob", jobRepository)
      .incrementer(new RunIdIncrementer())
      .start(step1)  // 첫 번째 Step 시작
      .next(step2)  // 이어서 두 번째 Step 실행
      .next(step3)  // 마지막으로 세 번째 Step 실행
      .build();
  }

  @Bean
  @JobScope
  public Step step1(JobRepository jobRepository, PlatformTransactionManager transactionManager) {

    return new StepBuilder("step1", jobRepository)
      .tasklet(((contribution, chunkContext) -> {
        System.out.println("Step 1");

        return RepeatStatus.FINISHED;
      }), transactionManager)
      .build();
  }

  @Bean
  @JobScope
  public Step step2(JobRepository jobRepository, PlatformTransactionManager transactionManager) {

    return new StepBuilder("step2", jobRepository)
      .tasklet(((contribution, chunkContext) -> {  // chunk는 transaction의 단위이다.
        System.out.println("Step 2");

        ExecutionContext executionContext = chunkContext
          .getStepContext()
          .getStepExecution()
          .getJobExecution()
          .getExecutionContext();

        executionContext.put("for next", "hello!");  

        return RepeatStatus.FINISHED;
      }), transactionManager)
      .build();
  }

  @Bean
  @JobScope
  public Step step3(JobRepository jobRepository, PlatformTransactionManager transactionManager) {

    return new StepBuilder("step3", jobRepository)
      .tasklet(((contribution, chunkContext) -> {
        System.out.println("Step 3");

        ExecutionContext executionContext = chunkContext
          .getStepContext()
          .getStepExecution()
          .getJobExecution()
          .getExecutionContext();

        System.out.println(executionContext.get("for next"));  // Step 2에서 설정한 context를 Step 3에서 사용할 수 있다.

        return RepeatStatus.FINISHED;
      }), transactionManager)
      .build();
  }

}
```

<br />

애플리케이션을 시작하면 step1, step2, step3 순서대로 실행되며, step2에서 설정한 `hello!` 를 step3 단계에서 가져와서 출력하는 것을 확인할 수 있다. ( `--job.name=multiStepJob` )  

<img src="/assets/screenshot/0512-4.png" />

<br />

또한 여러 개의 Step을 사용하여 분기 처리도 할 수 있다.  

**conditionalStepJob 구현 예시**

``` java
@Configuration
public class ConditionalStepJobConfig {

  @Bean
  public Job conditionalStepJob(JobRepository jobRepository,
    Step conditionalStartStep,
    Step allStep,
    Step failedStep,
    Step completedStep) {

    return new JobBuilder("conditionalStepJob", jobRepository)
      .incrementer(new RunIdIncrementer())
      .start(conditionalStartStep)
      .on("FAILED").to(failedStep)  // 실패한 경우
      .from(conditionalStartStep)
      .on("COMPLETED").to(completedStep)  // 성공한 경우
      .from(conditionalStartStep)
      .on("*").to(allStep)  // 그 외의 경우
      .end()
      .build();
  }

  @Bean
  @JobScope
  public Step conditionalStartStep(JobRepository jobRepository, PlatformTransactionManager transactionManager) {

    return new StepBuilder("conditionalStartStep", jobRepository)
      .tasklet(new Tasklet() {
        @Override
        public RepeatStatus execute(StepContribution contribution, ChunkContext chunkContext) throws Exception {
          System.out.println("start conditional step");

          // throw new RuntimeException();
          return RepeatStatus.FINISHED;
        }
      }, transactionManager)
      .build();
  }

  @Bean
  @JobScope
  public Step allStep(JobRepository jobRepository, PlatformTransactionManager transactionManager) {

    return new StepBuilder("allStep", jobRepository)
      .tasklet(new Tasklet() {
        @Override
        public RepeatStatus execute(StepContribution contribution, ChunkContext chunkContext) throws Exception {
          System.out.println("all Step (default)");

          return RepeatStatus.FINISHED;
        }
      }, transactionManager)
      .build();
  }

  @Bean
  @JobScope
  public Step failedStep(JobRepository jobRepository, PlatformTransactionManager transactionManager) {

    return new StepBuilder("failed Step", jobRepository)
      .tasklet(new Tasklet() {
        @Override
        public RepeatStatus execute(StepContribution contribution, ChunkContext chunkContext) throws Exception {
          System.out.println("failed Step");

          return RepeatStatus.FINISHED;
        }
      }, transactionManager)
      .build();
  }

  @Bean
  @JobScope
  public Step completedStep(JobRepository jobRepository, PlatformTransactionManager transactionManager) {

    return new StepBuilder("completedStep", jobRepository)
      .tasklet(new Tasklet() {
        @Override
        public RepeatStatus execute(StepContribution contribution, ChunkContext chunkContext) throws Exception {
          System.out.println("completed Step");

          return RepeatStatus.FINISHED;
        }
      }, transactionManager)
      .build();
  }

}
```

<br />

애플리케이션을 정상적으로 실행하면 completedStep이 실행되는 것을 확인할 수 있다. ( `--job.name=conditionalStepJob` )  

<img src="/assets/screenshot/0512-2.png" />

예외가 발생하여 conditionalStartStep이 실패한 경우 failedStep이 실행된다. 

<img src="/assets/screenshot/0512-3.png" />

<br />

# 8. 테스트 코드 작성

이제 간단한 테스트 코드를 작성해보자.  
Batch 작업을 위한 설정 파일과 Job을 등록한 설정 파일을 사용하여 애플리케이션 컨텍스트를 구성하고 하나의 Job을 테스트하는 방법이다.  

<br />

test 패키지 내부에 Batch 작업을 위한 설정 파일을 생성한다.

``` java
@Configuration
@EnableAutoConfiguration
@EnableBatchProcessing
public class SpringBatchTestConfig {
}
```

<br />

DB 데이터 이관 작업을 수행하는 trMigrationJob을 테스트하는 코드이다.  
JobLauncherTestUtils을 이용하여 trMigrationJob을 `launch()` 할 수 있다.

``` java
@SpringBootTest(classes = {SpringBatchTestConfig.class, TrMigrationConfig.class})
@SpringBatchTest
class TrMigrationConfigTest {

  @Autowired
  private JobLauncherTestUtils jobLauncherTestUtils;

  @Autowired
  private OrdersRepository ordersRepository;

  @Autowired
  private AccountsRepository accountsRepository;

  @BeforeEach
  void clean() {
    ordersRepository.deleteAllInBatch();
    accountsRepository.deleteAllInBatch();
  }

  @Test
  @DisplayName("orders 테이블에 데이터가 없으므로 accounts 테이블에 이관된 데이터는 없다")
  void success_no_data() throws Exception {
    // when
    JobExecution jobExecution = jobLauncherTestUtils.launchJob();

    // then
    assertThat(jobExecution.getExitStatus()).isEqualTo(ExitStatus.COMPLETED);
    assertThat(accountsRepository.count()).isEqualTo(0);
  }

  @Test
  @DisplayName("orders 테이블의 모든 데이터가 accounts 테이블로 이관된다")
  void success_has_data() throws Exception {
    // given
    Orders kakaoGift = new Orders(null, "kakao gift", 25000, new Date());
    Orders naverGift = new Orders(null, "naver gift", 15000, new Date());

    ordersRepository.save(kakaoGift);
    ordersRepository.save(naverGift);

    // when
    JobExecution jobExecution = jobLauncherTestUtils.launchJob();

    // then
    assertThat(jobExecution.getExitStatus()).isEqualTo(ExitStatus.COMPLETED);
    assertThat(accountsRepository.count()).isEqualTo(2);
  }

}
```

<br />

테스트를 실행하면 통과되는 것을 확인할 수 있다.

<img src="/assets/screenshot/0512-5.png" />

<br />

# 9. 스프링 스케줄링을 이용한 배치 작업 실행

helloWorldJob을 스케줄링을 이용해서 1분마다 실행하는 예제이다.

<br />

우선 `application.yml` 에 `enabled: false` 설정을 추가한다.  

``` yaml
spring:
  batch:
    job:
      name: ${job.name:NONE}  # job.name이 있으면 job.name 값을 할당하고, 없으면 NONE을 할당한다.
      enabled: false  # job.name이 파라미터로 넘어와도 실행하지 않는다.
```

<br />

기존의 HelloWorldJobConfig에 `@EnableScheduling` 을 추가한다.

``` java
@Configuration
@EnableScheduling  // 추가
public class HelloWorldJobConfig {

  @Bean
  public Job helloWorldJob(JobRepository jobRepository, Step helloWorldStep) {
    return new JobBuilder("helloWorldJob", jobRepository)
      .incrementer(new RunIdIncrementer())
      .start(helloWorldStep)
      .build();
  }
  ...
```

<br />

스케줄링을 위한 컴포넌트를 생성한다.  

**SimpleScheduler 구현 예시**
``` java
@Component
public class SimpleScheduler {

  @Autowired
  private JobLauncher jobLauncher;

  @Autowired
  private Job helloWorldJob;

  @Scheduled(cron = "0 */1 * * * *")  // second(0-59), minute(0-59), hour(0-23), day of the month(1-31), month(1-12), day of the week(0-7)
  public void helloWorldJobRun() throws
    JobInstanceAlreadyCompleteException,
    JobExecutionAlreadyRunningException,
    JobParametersInvalidException,
    JobRestartException {

    JobParameters jobParameters = new JobParametersBuilder()
      .addLong("requestTime", System.currentTimeMillis())  // 1분마다 실행되는 각각의 Job을 구별하기 위한 파라미터
      .toJobParameters();

    jobLauncher.run(helloWorldJob, jobParameters);
  }

}
```

<br />

애플리케이션을 실행하면 1분마다 "Hello World Spring batch"가 출력된다.

<img src="/assets/screenshot/0512-6.png" />

<br />

> - 수강한 강의는 Spring Boot 2.6.7, Spring Batch 4.x 버전을 사용했기 때문에 Spring Boot 3.x 이상, Spring Batch 5.x 이상의 프로젝트에서 제대로 동작하지 않는다.  
> - Spring Batch 5.x 버전으로 업데이트 하면서 기존 방식에서 변경된 부분이 많기 때문에 공식 문서를 참고하는 것이 가장 정확할 것 같다.  
> - 그리고 Spring Batch는 수백만 개의 대용량 데이터 일괄 처리 작업에 쓰인다는데 토이 프로젝트에서 적용할 일은 거의 없지 않을까 싶다.

<br />

#### reference

[예제로 배우는 핵심 Spring Batch](https://www.inflearn.com/course/%EC%98%88%EC%A0%9C%EB%A1%9C-%EB%B0%B0%EC%9A%B0%EB%8A%94-%ED%95%B5%EC%8B%AC-%EC%8A%A4%ED%94%84%EB%A7%81-%EB%B0%B0%EC%B9%98/dashboard)  
[Spring-Batch 5.0 Migration Guide](https://github.com/spring-projects/spring-batch/wiki/Spring-Batch-5.0-Migration-Guide)  
[Spring Batch 가이드 - Spring Batch Job Flow](https://jojoldu.tistory.com/328)  
