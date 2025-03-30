---
title: 카프카 세그먼트 관리
author: jemlog
date: 2024-03-24 00:20:00
categories: [Kafka]
tags: [kafka]
pin: false
image:
  path: '/assets/img/kafka.webp'
---

카프카는 분산 이벤트 스트리밍 플랫폼으로써 실시간으로 대량의 이벤트를 처리합니다. 
이때 이벤트는 모두 디스크에 저장되서 토픽을 구독하는 모든 컨슈머가 이벤트를 소비할 수 있는 일대다 모델을 가지고 있는데, 
그렇다면 카프카는 이 실시간으로 축적되는 이벤트를 어떻게 디스크에서 관리하고 있는지 궁금해졌습니다. 
이번 포스팅에서는 카프카가 디스크에서 데이터를 관리하는 방식에 대해 알아보고자 합니다.

## 카프카 파티션 구조 분석

카프카에서 메세지를 저장하고 소비하는 기본 단위는 파티션이다. 이 파티션이 내부적으로 메세지를 어떻게 관리하고 있는지 브로커 내부로 접속해서 한번 살펴보자.

<img width="1100" alt="스크린샷 2024-03-29 오후 1 37 14" src="https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/294d3c72-3279-496d-bc87-21ebe347f75c">

`test-topic-0`은 test-topic이라는 토픽에서 첫번째 파티션을 말한다. 
브로커 내에서 파티션은 이렇게 디렉토리로 관리된다. 
폴더 내부에 여러 파일들이 존재하는데 **xx.index, xx.log, xx.timeindex** 파일 이렇게 3개를 집중적으로 살펴보자. 
**leader-epoch-checkpoint** 파일은 멀티 브로커 구성과 연관된 파일이다.

브로커에 메세지가 들어오면 .log 파일에 저장된다. 
이 log 파일은 append-only인 로그 파일이고, **세그먼트**라고 부른다. 
여기서 00000000000000000000.log 같이 긴 숫자가 이름으로 설정되어 있는걸 알 수 있는데, 
해당 세그먼트 파일의 **시작 오프셋**이 몇번인지를 나타내는 것이다. 
예를 들어 이 세그먼트의 메세지가 오프셋 42번부터 시작하면 00000000000000000042.log 라는 이름이 붙게 될 것이다.

그러면 .index 파일의 용도는 무엇일까? 
세그먼트는 **WAL**(Write-Ahead Log)로써 append-only라는 특성을 가지고 있다. 
파일은 데이터 쓰기에는 유리하지만 탐색에는 불리하기 때문에 중간에 특정 오프셋의 데이터를 찾기 위해서는 시작 지점부터 byte 위치를 계산해서 찾아야 한다.
이 방법은 세그먼트의 크기와 찾고자 하는 Offset의 bytes 위치에 따라 비효율적일 수 있다. 
따라서 카프카는 인덱스 파일에 **Offset**과 그 **Offset의 bytes 위치**를 저장해서 탐색에 사용하도록 한다. 

이때 인덱스 파일에는 **모든 메세지의 위치가 저장되지는 않는다**. 
모든 메세지의 위치가 저장되면 인덱스 파일의 크기가 매우 비대해질 것이기 때문이다. 
따라서 카프카는 특정 기준을 만족하는 시점에 그때 이벤트의 위치를 인덱스에 기록하는 방식을 사용한다. 
보통 **log.index.interval.bytes**에 설정된 값만큼의 segment bytes가 만들어질때마다 해당 offset에 대한 byte position 정보를 기록한다.

마지막으로 .timeindex 파일은 메세지가 추가된 Unix 타임스탬프와 해당 메세지의 Offset이 저장되는 구조이다. 따라서 실제 bytes 위치를 알기 위해서는 한번 더 검색하는 과정이 필요하다.

## 카프카 세그먼트 롤업

만약 하나의 세그먼트 파일에 모든 메세지가 저장된다면 파일 하나의 크기가 매우 커질것이다. 따라서 카프카는 일정 조건 충족 시 세그먼트 파일을 분리하는 작업을 거치는데 이를 **롤업** 이라고 한다. 롤업과 관련된 설정들이 어떤게 있는지 살펴보자.

- **log.segment.bytes**
  - 개별 segment의 최대 크기
  - default는 1GB
  - 해당 크기를 넘기면 segment는 active segment가 아니게 된다.
- **log.roll.hours**
  - 개별 segment가 유지되는 최대 시간
  - default는 7일
  - 해당 시간 넘기면 해당 세그먼트는 더이상 active가 아니게 된다.
  - log.segment.bytes만큼 데이터가 차지 않아도 이 시간 지나면 close된다

### 세그먼트의 종류와 생명주기

