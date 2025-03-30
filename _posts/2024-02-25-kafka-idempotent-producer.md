---
title: 카프카 멱등성 프로듀서를 통한 신뢰성 있는 데이터 전송
date: 2024-02-25 00:20:00
categories: [Java]
tags: [kafka]
pin: false
image:
  path: '/assets/img/kafka.webp'
---

이번 포스팅에서는 카프카 프로듀서가 분산 환경에서 어떻게 신뢰성 있는 데이터 전송을 보장하는지 알아보고자 합니다. 

## 프로듀서 중복 전송 이슈

카프카는 분산시스템이기 때문에 얼마든지 네트워크 이슈로 인한 예외 상황이 발생할 수 있다. 데이터 중복 전송이 발생할 수 있는 하나의 케이스를 살펴보자.

![스크린샷 2024-02-26 오후 10 37 36](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/36562d63-7bd5-4309-81d5-6729d47a891c)

위의 그림과 같이 카프카 프로듀서는 브로커로 메세지를 전송한 뒤, 브로커가 메세지 저장 후 되돌려주는 ACK를 통해 데이터 정상 전송 여부를 판별한다. 만약 브로커에서 데이터 저장에 실패해서
ACK가 돌아오지 않는다면 프로듀서는 일정 시간을 기다리다가 내부적으로 메세지를 재전송한다.

이때 만약 브로커에는 데이터가 정상적으로 저장되고 ACK까지 정상적으로 반환했는데, ACK가 프로듀서에게 가는 도중에 **네트워크 이슈로 인해 유실**되면 어떻게 될까? 프로듀서는 브로커에 메세지가 정상적으로 저장되지 않았다 판단 후 메세지를 재전송하게 된다.
브로커에는 이미 데이터가 저장되어 있기 때문에 **메세지 중복 발행**이 발생하는 것이다.

메세지 중복 발행은 결국 컨슈머의 메세지 중복 처리로 이어질 수 있기에 우리는 프로듀서가 **멱등성**을 가지도록 만들어야 한다. 그리고 이런 멱등한 특정을 가진 프로듀서를 **멱등성 프로듀서**(Idempotent Producer)라고 한다.

## 멱등성 프로듀서 동작 과정

프로듀서가 메세지의 멱등성을 보장하기 위해서 어떤 메커니즘을 사용하는지 살펴보자. 

프로듀서는 멱등성 보장에 **Producer ID**와 **Sequence** 두 가지를 Header에 넣어서 브로커로 전송한다. 이때 Producer ID는 프로듀서마다 고유하게 할당되는 ID로써 프로듀서가 재가동 될 때 마다
새로운 Producer ID가 부여된다. 다음으로 메세지 Sequence는 메세지의 고유한 식별변호로써 0부터 시작해서 순차적으로 증가한다. 

![스크린샷 2024-02-26 오후 9 47 08](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/7c168739-f433-4614-8767-ff3e2f939069)

브로커는 새로운 데이터가 들어올때 자신이 가지고 있는 마지막 SEQ에 **+1**을 한 SEQ를 가진 메세지만 저장한다. 만약 SEQ 값이 같다면 중복 메세지라 판단하고 프로듀서에게 ACK만 반환한다. 

이러한 동작을 통해 멱등성 프로듀서는 네트워크 장애로 인한 재전송 상황에서도 중복되지 않는 메세지 발행을 보장한다.

## 배치 간 순서 보장

메세지 재전송을 하게 되면 필연적으로 메세지 간 순서가 뒤바뀌는 문제가 발생한다. 특히 카프카 프로듀서는 `max.in.flight.requests.per.connection`라는 설정값 만큼 RecordBatch를 묶어서 전송하는데 만약 중간에 위치한 배치가 저장에 실패하면 재시도 과정에서 순서가 뒤바뀔 수 있다.

멱등성 프로듀서는 여러 배치를 묶어서 전달할때도 순서를 보장해준다. 아래의 그림을 살펴보자.

![스크린샷 2024-02-26 오후 10 29 14](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/95137d3d-dd0e-4463-b85f-f5c899e52695)

프로듀서는 브로커에 Batch 1,2,3을 묶어서 전송하고 있다. 이때 Batch 1은 정상적으로 저장됐지만, Batch 2를 저장하는 과정에서 문제가 발생해서 저장에 실패했다. 

이제 Batch 3이 어떻게 되는지가 중요한데, Batch 1의 SEQ가 0이고 Batch 3의 SEQ가 2이기 때문에 기존 SEQ에서 +1인 경우만 저장된다는 설정에 의해 `OutOfOrderSequenceException`이 반환된다. 

프로듀서는 이 예외를 받을 시 기존에 전송했던 Batch 1, Batch 2, Batch 3을 다시 전송한다. 멱등성 보장에 의해 이미 저장된 Batch 1은 ACK만 반환되고 나머지 Batch 2와 Batch 3은 순서에 맞춰서 정상적으로 저장된다.

## 멱등성 프로듀서 설정

위에서는 멱등성 프로듀서가 보장해주는 것들이 어떤게 있는지 살펴봤다. 그렇다면 우리는 어떻게 멱등성 프로듀서를 사용할 수 있을까?

프로듀서에 `enable.idempotence` 라는 설정을 true로 설정함으로써 우리는 멱등성 프로듀서를 사용할 수 있다. 하지만 카프카 2.4 버전 이후부터 이 값은 default가 true로 설정되있기 때문에 우리는 기본적으로 멱등성 프로듀서를 사용한다.

이외에 멱등성 프로듀서를 사용하기 위해서는 몇가지 설정이 추가적으로 되있어야 한다.

- **acks** : all (default - all)
- **retries** : 0 보다 큰 값 (default - Integer.MAX_VALUE)
- **max.in.flight.requests.per.connection** : 1과 5 사이의 값 (default - 5)

이 3가지 설정이 제대로 되있어야 우리는 멱등성 프로듀서를 사용할 수 있다. 만약 `enable.idempotence`를 명시적으로 true로 설정하지 않은 상황에서 위의 설정들을 준수하지 않으면 프로듀서는 정상 동작하지만 멱등성은 보장되지 않는다.

하지만 만약 `enable.idempotence`를 true라고 명시적으로 설정한 상태에서 위의 조건들을 준수하지 않았다면 예외가 발생하며 아예 프로듀서가 기동되지 않는다.

![스크린샷 2024-02-26 오후 11 05 08](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/f4a9fae9-f79b-4f17-aaa4-ed92c0d6ee53)

위와 같이 명시적으로 enable.idempotence를 설정한 상태에서 배치 개수를 7로 설정하면 어떤 일이 발생하는지 살펴보자

![스크린샷 2024-02-26 오후 11 06 03](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/4a8b5e70-ab1a-46a8-bf35-800c8e2a307c)

위와 같이 예외가 발생하면서 아예 프로듀서가 동작하지 않는걸 알 수 있다.

## 결론

멱등성 프로듀서를 사용하면 성능이 20% 정도 감소한다고 한다. 하지만 분산 시스템 환경에서 데이터 중복 전송의 위험성을 방지하고 순서를 보장해준다면 그정도 성능 저하는 충분히 감안할 수 있다고 생각한다.
















[nodejs]: https://nodejs.org/
[starter]: https://github.com/cotes2020/chirpy-starter
[pages-workflow-src]: https://docs.github.com/en/pages/getting-started-with-github-pages/configuring-a-publishing-source-for-your-github-pages-site#publishing-with-a-custom-github-actions-workflow
[latest-tag]: https://github.com/cotes2020/jekyll-theme-chirpy/tags
