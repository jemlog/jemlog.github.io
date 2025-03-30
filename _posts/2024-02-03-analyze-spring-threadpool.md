---
title: ThreadPoolExecutor 파헤쳐보기
date: 2024-02-03 00:20:00
categories: [Java]
tags: [thread]
pin: false
image:
  path: '/assets/img/spring.webp'
---

최근 스프링이 제공하는 @Async를 사용한 비동기 작업을 구현하다가, 작업을 실행하는 쓰레드풀에 대해 조금 더 자세히 알아봐야 겠다는 생각이 들었습니다. 이번 글에서는 설정값들을 변경하며 ThreadPoolExecutor를 가지고 놀아본 경험을 기록해보고자 합니다.

## ThreadPoolExecutor은 무엇인가

ThreadPoolExecutor는 ExecutorService를 구현한 클래스로서 매개변수를 통해 다양한 설정과 조정이 가능하며 사용자가 직접 컨트롤 할 수 있는 쓰레드풀입니다.

![스크린샷 2024-02-04 오후 9 18 53](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/ff384b56-4879-4b69-9d23-842849bb790e)


위의 코드와 같이 여러 옵션들을 사용자가 쓰레드풀 생성시에 직접 설정할 수 있습니다. 


![스크린샷 2024-02-04 오후 9 20 42](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/29599288-aa2b-4387-80fc-2959cecaec72)


저희가 고정 크기의 쓰레드풀을 생성하기 위해서는 `Executors.newFixedThreadPool(nThreads)` 메서드를 사용합니다. 이때 메서드 내부적으로도 결국 ThreadPoolExecutor가 사용됩니다. 

## CorePoolSize & MaximumPoolSize

먼저 쓰레드풀 설정의 핵심인 CorePoolSize와 MaximumPoolSize에 대해 알아보겠습니다. 

MaximumPoolSize는 직관적으로 쓰레드풀에 생성할 수 있는 **쓰레드의 최대 개수** 입니다. 만약 요청이 많이 들어오는 상황에서 MaximumPoolSize 만큼의 쓰레드가 모두 동작하고 있고, 새로운 쓰레드가 필요하다면 ThreadPoolExecutor는 새로 들어오는 작업을 어떻게 처리할지를 내부적인 정책에 따라
선택하게 됩니다. 어떤 정책을 사용할지도 사용자가 선택할 수 있는데, 이 부분은 아래에서 더 자세히 살펴보겠습니다.

그렇다면 CorePoolSize는 어떤걸 말하는 걸까요? CorePoolSize는 쓰레드풀이 **최소한으로 유지할 쓰레드의 개수**를 나타냅니다. 만약 CorePoolSize가 5라면 ThreadPoolExecutor는 요청이 없는 유휴 상태에서도 최소 5개의 쓰레드를 풀 내에 유지하게 됩니다.

그럼 CorePoolSize는 어플리케이션이 시작하자마자 곧바로 채워지는 걸까요? 자바 쓰레드는 기본적으로 생성 과정에서 OS 쓰레드를 함께 생성 후 일대일 매핑 과정을 거치기 때문에 생성 비용이 비쌉니다. 따라서 당연히 런타임에 쓰레드풀의 동작 속도를 향상시키기 위해 미리 CorePoolSize 만큼 생성하는게 디폴트일 것이라 생각했습니다.

하지만 실험 결과, 처음부터 CorePoolSize만큼 채우지 않고, **요청이 들어올때마다 쓰레드를 CorePoolSize까지 하나씩 늘려나갔습니다**.