카프카의 세그먼트는 일정한 생명 주기를 가진다. 
처음에 **Active** 상태로 시작해서 세그먼트에 데이터를 쓸 수 있다. 
그러다가 위에서 살펴본 조건에 부합해서 롤업이 일어나면 Active 상태에서 **Close** 상태로 변경된다.
이때부터는 해당 세그먼트에는 데이터를 추가하는게 불가능하다. 
이런 특성 때문에 하나의 파티션에 여러 세그먼트가 있다고 해도 데이터를 쓸 수 있는 **Active 세그먼트는 가장 최근에 추가된 세그먼트 단 하나이다**.

## 세그먼트 삭제 정책

카프카는 분산 이벤트 스트리밍 플랫폼으로써 수많은 메세지를 실시간으로 세그먼트에 적재하게 된다. 
그만큼 관리해야 하는 세그먼트의 개수와 크기가 빠르게 증가해서 차후에는 용량을 크게 차지할 수도 있다.
그리고 카프카의 이벤트 데이터는 보통 최신성이 더 중요하기 때문에 과거의 데이터를 조회할 경우가 적다고 생각된다. 
따라서 카프카는 과거의 세그먼트들을 어떻게 관리할지에 대한 2가지 정책이 존재하는데, **delete** 정책과 **compaction** 정책이 있다. 
두가지 방법을 혼합해서 사용할 수도 있다.

먼저 세그먼트를 완전히 삭제하는 전략에 대해 알아보자. `log.cleanup.policy`를 **delete**로 설정하면 사용할 수 있다. 
브로커 레벨의 설정들을 통해 어느 주기마다 삭제를 진행하는지 관리할 수 있다. 어떤 설정들이 있는지 살펴보자.

- **log.retention.hours**
  - 개별 segment가 삭제되기 전에 유지되는 시간. 기본은 1주일이다
  - 이 시간이 크게 설정되면 디스크 공간이 많이 필요하고, 작게 설정하면 과거 이벤트를 조회하기 힘듦
- **log.retention.bytes**
  - segment 삭제 조건이 되는 파일의 전체 크기 서정
  - default는 무한대이다. 공간 효율적 사용을 위해 보통 설정하는게 좋다
- **log.retention.check.interval.ms**
  - 브로커가 백그라운드로 segment 삭제 대상을 찾기 위한 주기

## 세그먼트 컴팩션 정책

앞에서는 세그먼트를 완전히 삭제하는 정책을 알아봤다. 이번에는 컴팩션 정책에 대해 알아보자.
컴팩션 정책은 메세지의 Key들을 기준으로 가장 최신의 데이터만 보존해서 압축하는 방식을 사용한다.
컴팩션 작업은 몇가지 과정을 거쳐서 실행되는데 하나씩 살펴보자.

![스크린샷 2024-03-28 오후 11 04 12](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/f89ce5d8-16c6-4160-8c16-c7849c339855)

로그 세그먼트 중 현재 메세지가 실시간으로 쌓이고 있는 세그먼트를 **active segment**라고 한다. 
해당 세그먼트에서는 컴팩션 작업이 수행되지 않는다. 
컴팩션 작업은 오래된 세그먼트부터 수행 대상이 되는데,
가장 최근에 clean 상태가 된 세그먼트들은 제외하고 바로 그 앞의 세그먼트부터 청소 대상이 된다. 이때 이 세그먼트들을 **dirty segment**라 부르고 가장 오래된 dirty segment를 **first dirty segment**라고 부른다.

first dirty segment부터 시작해서 어느 dirty segment까지 삭제 범위에 포함할지 결정해야 한다. 이때 선택 기준은 log cleaner 쓰레드가 어느 정도의 메모리를 할당받을 수 있는지 여부이다.

![스크린샷 2024-03-28 오후 11 17 53](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/5ef840da-53ab-47b5-aa16-a47371836c16)

Log 컴팩션 시 이벤트가 살아남는 조건은 같은 Key 기준으로 가장 최근의 값이여야 한다는 것이다. 
삭제 범위를 정한 후에는 **Dirty Segments Map**이라는걸 만든다. 
이 Map에는 Key 별로 가장 최근 데이터의 Offset을 저장한다. 
first dirty segment부터 시작해서 Map에 Key 별 가장 최근 값을 업데이트 하는 과정을 거친다.

![스크린샷 2024-03-28 오후 11 22 27](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/97b0b9b8-64a1-4c8e-a7b3-bad74beba458)

그 다음부터는 기존에 컴팩션이 완료됐던 세그먼트들도 모두 포함해서 순회하는 과정을 거친다.
이벤트의 Key를 확인 후 Map에 들어있는 Value인 Offset보다 오래된 데이터라면 모두 삭제해버린다.

