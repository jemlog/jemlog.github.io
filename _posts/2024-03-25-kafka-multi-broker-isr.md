---
title: 카프카 멀티 브로커 ISR 분석
author: jemlog
date: 2024-03-25 00:20:00
categories: [Kafka]
tags: [kafka]
pin: false
image:
  path: '/assets/img/kafka.webp'
---

이번 포스팅에서는 카프카의 멀티 브로커에서 사용되는 **ISR**(In Sync Replica)라는 개념에 대해서 알아보고자 합니다. 
ISR은 카프카가 메세지 유실 없이 가용성을 보장하기 위한 중요한 개념입니다.
또한 ISR 관련 설정에 따라 프로듀서가 메세지를 브로커에 정상적으로 전송하지 못할 수도 있기 때문에 잘 알아두는게 중요합니다.

## 주키퍼를 사용한 카프카 클러스터 관리

ISR에 대해 알아보기 전에 카프카가 어떻게 멀티 브로커 환경을 구축하는지 살펴보고자 한다. 카프카는 멀티 브로커의 메타데이터 관리를 위해 **주키퍼**라는 외부 컴포넌트를 사용한다.

### 주키퍼

주키퍼는 분산 시스템에서 시스템 간의 정보 공유, 상태 체크, 서버 들 간의 동기화를 위한 락 등의 역할을 하는 분산 코디네이션 서비스이다.
주키퍼는 상태 정보들을 주키퍼의 Z Node라는 곳에 Key-Value 형태로 저장하며, 디렉토리 기반의 구조로 되어 있다.

분산 시스템에서 주키퍼를 단일 노드로 사용한다면 주키퍼가 SPOF(Single Point Of Failure)가 될수도 있다. 따라서 가용성을 위해 주키퍼도 클러스터로 구축하는데, 이를 **주키퍼 앙상블**이라고 한다. 
최근에는 카프카 클러스터 구축에 있어서 주키퍼 같은 외부 컴포넌트에 대한 의존을 줄이기 위해 **Kraft** 모드라는게 생겨났는데, 차후 포스팅에서 한번 다뤄보고자 한다.

### 카프카에서 주키퍼의 역할

그렇다면 카프카에서 주키퍼는 어떻게 사용될까? 주키퍼는 개별 노드들의 상태 정보를 Z Node에 저장한다. 
클러스터의 개별 노드들은 주키퍼의 Z Node를 계속 모니터링하면서 Z Node에 변경 발생 시 Watch Event가 트리거가 되서 변경 정보가 개별 노드들에게 통보된다.
이외에도 주키퍼는 클러스터 내 브로커의 멤버쉽을 관리하고 토픽이나 레플리카들의 정보를 가진다. 

## 멀티 브로커를 통한 파티션 복제

그렇다면 멀티 브로커를 통해 어떤걸 할 수 있을까. 하나의 토픽은 여러 파티션으로 분리되고, 카프카의 가용성을 위해서 파티션을 여러 노드에 복제해서 관리할 수 있다. 이때 프로듀서가 데이터를 넣고 컨슈머가 소비하는 파티션을 **리더 파티션**이라 하고, 해당 파티션을 가지고 있는 
브로커를 리더 브로커라고 한다. 반면 해당 리더 브로커로부터 데이터를 Fetch 해서 자신의 파티션에 동기화 하는 파티션을 팔로워 파티션이라 하고 해당 파티션을 가지고 있는 브로커를 팔로워 브로커라고 한다.

## 컨트롤러 브로커

멀티 브로커 간의 파티션 리더 선출 작업을 위해서는 이를 통제하는 하나의 컨트롤러가 필요하다. 카프카에서는 이 노드를 **컨트롤러 브로커**라고 한다.
컨트롤러 브로커는 멀티 브로커 간의 파티션 리더 재선출 과정을 조율하고 결과를 주키퍼에게 전달하는 역할을 수행한다. 
특정 브로커에 장애가 발생한 뒤, 컨트롤러 브로커에 의해 재선출이 일어나는 과정을 살펴보자. 

1. 하나의 브로커가 셧다운 되고 HeartBeat가 오지 않으면 주키퍼는 해당 노드의 정보를 Z-Node에서 제거한다
2. 컨트롤러 브로커가 Z-Node를 계속 모니터링 하다가 Watch Event로 브로커가 다운됐다는 정보를 받는다
3. 컨트롤러 브로커는 다운된 브로커가 관리하던 파티션들에 대해 새로운 리더를 선출한다
4. 결정된 새로운 리더 정보를 주키퍼에 저장하고 해당 리더 파티션을 복제하는 모든 브로커들에게 새로운 리더/팔로워 정보를 전달한다
5. 컨트롤러 브로커는 다른 브로커들이 가지는 메타데이터 캐시를 전부 새로운 리더 팔로워 정보로 갱신하도록 요청한다

### 컨트롤러 브로커가 죽는 경우

만약 컨트롤러 브로커가 죽으면 어떻게 될까? 
컨트롤러 브로커에게서 일정 시간 동안 Heartbeat가 오지 않으면 주키퍼는 컨트롤러 브로커가 죽었다 판단하고 전체 브로커에게 새로운 컨트롤러 선출에 대해 공지한다. 
이때 모든 브로커 중 **가장 먼저 주키퍼에게 요청이 붙은 브로커**가 컨트롤러 브로커로 재선출 된다.

## In Sync Replica

Active-Standby 형태의 복제본을 구축하는 이유가 무엇인지 생각해보자. 리더가 장애가 났을때 팔로워를 리더로 승격시켜서 사용자에게 **데이터 손실 없이 응답을 내려주기 위함**이다.

