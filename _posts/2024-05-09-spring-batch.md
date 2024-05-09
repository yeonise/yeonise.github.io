---
layout: post
title: "Spring Batch"
subtitle: 
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
    ItemReader<Orders> trOrdersReader, ItemProcessor<Orders, Accounts> trOrderProcessor, ItemWriter<Accounts> trOrderWriter) {
    return new StepBuilder("trMigrationStep", jobRepository)
      .<Orders, Accounts>chunk(5, platformTransactionManager)
      .reader(trOrdersReader)
      .processor(trOrderProcessor)
      .writer(trOrderWriter)  // chunk size 충족 시 write 수행
      .build();
  }

  @Bean
  @StepScope
  public RepositoryItemReader<Orders> trOrdersReader() {
    return new RepositoryItemReaderBuilder<Orders>()
      .name("trOrdersReader")
      .repository(ordersRepository)
      .methodName("findAll")
      .pageSize(5)  // 읽는 데이터의 개수
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
        return new Accounts(item);  // 읽은 데이터를 처리
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

데이터가 비어있던 accounts 테이블로 정상적으로 이관되었다.  

<img src="/assets/screenshot/0509-8.png" />

<br />

<br />

# 6. 파일 읽고 쓰기

<br />

# 7. 여러 개의 Step을 이용한 실행 상태에 따른 분기 처리 

<br />

# 8. 테스트 코드 작성

<br />

# 9. 스프링 스케줄링을 이용한 배치 작업 실행

<br />

#### reference

[예제로 배우는 핵심 Spring Batch](https://www.inflearn.com/course/%EC%98%88%EC%A0%9C%EB%A1%9C-%EB%B0%B0%EC%9A%B0%EB%8A%94-%ED%95%B5%EC%8B%AC-%EC%8A%A4%ED%94%84%EB%A7%81-%EB%B0%B0%EC%B9%98/dashboard)  
[Spring-Batch 5.0 Migration Guide](https://github.com/spring-projects/spring-batch/wiki/Spring-Batch-5.0-Migration-Guide)  
[Spring Batch 가이드 - Spring Batch Job Flow](https://jojoldu.tistory.com/328)  