```java
@DisplayName("corePoolSize 까지 쓰레드가 존재하기 이전에는 매번 새로운 쓰레드를 만든다")
@Test
void threadPoolTest() throws InterruptedException {

    /*
    1번 파라미터 : corePoolSize
    2번 파라미터 : maximumPoolSize
    3번 파라미터 : keepAliveTime
    4번 파라미터 : keepAliveTime TimeUnit
    5번 파라미터 : Task Queue  
     */
    ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(5, 5, 60, TimeUnit.SECONDS, new LinkedBlockingDeque<>());
    
    CountDownLatch countDownLatch = new CountDownLatch(3);

    // 쓰레드풀 생성 후 Task 3개 제출
    for(int i =0; i < 3; i++){
        threadPoolExecutor.execute(() -> {
            System.out.println(Thread.currentThread().getName() + " start");
            countDownLatch.countDown();
        });
    }

    countDownLatch.await();
    
    // CorePoolSize가 5라도 처음에는 들어온 요청 개수 만큼 쓰레드풀이 유지된다
    // 테스트 성공!
    Assertions.assertThat(threadPoolExecutor.getPoolSize()).isEqualTo(3);
}
```

테스트 코드를 실행할 시 정상적으로 요청 3개 처리 후 쓰레드 풀 크기가 3으로 설정된 것을 알 수 있습니다.

![스크린샷 2024-02-04 오후 9 47 48](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/d9759426-1284-4597-a802-9ef3e40a08fd)

이를 통해 CorePoolSize만큼 먼저 쓰레드를 할당하고 시작하는게 디폴트가 아니라는걸 알았습니다. 그렇다면 쓰레드풀 초기화 시점에 미리 CorePoolSize만큼 생성하고 시작하는 방법은 없을까요? ThreadPoolExecutor의 `prestartAllCoreThreads()`를 호출하면 모든 코어 쓰레드를 초기화 시점에 다 채워넣을 수 있습니다.

![스크린샷 2024-02-04 오후 9 52 12](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/4a5af110-43f6-4daf-a94e-59184a081f73)

해당 메서드를 호출하면 내부적으로 n(corePoolSize)만큼 addWorker()를 호출해서 쓰레드풀에 쓰레드를 채워넣습니다. 

```java
@DisplayName("prestart를 하면 처음부터 corePoolSize만큼 쓰레드가 채워진다")
@Test
void threadPoolTest_Prestart() throws InterruptedException {

    
    ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(5, 5, 60, TimeUnit.SECONDS, new LinkedBlockingDeque<>());
    
    // 처음부터 코어 쓰레드를 모두 생성
    threadPoolExecutor.prestartAllCoreThreads();


    CountDownLatch countDownLatch = new CountDownLatch(3);
    
    for(int i =0; i < 3; i++){
        threadPoolExecutor.execute(() -> {
            System.out.println(Thread.currentThread().getName() + " work");
            countDownLatch.countDown();
        });
    }

    countDownLatch.await();

    // 작업이 총 3회 제출됐지만, 쓰레드풀의 쓰레드 개수는 CorePoolSize인 5개가 채워져있다.
    // 테스트 통과!
    Assertions.assertThat(threadPoolExecutor.getPoolSize()).isEqualTo(5);
}
```

## KeepAliveTime

어플리케이션에 트래픽이 많아져서 쓰레드풀에도 태스크가 많이 제출된다면 CorePoolSize 개수 이상의 쓰레드가 필요해집니다. MaximumPoolSize까지 차근차근 개수를 늘리며 다수의 요청에 대응하는 것이지요. 하지만 요청이 많이 들어오지 않는 순간에도
굳이 MaximumPoolSize만큼 쓰레드풀을 유지하는건 리소스를 낭비하는 방법이라 생각합니다.

ThreadPoolExecutor는 KeepAliveTime이라는 설정값을 제공합니다. 만약 CorePoolSize 이상의 쓰레드가 keepAliveTime 동안 유휴상태로 머물러 있다면 CorePoolSize까지 쓰레드 개수를 차근차근 줄여나갑니다. 

만약 현재 쓰레드풀의 CorePoolSize가 **5**, MaximumPoolSize가 **10**, keepAliveTime이 **10초** 그리고 현재 쓰레드가 **10개**까지 생성되어 있다면 10초간의 유휴 상태를 보낸 후에는 5개까지 쓰레드 개수를 감소시킵니다.

