---
title: 스케쥴러를 사용한 푸시 알림과 배치 처리 기능 구현 (이론편)
author: jemlog
date: 2023-12-06 20:55:00 +0800
categories: [Project]
tags: [spring]
pin: false
img_path: '/assets/img'
---

Spring은 어노테이션을 사용해서 손쉽게 스케쥴링을 사용하도록 도와줍니다. 하지만 @Scheduler 어노테이션만 붙여서는 실제 운영 시에 문제가 발생할 수 있습니다.
오늘은 어떤 문제들이 발생할 수 있는지 알아보고 이를 해결하는 방법들을 학습해보고자 합니다.



### ThreadPoolTaskScheduler 도입

첫번째로 발생 가능한 문제 상황을 설명하기 위해 별도의 간단한 스케쥴링 예제를 가져왔습니다.

```java
@Component
public class SchedulerService {

  @Scheduled(fixedDelay = 1000)
  public void test1() throws InterruptedException {
    log.info("Scheduling 1 start : " + Thread.currentThread().getName());
    Thread.sleep(5000);
  }
  
  @Scheduled(fixedDelay = 1000)
  public void test2() {
    log.info("Scheduling 2 start : " + Thread.currentThread().getName());
  }
}
```

위의 예제는 1초마다 스케쥴러를 호출하는 간단한 로직을 가지고 있습니다. 두 스케쥴러간 다른 점은 test1() 메서드의 경우 5초간 sleep하는 동작이 포함되어 있습니다.
이 상황에서 어플리케이션을 실행하면 어떻게 될까요? 얼핏 봐서는 test2() 스케쥴러는 1초마다 실행되고, test1()은 별도로 5초씩 쉬면서 실행될 것 같습니다. 하지만 결과는 예상과 다릅니다.

<img width="942" alt="스크린샷 2023-12-19 오후 2 51 13" src="https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/e1d8e094-e0ba-4897-af23-e62056434628">

쓰레드가 sleep 상태에 들어간 5초 동안은 2개의 스케줄러가 모두 동작을 멈춥니다. 로그가 찍힌 시간을 보면 sleep이 끝나는 5초 뒤에 다음 스케줄러가 동작하는걸 알 수 있습니다.

왜 이런 현상이 발생하는지는 로그에 기록한 쓰레드 이름을 보면 알 수 있습니다. 두 스케쥴러 모두 `scheduling`이라는 하나의 쓰레드에서 작업을 처리하고 있습니다. 혹시나 스케쥴링 메서드를 서로 다른 클래스로 분리하면 개별적으로 실행될까 싶어
실험해봤지만 결과를 똑같았습니다.

스프링 스케쥴러의 기본 동작은 하나의 쓰레드만 스케쥴링에 사용합니다. 이 방식은 동시 실행되는 스케쥴러가 많고, 스케쥴링 주기가 짧을 수록 문제가 될 것입니다. 따라서 현재 서비스에서 스케쥴러 간의 실행 쓰레드를 분리할 필요가 있습니다.
스프링은 `ThreadPoolTaskScheduler`를 통해 스케쥴링을 위한 쓰레드풀을 제공해줍니다.

```java
@Configuration
class SchedulerConfig implements SchedulingConfigurer {

    @Override
    public void configureTasks(ScheduledTaskRegistrar taskRegistrar) {
        ThreadPoolTaskScheduler threadPoolTaskScheduler = new ThreadPoolTaskScheduler();

        threadPoolTaskScheduler.setPoolSize(10);
        threadPoolTaskScheduler.setThreadGroupName("scheduler thread pool");
        threadPoolTaskScheduler.setThreadNamePrefix("scheduler-thread-");
        threadPoolTaskScheduler.initialize();

        taskRegistrar.setTaskScheduler(threadPoolTaskScheduler);
    }
}
```
ThreadPoolTaskScheduler는 위의 코드와 같이 등록할 수 있습니다. ThreadPool의 크기는 스케쥴러의 개수와 호출 주기, 그리고 메모리 상태를 종합적으로 고려해서 설정해야 한다고 생각합니다.

<img width="976" alt="스크린샷 2023-12-19 오후 2 45 26" src="https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/12b34dda-c7b2-4d9b-8adb-b3e70dcdb28e">

이번에는 test1 스케줄러의 sleep과 관계없이 test2 스케쥴러가 동작합니다. 쓰레드 이름을 잘 보면 우리가 생성한 쓰레드풀의 쓰레드가 사용되고 있는걸 알 수 있습니다.