![스크린샷 2024-03-28 오후 11 24 28](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/1b813565-7af0-4271-96ca-bd902f9b4483)

기존 세그먼트들에서 Key별 최신 Offset의 이벤트들만 남겼다면 그 이벤트들만으로 컴팩션 작업을 통해 새로운 세그먼트에 복사한다. 
이때 만약 두 개 이상의 세그먼트를 정리할때 토픽의 `segment.bytes` 보다 크기가 작다면 하나의 세그먼트에 압축해서 복사할 수 있다. 

![스크린샷 2024-03-28 오후 11 31 36](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/d1cb2366-5582-4228-94a9-532a59b7e4a7)

새로운 세그먼트에 컴팩션 작업이 끝났다면 기존의 세그먼트들은 삭제해버린다. 
새로운 세그먼트는 이제 previously cleaned segment가 되고, 바로 뒤에 있는 dirty segment는 first dirty segment가 된다. 
컴팩션 과정은 이 절차를 계속해서 반복한다.

### 로그 컴팩션이 되는 기준

그렇다면 로그 컴팩션은 언제 발생하는걸까? 브로커 레벨의 설정값들을 통해 조절할 수 있는데 한번 알아보자.

- **log.cleaner.min.cleanable.ratio**
  - 로그 클리너가 컴팩션 작업을 수행하기 위한 파티션 내 전체 데이터 중 dirty 데이터 비율
  - default는 50% 이고, 이 값이 작을수록 더 빈번한 컴팩션 작업이 발생한다
- **log.cleaner.min.compaction.lag.ms**
  - 메세지 생성 후에 최소 log.cleaner.min.compaction.lag.ms가 지나야 컴팩션 대상이 된다
  - default는 0ms
- **log.cleaner.max.compaction.lag.ms**
  - dirty ratio 이하여도 메세지 생성 후 log.cleaner.max.compaction.lag.ms 지나면 컴팩션 대상이 된다
  - default는 무한대

3가지 설정값 중 하나만 충족한다고 로그 컴팩션이 실행되지 않고, 2가지 경우 중 하나를 충족해야 한다.
- log.cleaner.min.cleanable.ratio를 넘길 만큼 dirty 데이터가 생성됐고, log.cleaner.min.compaction.lag.ms가 경과해야 한다.
- log.cleaner.max.compaction.lag.ms를 경과해야 한다.

### Tomstone과 Transaction Marker 관리

Tomstone과 Transaction 마커가 있는 경우에는 특별한 처리를 해줘야 한다. 클라이언트 어플리케이션이 이것들을 놓치면 안되기 때문이다.

![스크린샷 2024-03-29 오전 9 33 35](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/3f1588a6-2393-46f9-9271-5cc50ea931ae)

몇가지 과정을 거쳐서 이를 보장해줄 수 있다.

1. Clean Up 작업을 하는동안 Tomstone이나 Transaction Marker를 마추치면 그 이벤트들에 **to-delete-timestamp**를 함께 마킹한다. 이는 처음 Clean Up이 시도되는 시각에 delete.retention.ms 값을 더한 것이다. 이후 삭제 작업은 하지 않기 때문에 클라이언트들이 이 이벤트들을 사용 가능하다
2. 다음 Clean Up 루틴이 돌때 Log Cleaner는 to-delete-timestamp 시간이 경과했는지 체크한다. 이때 to-delete-timestamp가 만료된 tomstone이나 Transaction Marker들은 제거하는 작업을 수행한다. 

위의 과정을 통해 클라이언트들을 이 이벤트들을 안전하게 처리 가능하다.

## 결론

카프카는 전형적인 메세지큐와 다르게 이벤트를 모두 디스크에 저장해서 관리함으로, 여러 컨슈머가 메세지를 소비할수도 있고 메세지 처리를 재시도할 수도 있다. 
실시간으로 데이터셋이 급증하는 만큼 내부적으로 메세지를 어떻게 저장하는지에 대한 궁금함이 있어왔다.
이번 기회를 통해서 그 원리를 자세히 알아볼 수 있는 좋은 기회였다 생각한다.

## Reference

- [https://developer.confluent.io/courses/architecture/compaction/](https://developer.confluent.io/courses/architecture/compaction/)

[nodejs]: https://nodejs.org/
[starter]: https://github.com/cotes2020/chirpy-starter
[pages-workflow-src]: https://docs.github.com/en/pages/getting-started-with-github-pages/configuring-a-publishing-source-for-your-github-pages-site#publishing-with-a-custom-github-actions-workflow
[latest-tag]: https://github.com/cotes2020/jekyll-theme-chirpy/tags