```java
@DisplayName("keepAliveTime이 지나면 corePoolSize만큼 줄어든다")
@Test
void threadPoolTest_keepAlive() throws InterruptedException {

    // Given
    ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(5, 10, 5, TimeUnit.SECONDS, new SynchronousQueue<>());
    
    CountDownLatch countDownLatch = new CountDownLatch(10);
        
    for(int i =0; i < 10; i++){
        threadPoolExecutor.execute(() -> {
            System.out.println(Thread.currentThread().getName() + " work");
            countDownLatch.countDown();
        });
    }

    countDownLatch.await();
    Thread.sleep(6000); // 5초간의 유휴 시간 확보를 위해 6초간의 sleep
        
    // 총 10개의 태스크가 제출됐지만 6초 후에는 CorePoolSize 만큼만 쓰레드가 유지된다
    Assertions.assertThat(threadPoolExecutor.getPoolSize()).isEqualTo(5);
}
```
그렇다면 KeepAliveTime이 지났을때 쓰레드풀을 아예 비워버리는 방법도 있을까요? ThreadPoolExecutor는 `allowCoreThreadTimeOut()` 메서드를 통해 CorePoolSize 개수 이하로도 
개수가 줄어들게 만들어줍니다.

```java
@DisplayName("coreThreadTimeOut을 설정하면 keepAliveTime 이후 core Thread Pool도 제거된다")
@Test
void threadPoolTest_core_thread() throws InterruptedException {

    
    ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(5, 10, 5, TimeUnit.SECONDS, new SynchronousQueue<>());
    
    // CorePoolSize도 제거하는걸 허용하는 옵션
    threadPoolExecutor.allowCoreThreadTimeOut(true);
        
    CountDownLatch countDownLatch = new CountDownLatch(10);

    // When
    for(int i =0; i < 10; i++){
        threadPoolExecutor.execute(() -> {
            System.out.println(Thread.currentThread().getName() + " work");
            countDownLatch.countDown();
        });
    }

    countDownLatch.await();
    Thread.sleep(6000);

    // CorePoolSize가 5개임에도 쓰레드풀을 완전히 비워버린다
    Assertions.assertThat(threadPoolExecutor.getPoolSize()).isEqualTo(0);
}
```

## Task Queue

스레드풀은 만약 corePoolSize 이상으로 태스크가 제출되면 **쓰레드를 곧바로 늘리지 않고 작업 큐에 작업을 추가합니다**. 작업 큐에 테스크를 계속 쌓아나가다가 작업 큐가 꽉 차서 더이상 수용할 수 없는 순간부터
쓰레드풀은 MaximumPoolSize까지 새로운 쓰레드를 생성합니다. 

그래서 만약 극단적으로 작업 큐 사이즈를 무한대로 설정하면 테스크 또한 무한대로 받을 수 있기 때문에 CorePoolSize 이상으로 쓰레드가 늘어날 일이 없습니다. 

![스크린샷 2024-02-04 오후 10 33 11](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/a1348daa-c18d-416e-b7f1-a534a1002a54)

저희가 보통 @Async를 실행하는 쓰레드풀을 빈으로 등록할때 사용하는 ThreadPoolTaskExecutor를 가져와봤습니다. QueueCapacity라는 변수의 디폴트값이 `Integer.MAX_VALUE`로 설정된 걸 알 수 있습니다.
만약 해당 값을 사용자가 따로 설정하지 않는다면 작업 정체 상황에서 작업 큐에 계속 테스크가 쌓이게 될 것입니다.

ThreadPoolExecutor 클래스에서는 직접 작업 큐의 종류를 선택해서 생성자 파라미터로 전달할 수 있는데요. 어떤 타입의 큐들을 사용할 수 있는지 살펴보겠습니다.