### @Async를 사용한 비동기 처리
쓰레드 풀을 도입함으로써 스케줄링 간 독립성을 보장할 수 있었습니다. 그렇다면 이제 모든 문제가 해결됐을까요? 한가지 더 해결해야 할 문제가 있습니다.
```java
@Slf4j
@Component
public class SchedulerService {

    @Scheduled(fixedDelay = 1000)
    public void test1() throws InterruptedException {
        log.info("scheduling 1 start : " + Thread.currentThread().getName());
        Thread.sleep(3000); // 시간이 많이 소요되는 작업
        log.info("scheduling 1 end : " + Thread.currentThread().getName());
    }

    @Scheduled(fixedDelay = 1000)  // 1초마다 수행
    public void test2() {
        log.info("scheduling 2 start : " + Thread.currentThread().getName());
    }
}
```
위의 예제를 다시 살펴보겠습니다. test1 스케쥴러는 start 한 뒤에 3초를 sleep한뒤 end를 실행합니다. 하지만 저희는 test1가 1초마다 실행되기를 바랍니다. 저희 생각대로 동작하는지 살펴보겠습니다.

<img width="938" alt="스크린샷 2023-12-19 오후 2 54 50" src="https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/5ca14a2d-4f2a-4182-9851-d3f28c4ca04e">

결과를 보면 처음 test1이 시작한뒤 다음 스케쥴링은 4초뒤에 실행되는걸 알 수 있습니다. 하나의 스케쥴러 내에서의 작업의 독립성이 보장되지 않는 것입니다. 하나의 스케쥴러는 하나의 스케쥴러 쓰레드가 처리하는데 해당 쓰레드가 sleep 상태에 들어가서 발생한 현상입니다.

이를 해결하기 위해서는 스케쥴러가 비동기로 실행될 수 있도록 별도의 쓰레드풀을 할당해야 합니다.

```java
@Configuration
public class SchedulerAsyncConfig {

    @Bean(name = "schedulerTaskExecutor")
    public ThreadPoolTaskExecutor executor(){
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setQueueCapacity(100);
        executor.setMaxPoolSize(100);
        executor.setThreadNamePrefix("custom-thread");
        return executor;
    }
}
```
이번에는 `ThreadPoolTaskExecutor`를 등록해보겠습니다. 이때 쓰레드풀 설정값이 중요한데요, 하나씩 살펴보겠습니다.
- **corePoolSize** : 쓰레드풀의 크기를 나타냅니다. 해당 값이 너무 크면 유휴 쓰레드가 많아지고 너무 적으면 대기 큐로 넘어가는 요청이 많아집니다. 따라서 서비스 특성에 따라 적정한 쓰레드 개수를 선택해야 합니다.
- **queueCapacity** : 유휴 쓰레드가 없을 시 요청들이 대기하는 큐입니다.
- **maxPoolSize** : 대기 큐도 꽉 찼을때 최대로 늘릴 수 있는 쓰레드 개수입니다. 만약 maxPoolSize도 넘어가는 요청이 들어오면 예외가 발생합니다.

```java
@Slf4j
@Component
public class SchedulerService {

    @Async(value = "schedulerTestTaskExecutor") // 등록한 쓰레드풀을 상용해서 비동기 동작
    @Scheduled(fixedDelay = 1000)
    public void test1() throws InterruptedException {
        log.info("scheduling 1 start : " + Thread.currentThread().getName());
        Thread.sleep(3000);
        log.info("scheduling 1 end : " + Thread.currentThread().getName());
    }
    
    @Scheduled(fixedDelay = 1000)  // 1초마다 수행
    public void test2() {
        log.info("scheduling 2 start : " + Thread.currentThread().getName());
    }
}
```
실행하려고 하는 메서드 위에 @Async를 붙이면 비동기 처리가 가능합니다. 이때 value로 쓰레드 풀 등록시 지정한 bean name을 지정하면 해당 쓰레드풀을 사용할 수 있습니다.

![스크린샷 2023-12-15 오후 9 10 06](https://github.com/dnd-side-project/dnd-9th-1-backend/assets/82302520/141804f4-72da-4b4b-b4e3-b163f7afbb1d)

test1 스케쥴러가 3초간 sleep에 들어갔음에도 async-thread-2가 1초뒤에 다음 스케쥴러를 실행시키는걸 알 수 있습니다.
처음 sleep 상태에 들어간 async-thread-1의 작업은 비동기적으로 3초 뒤에 알아서 end라는 메세지와 함께 종료됩니다.

## 결론

지금까지 간단한 예제를 통해 ThreadPoolTaskScheduler와 ThreadPoolTaskExecutor를 스케줄러에 적용해봤습니다. 이 두가지를 적용해서 저희는 각 스케쥴러 간 독립성이 보장되는 것은 물론, 하나의 스케쥴러의 작업 간에도 독립성이 보장되도록 만들 수 있었습니다.


[nodejs]: https://nodejs.org/
[starter]: https://github.com/cotes2020/chirpy-starter
[pages-workflow-src]: https://docs.github.com/en/pages/getting-started-with-github-pages/configuring-a-publishing-source-for-your-github-pages-site#publishing-with-a-custom-github-actions-workflow
[latest-tag]: https://github.com/cotes2020/jekyll-theme-chirpy/tags
