---
title: Redis pub/sub 내부 구현 방식과 특징 분석
author: jemlog
date: 2024-01-10 00:20:00
categories: [Redis]
tags: [pubsub]
pin: false
image:
  path: '/assets/img/redis.jpg'
---

프로젝트에 레디스 pub/sub을 적용하면서 내부 구현 방식과 다른 기술과의 차이점이 궁금해졌다. 먼저 내부가 어떻게 생겼는지 파악한 뒤에 구조는 다르지만 비슷한 기능을 제공하는 **카프카**와 비교 분석을 진행해보자.

## Redis pub/sub의 내부 동작 원리

레디스가 내부적으로 pub/sub을 어떻게 구현했는지 살펴보자.

<img width="500" alt="스크린샷 2024-01-11 오전 12 09 50" src="https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/2e3ca260-8563-4704-9d76-92c44460d115">

레디스 pub/sub 내에는 `pubsub_channel`이라는 딕셔너리 타입의 변수가 존재한다. 해당 딕셔너리 안에 Channel과 구독자 정보를 보관한다. 딕셔너리의 Key는 `Channel`이고 Value는 구독자를 보관하는 `LinkedList`가 들어간다.

만약 채널에 구독자가 추가되면 채널과 매핑된 연결 리스트에 구독자 정보를 추가한다. 발행자가 메세지를 Channel에 보내면 딕셔너리에서 Channel을 찾고 그 채널에 해당하는 연결 리스트를 순회해서 메세지를 구독자들에게 발행한다.

## 다른 메세지 큐와 비교한 레디스 pub/sub의 특징

레디스 pub/sub은 컴포넌트들 간의 디커플링을 위한 메세지 큐로 사용할 수 있다. 그렇다면 다른 메세지 큐(Kafka)와의 차이점은 어떤 것들이 있는지 알아보자.

### 메세지 보관 여부
카프카의 경우 내부적으로 큐를 가지고 있다. 따라서 메세지가 프로듀서에 의해 발행되면 **내부의 큐에 메세지가 쌓이게 된다**. 메세지는 **디스크에 저장**되기 때문에 컨슈머가 메세지 수신에 실패하더라도 재시도가 가능하다.

반면 레디스 pub/sub은 내부에 큐가 없기 때문에 한번 발행된 메세지는 저장되지 않는다. 따라서 발행자가 한번 메세지를 발행하고 구독자들에게 뿌리고 나면 해당 메세지는 바로 사라진다. 

따라서 현재 서비스가 메세지의 손실이 발생하면 안되고, `재시도 큐`나 `Dead Letter Queue`를 사용해서라도 실패 처리를 제대로 해줘야 한다면 카프카를 사용해야 한다. 하지만 현재 서비스가 **메세지의 손실을 감내**할 수 있고 **빠른 응답 속도**가 더 중요하다면
레디스 pub/sub을 사용하는게 효율적이다.

### 메세지 전송 방식

메세지 전송 모델에는 **Push 방식**과 **Pull 방식**이 있다. Push 방식은 메세지 발행자의 속도에 맞춰서 메세지를 수신자에게 전달하는 방식이다. 반면 Pull 방식은 메세지 수신자가 원하는 속도로 메세지를 받아온다. 여기서 카프카의 경우 Pull 방식을 사용하고 레디스 pub/sub의 경우 push 방식을 사용한다.

Push 방식을 사용하는 경우 발행자의 송신 성능에 비해 수신자의 처리 성능이 떨어지지 않도록 성능을 최적화 하는게 중요하다. 

<img width="600" alt="스크린샷 2024-01-11 오전 12 38 50" src="https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/35aef840-146d-42cd-bbaf-c264cf9a6720">

만약 메세지 수신 서버의 데이터 처리 속도가 높지 않다면 OS의 TCP 소켓 수신 버퍼에는 계속 패킷이 쌓인다. 소켓 버퍼의 여유 공간이 없기 때문에 **TCP의 흐름 제어**에 의해 수신측은 송신측에 허용 가능 Window Size를 계속 줄여서 보내게 된다. 이는 메세지 발행 서버와 메세지 수신 서버 간의 레이턴시 증가로 이어져 서비스의 **TPS**와 **응답 시간**에 악영향을 미친다. 따라서 수신 서버의 어플리케이션은 수신 데이터 처리 속도가 떨어지지 않도록 주의를 기울여야 한다.

`Spring Data Redis`를 사용하면 Lettuce를 Redis Client로 사용하게 된다. Lettuce는 Netty 기반으로 구성되어 있고 비동기 논블로킹을 제공한다. Netty는 NioEventLoop를 사용해서 무한 루프를 돌며 TCP 소켓 버퍼에서 데이터를 읽어온다. 이후 실제 데이터 관련 처리는 다른 쓰레드 풀에 넘겨버리면 다시 데이터 읽는데 집중할 수 있어서 블로킹 없이 데이터를 빠르게 읽어올 수 있다.

### TCP 수신 소켓 버퍼가 계속 채워지는 경우 대처법

여기까지만 해도 송신 서버의 메세지 전송 속도에 맞춰 빠르게 데이터를 처리할 수 있다. 하지만 만약 트래픽이 높아져서 수신 버퍼에 패킷이 계속 쌓인다면 어떻게 대처할 수 있을까?

- NioEventLoop에 데이터 읽는 로직 이외의 동작이 수행되는지 체크
  - NioEventLoop는 **싱글 스레드**이기에 **무거운 연산**이나 **I/O**가 들어가면 곧바로 레이턴시 증가로 이어진다. 따라서 NioEventLoop가 오직 **데이터 수신 작업에만 집중**하도록 만들어야 한다.
- 불필요한 쓰레드 개수를 줄여서 컨텍스트 스위칭을 감소
  - 애플리케이션 내에 쓰레드 개수가 많아질수록 CPU를 할당하기 위한 **컨텍스트 스위칭 비용이 증가**한다. 따라서 불필요한 쓰레드가 있다면 개수를 조절해야 한다.

## 결론

지금까지 레디스 Pub/Sub의 내부 구현 방식과 레디스 Pub/Sub의 Push 방식 사용 시 주의해야 하는 점을 살펴봤다. 서비스를 거시적인 관점에서 바라보면 여러 컴포넌트들로 구성된다. 웹서버, WAS 서버, 메세지 큐, 데이터베이스 등 많은 컴포넌트의 조합으로 구성되어 있고 각 컴포넌트에 부합하는 여러 기술 후보군이 존재한다.

기술은 기본적으로 상황에 따라 **트레이드 오프**가 존재하기 때문에 서비스 특성에 따라 어떤 기술이 단점 대비 장점이 확실한지를 꼼꼼히 따져봐야 한다. 메세지 큐로 레디스 Pub/Sub과 카프카의 장단점을 살펴본 이번 학습처럼 앞으로도 여러 기술을 비교 분석해보는 시간을 가져야겠다고 생각했다.

## Reference

- [https://www.youtube.com/watch?v=SF7eqlL0mjw](https://www.youtube.com/watch?v=SF7eqlL0mjw)






[nodejs]: https://nodejs.org/
[starter]: https://github.com/cotes2020/chirpy-starter
[pages-workflow-src]: https://docs.github.com/en/pages/getting-started-with-github-pages/configuring-a-publishing-source-for-your-github-pages-site#publishing-with-a-custom-github-actions-workflow
[latest-tag]: https://github.com/cotes2020/jekyll-theme-chirpy/tags