### SynchronousQueue
- 내부적으로 크기가 0인 큐로서 스레드 간에 작업을 전달하는 역할을 한다. 태스크를 대기열에 넣으려고 할때 실행할 스레드가 즉시 없으면 새로운 스레드를 생성합니다
- `Executors.newCachedThreadPool()`에서 사용됩니다

### LinkedBlockingQueue
- 무제한 크기의 큐로서 corePoolsize의 스레드가 모두 사용중인 경우 새로운 작업이 제출되면 대기열에 등록하고 대기하게 됩니다
- `Executors.newFixedThreadPool()`에서 사용됩니다

### ArrayBlockingQueue
- 내부적으로 고정된 크기의 배열을 사용하여 작업을 추가하고 큐를 생성할 때 최대 크기를 지정해야 합니다

## RejectPolicy

그렇다면 만약 작업 큐도 꽉 찼고 쓰레드도 MaximumPoolSize까지 생성된 상황에서 새로운 태스크가 들어오면 어떻게 될까요? 더이상 태스크를 보관할 곳도 없기 때문에 이때부터는 예외 처리를 고려해야 합니다.

ThreadPoolExecutor는 이 상황에 어떻게 동작해야 하는지에 대한 정책을 설정할 수 있는데요. `threadPoolExecutor.setRejectedExecutionHandler(policy)` 메서드를 사용해서 정책을 등록합니다.

사용 가능한 정책들로는 어떤 것들이 있는지 살펴보겠습니다.

### AbortPolicy
Abortpolicy는 새롭게 작업이 들어오면 곧바로 RejectedException을 예외로 반환합니다. 물론 해당 작업은 버려지게 됩니다. ThreadPoolExecutor는 해당 정책을 디폴트 설정으로 사용합니다.

```java
@DisplayName("AbortPolicy로 선택하면 RejectedExecutionException을 반환한다")
@Test
void abort_policy_test() {

    ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(5, 5, 5, TimeUnit.SECONDS, new ArrayBlockingQueue<>(2));

    // 정책 설정
    threadPoolExecutor.setRejectedExecutionHandler(new ThreadPoolExecutor.AbortPolicy());

    /*
    MaximumPoolSize가 5, Queue 사이즈가 2인 상황에서 8개 작업이 동시에 들어오는 상황을 테스트  
     */
    Assertions.assertThatThrownBy(() -> {
        for(int i =0; i < 8; i++){
            threadPoolExecutor.execute(() -> {
                try {
                    System.out.println(Thread.currentThread().getName() + " work");
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }

            });
        }
    }).isInstanceOf(RejectedExecutionException.class);
}
```

### CallerRunsPolicy
CallerRunsPolicy는 새롭게 작업이 들어올때 수행할 수 있는 여분의 쓰레드를 생성할 수 없다면 쓰레드풀에 작업을 제출한 부모 쓰레드가 스스로 작업을 수행합니다. 만약 메인 쓰레드에서 해당 쓰레드풀에 작업을
제출했다면 메인 쓰레드가 직접 수행하게 되는 것입니다.

```java
@DisplayName("CallerRunsPolicy로 선택하면 부모 쓰레드가 실행한다")
@Test
void caller_run_policy_test() throws InterruptedException {
    
    
    ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(5, 5, 5, TimeUnit.SECONDS, new ArrayBlockingQueue<>(2));
    
    // 정책 설정
    threadPoolExecutor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());

    CountDownLatch countDownLatch = new CountDownLatch(8);

    
    for(int i =0; i < 8; i++){
        threadPoolExecutor.execute(() -> {
            System.out.println(Thread.currentThread().getName()); // 포화 상태에서 새로 들어온 쓰레드는 메인 쓰레드가 실행 예정
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
            countDownLatch.countDown();
        });
    }
        
    countDownLatch.await();
}
```

