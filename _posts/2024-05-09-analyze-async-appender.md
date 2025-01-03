---
title: Logback AsyncAppender 동작원리 분석
author: jemlog
date: 2024-05-09 00:20:00
categories: [Spring]
tags: [logback]
pin: false
image:
  path: '/assets/img/logback.png'
---

Logback에 대해 자세히 알아보는 과정에서 AsyncAppender를 통해서 로깅 작업을 비동기적으로 수행할 수 있다는걸 알게 됐습니다.
그렇다면 내부적으로 로깅을 대신해주는 쓰레드풀이 존재하는건지 동작 원리가 궁금해서 직접 디버깅을 통해 알아보는 시간을 가졌습니다.

## AsyncAppender 설정

우선 logback에서 AsyncAppender를 어떻게 설정하는지 알아보자.

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<configuration>
  
    <appender name="ASYNC-INFO" class="ch.qos.logback.classic.AsyncAppender">
        <appender-ref ref="INFO_LOG"/>
        <queueSize>100</queueSize>
        <discardingThreshold>20</discardingThreshold>
        <includeCallerData>false</includeCallerData>
        <neverBlock>true</neverBlock>
    </appender>
  
    <root level="INFO">
        <appender-ref ref="ASYNC-INFO" />
    </root>
  
</configuration>
```

logback.xml 파일에서 `ch.qos.logback.classic.AsyncAppender`를 추가하면 AsyncAppender를 사용 가능하다.
이때 AsyncAppender는 직접 로깅을 하는것이 아니라 비동기 처리 후 다른 Appender로 로깅을 위임한다. 따라서 반드시 `appender-ref`를 통해
실제 로그를 출력할 Appender를 추가해주자. 이번 예시에서는 `RollingFileAppender`를 AsyncAppender에 연결해봤다.

## AsyncAppender 동작 디버깅

그렇다면 AsyncAppender가 어떤 방식으로 동작하는지 살펴보자. AsyncAppender를 디버깅하면 `AsyncAppenderBase`라는 클래스에서 시작한다.

<img width="644" alt="스크린샷 2024-05-09 오후 11 28 51" src="https://github.com/jemlog/tech-study/assets/82302520/be220941-7315-4930-b879-56fec6513154">

어플리케이션이 시작될때 AsyncAppenderBase의 `start()` 메서드를 호출해서 초기화를 진행한다. 중간에 BlockingQueue를 생성하는걸 볼 수 있는데, AsyncAppender는 유입되는 로그 이벤트를 해당 큐에 적재한다. 이때 큐에 이벤트를 넣는 과정까지는 현재 요청을 수행하고 있는 톰캣 쓰레드가 진행한다.

그 다음으로는 **discardingThreshold**라는 값을 초기화한다. AsyncAppender는 큐에 로그가 많이 적재되서 특정 임계치를 넘어서면 이후에 들어오는 로그를 버릴지 아니면 톰캣 스레드를 잠시 Blocking할지를 결정한다. 이때 임계치가 discardingThreshold다. 해당 값은 `queueSize / 5`라는 공식으로 계산되는걸 알 수 있는데, 기본적으로 큐에 빈공간이 20% 미만으로 남으면 임계치 초과로 판단한다는 말이다. 이때 20%는 default 설정이고, 사용자가 직접 비율을 설정할수도 있다. 참고로 queueSize의 default 설정은 256이다.

start() 메서드 내부에서는 BlockingQueue를 초기화하고 discardingThreshold를 계산하는 과정이 진행된다.
이때 queueSize를 5로 나누는걸 알 수 있는데, BlockingQueue의 빈공간이 20% 미만으로 남으면 더이상 로그를 추가할 수 없다.

마지막으로 `worker.start()`를 호출하는데, 특정 스레드를 시작하는 것처럼 보인다. worker에 대해 조금 더 자세히 살펴보자.

![스크린샷 2024-05-11 오후 3 00 31](https://github.com/jemlog/tech-study/assets/82302520/3aaa15ef-b666-492f-819e-b42384100186)

워커 스레드 내부에서 while문을 사용해서 무한루프를 돌며 작업하는걸 알 수 있다. Netty의 이벤트루프나 카프카 프로듀서의 Sender 스레드와 비슷한 방식으로 동작한다는 생각이 들었다.

elements라는 리스트를 생성한 후, blockingQueue에서 drainTo()를 통해 로그 이벤트를 가져온다. 그리고 로그 이벤트 리스트를 순회하면서 `aai.appendLoopOnAppenders()`를 호출하는걸 알 수 있다. 여기서 aai가 무엇을 말하는걸까?

<img width="477" alt="스크린샷 2024-05-11 오후 3 06 48" src="https://github.com/jemlog/tech-study/assets/82302520/c649cfa1-ae42-4428-9ded-197054ce560c">

`appendLoopOnAppenders()` 내부로 들어오면 appenderArray를 순회하면서 각각의 요소들의 `doAppend()`를 호출하는걸 알 수 있다. 그러면 appenderArray에는 뭐가 들어있을까?

<img width="644" alt="스크린샷 2024-05-11 오후 3 08 56" src="https://github.com/jemlog/tech-study/assets/82302520/5d6b2c92-f09a-4f72-add4-c89bee4097bc">

appenderArray내에서 우리가 위에서 AsyncAppender와 ref로 연결한 **RollingFileAppender**가 들어와있다. AsyncAppender는 오직 하나의 Appender만 ref가 가능하기 때문에 appenderArray내에는 하나의 appender만 들어온다.
굳이 배열을 사용한 이유는 `appenderLoopOnAppenders()` 메서드를 가진 **AppenderAttachableImpl** 클레스가 Appender를 n개 ref할 수 있는 곳에서도 사용되기 때문에 호환성을 위함이 아닐까 싶다.

`doAppend()`안으로 더 들어간다면 우리가 알아보고자 했던 AsyncAppender를 넘어 RollingFileAppender를 탐색해야 하기 때문에 해당 포스팅에서는 여기까지만 들어가겠다.

마지막으로는 톰캣 쓰레드가 BlockingQueue에 로그를 적재하는 부분을 살펴보자. 

<img width="561" alt="스크린샷 2024-05-11 오후 3 17 50" src="https://github.com/jemlog/tech-study/assets/82302520/f85cefff-1f36-49d1-b687-0c2bf7404f80">

톰캣 쓰레드는 `append()` 메서드를 호출하고 내부적으로 `put()` 메서드를 호출해서 실제 BlockingQueue에 로그 이벤트를 적재한다.

## 정리

AsyncAppender는 크게 다음의 순서로 동작한다고 정리할 수 있을 것 같다.

1. 톰캣 쓰레드가 log를 AsyncAppender의 BlockingQueue에 계속 적재한다.
2. Worker Thread가 무한루프를 돌면서 BlockingQueue에서 데이터를 빼낸다.
3. 빼낸 데이터를 ref로 연결된 Appender에게 전달해서 실제로 로그를 기록한다.

즉, 톰캣 쓰레드가 직접 로그를 쓰는 I/O작업까지 진행하지 않고, Worker 쓰레드가 비동기적으로 진행하기 때문에 성능을 향상시킬 수 있다는 것이다.

[nodejs]: https://nodejs.org/
[starter]: https://github.com/cotes2020/chirpy-starter
[pages-workflow-src]: https://docs.github.com/en/pages/getting-started-with-github-pages/configuring-a-publishing-source-for-your-github-pages-site#publishing-with-a-custom-github-actions-workflow
[latest-tag]: https://github.com/cotes2020/jekyll-theme-chirpy/tags