하지만 만약 팔로워가 리더의 최신 데이터를 제대로 복제하고 있지 못한 상태에서 리더로 승격된다면 어떤 일이 발생할까? 
컨슈머 입장에서는 기존 리더가 가지고 있던 **최신 데이터가 유실된 상태**로 메세지를 소비하게 될 것이다. 
이렇게 되면 시스템의 신뢰성이 손상 받게 된다.

이 문제를 방지하기 위해 **ISR**(In Sync Replica)라는 개념이 사용되고, 결론부터 말하면 ISR 범위 내에 속해있는 팔로워들만 리더로 승격이 될 수 있다. ISR이 무엇인지 자세히 알아보자.

<img width="787" alt="스크린샷 2024-03-20 오전 10 41 59" src="https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/9b021afc-859e-42d3-99df-f70f0e94ad77">

위의 사진은 멀티 브로커 환경에서 토픽의 정보를 조회한 결과이다. Isr이라는 항목이 있는걸 알 수 있다.

0번 파티션을 예로 들면, 파티션 리더 브로커는 3번 브로커가 할당되어 있고 나머지 1,2번 브로커는 팔로워로 리더의 정보를 복제해서 저장한다. 
이때 Isr은 2,1,3으로 설정이 되어 있는데 리더 브로커 3번이 예기치 못하게 장애가 발생해도 ISR에 속해 있는 1,2번 브로커 중 하나가 리더로 선출될 수 있다.
그렇다면 특정 브로커가 ISR에서 제외되는 상황에 어떤 것들이 있는지 살펴보자.

### 특정 브로커로부터 주키퍼에게 HeartBeat가 오지 않는 경우

기본적으로 브로커는 주키퍼에 연결이 되있어야 한다. **zookeeper.session.timeout.ms**(기본 6초, 최대 18초) 시간 내에 주키퍼에게 HeartBeat를 지속적으로 보내지 않으면 ISR에서 제외된다.

### 팔로워 브로커가 리더 브로커의 메세지를 빠르게 가져오지 않는 경우

팔로워 브로커는 기본적으로 **500ms** 마다 리더 브로커에게 데이터 동기화를 위한 fetch 요청을 전송한다. 이때 **replica.lag.time.max.ms**(기본 10초, 최대 30초)라는 설정이 있는데 특정 브로커가 장애가 난 경우와 데이터 동기화 속도가 느린 경우 두 가지 케이스와 연관이 있다.

만약 브로커에 문제가 발생해서 replica.lag.time.max.ms 동안 데이터를 fetch하지 못하면 ISR에서 제외된다. 또한 만약 replica.lag.time.max.ms 동안 리더 파티션의 최신 오프셋을 팔로워 파티션이 계속 따라가지 못하면 ISR에서 제외한다.

## min.insync.replicas

카프카 프로듀서는 브로커에 메세지를 전송한 후 브로커로부터 ACK 메세지가 돌아와야 정상적으로 메세지가 전달됐다고 판단하고 다음 메세지를 전송한다. 
브로커는 리더 브로커가 메세지를 받은 뒤 팔로워 브로커들에게 복제까지 성공하면 ACK를 반환하는데, 프로듀서의 `acks` 설정값에 따라 복제 성공해야 하는 팔로워의 개수가 달라진다.

- acks: 0 일때 프로듀서는 브로커로부터 응답을 기다리지 않고 바로 다음 메세지를 전송한다.
- acks: 1 일때는 리더브로커에만 메세지가 전송되면 바로 프로듀서에게 ACK를 반환한다.
- acks: -1,all 일때는 min.insync.replicas에 설정된 개수만큼의 브로커에 복제가 성공해야 ACK 메세지를 전송한다.

여기서 `min.insync.replicas` 라는 설정값에 대해 좀 더 자세히 알아보자.

![스크린샷 2024-03-27 오전 8 50 44](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/073b536c-3aee-4c23-9e9e-36380b297163)

위의 그림처럼 3개의 브로커가 있다고 가정해보자. 
**replication-factor**가 3이고 **min.insync.replicas** 설정이 2라면 리더 브로커와 팔로워 브로커 총 2개에 데이터가 정상적으로 동기화되야 프로듀서에게 ACK를 반환할 수 있다.

여기서 주의할 점은 **min.insync.replicas**를 계산할때는 리더 브로커도 포함해야 한다는 것이다. 
만약 이 상황에서 레플리카 2개가 모두 죽어버리고 리더 브로커만 살아있다면 ACK 조건을 충족하지 못하기 때문에 브로커는 프로듀서에게 **NOT_ENOUGH_REPLICAS** 예외를 반환하게 된다.

## 결론

카프카 운영 시 싱글 브로커를 사용하는 곳은 거의 없을 것이다. 가용성을 위해서 멀티 브로커 환경을 구축하게 될 것이고, 안정적으로 운영하기 위해서는 ISR에 대해 자세히 알아야 한다고 생각한다. 이번 기회를 통해 멀티 브로커 환경에 조금 더 익숙해 질 수 있는 기회였다 생각한다.


[nodejs]: https://nodejs.org/
[starter]: https://github.com/cotes2020/chirpy-starter
[pages-workflow-src]: https://docs.github.com/en/pages/getting-started-with-github-pages/configuring-a-publishing-source-for-your-github-pages-site#publishing-with-a-custom-github-actions-workflow
[latest-tag]: https://github.com/cotes2020/jekyll-theme-chirpy/tags