아래와 같이 나머지 요청들은 쓰레드풀의 쓰레드에서 실행되지만 포화 상태 이후에 들어온 테스크는 쓰레드풀을 작동시킨 메인 쓰레드가 직접 수행합니다.

![스크린샷 2024-02-04 오후 11 14 54](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/97616ecf-9185-4fc5-87fa-924d4d5b38fc)

### DiscardPolicy
DiscardPolicy는 새롭게 작업이 들어올때 AbortPolicy와 다르게 예외를 터뜨리지 않고 조용하게 새로운 작업을 버립니다. 예외 처리가 불가능하기 때문에 작업 유실을 감내할 수 있는 기능에 사용합니다.

```java
    @DisplayName("DiscardPolicy로 선택하면 조용히 버려진다")
    @Test
    void discard_policy_test() throws InterruptedException {
         
        // maximumPoolSize가 1개, 작업 큐가 2개로 4번째 태스크부터 유실된다
        ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(1, 1, 5, TimeUnit.SECONDS, new ArrayBlockingQueue<>(2));

        threadPoolExecutor.setRejectedExecutionHandler(new ThreadPoolExecutor.DiscardPolicy());

        CountDownLatch countDownLatch = new CountDownLatch(3);

        // AbortPolicy와 다르게 RejectedExecutionException 발생하지 않고 정상 통과한다
        for(int i =0; i < 4; i++){
            threadPoolExecutor.execute(() -> {
                System.out.println(Thread.currentThread().getName());
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
                countDownLatch.countDown();
            });
        }
    }
```

### DiscardOldestPolicy
DiscardOldestPolicy는 새롭게 작업이 들어올때 기존의 작업 중 가장 오래된 작업을 버리고, 새로운 작업을 추가합니다. DiscardPolicy와 함께 작업 유실을 감내할 수 있는 기능에서 사용 가능합니다.

```java
@DisplayName("DiscardOldestPolicy로 선택하면 작업 큐의 가장 오래된 작업이 유실된다")
@Test
void discard_oldest_policy_test() throws InterruptedException {
       
    ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(1, 1, 5, TimeUnit.SECONDS, new ArrayBlockingQueue<>(2));

    threadPoolExecutor.setRejectedExecutionHandler(new ThreadPoolExecutor.DiscardOldestPolicy());

    CountDownLatch countDownLatch = new CountDownLatch(3);

    /*
    first ... fourth 순서로 ThreadPoolExecutor에 진입
    */
    
    threadPoolExecutor.execute(() -> {
        System.out.println("first");
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
               throw new RuntimeException(e);
        }
        countDownLatch.countDown();
    });
    
    threadPoolExecutor.execute(() -> {
        System.out.println("second");
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
        countDownLatch.countDown();
    });

    threadPoolExecutor.execute(() -> {
        System.out.println("third");
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
        countDownLatch.countDown();
    });

    threadPoolExecutor.execute(() -> {
        System.out.println("fourth");
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
        countDownLatch.countDown();
    });

    countDownLatch.await();
    threadPoolExecutor.shutdown();
}
```

<img width="785" alt="스크린샷 2024-02-04 오후 11 32 54" src="https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/756137db-30d0-409f-90f8-4a26ce471fc0">

## 결론

이번 기회를 통해 ThreadPool의 개념적인 부분에 대해 많이 이해할 수 있었습니다. 아직 실전에 적극적으로 사용해보지는 못해서 앞으로 쓰레드풀 적용을 통해 성능을 개선할 수 있는 부분들을 적극적으로 찾아봐야겠다 생각했습니다.



[nodejs]: https://nodejs.org/
[starter]: https://github.com/cotes2020/chirpy-starter
[pages-workflow-src]: https://docs.github.com/en/pages/getting-started-with-github-pages/configuring-a-publishing-source-for-your-github-pages-site#publishing-with-a-custom-github-actions-workflow
[latest-tag]: https://github.com/cotes2020/jekyll-theme-chirpy/tags
