---
title: 카프카 내부 최적화
author: jemlog
date: 2024-03-24 00:20:00
categories: [Kafka]
tags: [kafka]
pin: false
image:
  path: '/assets/img/kafka.webp'
---

카프카는 분산 이벤트 스트리밍 플랫폼으로써 대량의 이벤트에 대해 높은 처리량을 제공해주는 것으로 유명해서 많은 기업에서 도입한 솔루션입니다. 
그렇다면 카프카는 내부적으로 어떤 구조를 사용하기에 이렇게 높은 처리량을 소화할 수 있는걸까요? 
오늘은 그 방법에 대해 살펴보고자 합니다.

## 순차 I/O 사용한 성능 최적화

일반적으로 디스크 I/O는 메모리 I/O 보다 엄청나게 느리다고 알려져 있다.
하지만 정확히는 디스크 랜덤 I/O의 속도가 매우 느린 것이다.
순차 I/O를 사용할 경우 RAID로 구성된 현대적 디스크에서 수백 MB/sec 수준의 읽기/쓰기 성능을 달성할 수 있다. 

<img width="583" alt="스크린샷 2024-03-30 오전 10 31 58" src="https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/26d773ac-3f7c-461d-b1b5-c5f369e16ad8">

위의 표를 보면 디스크 순차 I/O의 경우 오히려 메모리 랜덤 I/O 보다 더 높은 성능을 보여주는걸 알 수 있다. 
특히 순차 읽기 & 쓰기의 경우 예측 가능한 사용 패턴을 가지고 있다는 특성때문에 현대 OS에서는 **read-ahead**나 **write-behind** 같은 기술을 사용해서 순차 I/O를 최적화한다.
여기서 read-ahead는 연속된 데이터를 조회할때 다음 데이터도 함께 조회될 꺼라는 예측을 통해 백그라운드로 다음 데이터를 미리 메모리에 로딩하는 기술이다.
그리고 write-behind는 쓰기 작업을 버퍼링 했다가 큰 규모로 한번에 디스크에 쓰는 기술을 말한다.

그렇다면 카프카는 디스크에 데이터를 어떤 구조로 저장을 할까? 
카프카는 **WAL**(Write-Ahead Log)를 사용한다. 
WAL은 새로운 항목이 추가되기만 하는 로그 파일이다. 
WAL에 대한 접근 패턴은 읽기/쓰기 모두 순차적이라는 특징이 있다.
카프카는 이벤트 스트리밍 플랫폼으로써 엄청난 데이터가 실시간으로 증가하게 된다. 
하지만 내부적으로 WAL을 사용하기 때문에 **카프카의 성능을 데이터 사이즈에서 디커플링** 할 수 있다.

이렇게 데이터 저장에 디스크를 적극 사용할 수 있게 되면서 몇가지 장점이 생겼다. 
먼저 비싼 메모리 대신 상대적으로 **저렴한 디스크**를 사용해도 대량의 **읽기 쓰기에 대한 안정적인 성능**을 유지할 수 있게 됐다.
그리고 메모리에 데이터를 일시적으로 저장 후 컨슈머가 소비하면 삭제 해버리는 다른 메세징 시스템과 다르게 메세지를 일정 기간동안 보존해서 여러 컨슈머들이 메세지를 소비할 수도 있고 재시도 처리도 가능해진다.

## OS 페이지 캐시를 사용한 성능 최적화

현대 OS들은 디스크 접근 횟수 자체를 줄이기 위해서 메인 메모리 상에 **디스크 캐싱**을 적극적으로 활용한다. 
때로는 Free 메모리의 대부분을 디스크 캐싱에 사용하기도 한다.
페이지 캐시를 사용하면 대부분의 디스크 읽기와 쓰기가 이 디스크 캐시를 통하게 된다.
카프카는 이 OS 페이지 캐시를 적극적으로 활용한다. 
그렇다면 카프카는 어플리케이션 레벨의 인메모리 캐시는 사용하지 않는 것일까? 
우선 카프카는 JVM 위에서 동작한다. Java 메모리를 사용할때는 두가지 특징이 있다.

- 객체의 메모리 오버헤드는 매우 높기 때문에 종종 데이터 저장을 위해 2배의 메모리를 사용하기도 한다
- 자바의 GC는 힙 내에 데이터가 많을수록 느려진다.

이런 Java 메모리의 특징으로 인해 페이지 캐시를 사용하는게 어플리케이션 레벨의 인메모리 캐시를 사용하는 것보다 효율적이다. 
페이지 캐시는 모든 Free 메모리에 접근할 수 있기 때문에 어플리케이션 인메모리 캐시보다 2배 이상의 용량을 사용할 수 있다.
또한 압축된 바이트 구조로 데이터를 저장할 수 있기 때문에 객체에 비해 또 2배 이상을 저장할 수 있다.

어플리케이션 레벨에서 인메모리 캐시를 사용하면 어플리케이션이 종료되면 캐시 데이터가 휘발된다. 
이렇게 되면 어플리케이션 재시작 시 캐시가 Warm Up 되는 동안의 성능 패널티가 발생한다.
반면 페이지 캐시를 사용하면 어플리케이션 재시작과 무관하게 캐시 데이터가 유지되기 때문에 항상 캐시가 채워진 상태를 유지할 수 있다. 

카프카의 메세지는 하나의 컨슈머만 소비하는 것이 아니라 여러 컨슈머가 같은 메세지를 소비할 수 있다.
만약 디스크 캐싱을 하지 않았다면 모든 컨슈머가 메세지를 요청할때마다 디스크 I/O가 발생할 것이다.
카프카는 페이지 캐시를 적극적으로 사용해서 여러 컨슈머가 디스크의 데이터를 요구할때 처음 한번만 디스크 I/O로 디스크 캐시에 적재하고 나면 다음 컨슈머들은 디스크 접근할 필요 없이 빠르게 캐시에서 조회할 수 있다.

## Zero Copy를 사용한 성능 최적화

카프카는 범용적으로 사용될 수 있는 이벤트 스트리밍 플랫폼인 만큼 디스크에 저장된 데이터를 **네트워크를 통해 다양한 외부 컴포넌트로 전송**해야 한다. 
이때 일반적인 어플리케이션의 경우 디스크에서 네트워크 전송에 이르기 까지 다소 복잡한 과정을 거친다.

![스크린샷 2024-03-30 오후 1 19 12](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/c4bfa71d-ef36-4199-8750-ef2d03a2a1be)

1. 디스크의 파일에서 커널의 Read Buffer로 데이터를 복사한다
2. 커널의 Read Buffer에서 유저 어플리케이션의 Application Buffer로 데이터를 복사한다
3. 커널의 Socket Buffer로 데이터를 복사한다
4. NIC Buffer로 데이터를 복사한다

위의 과정처럼 디스크에서 네트워크로 데이터를 전송하기 위해 **총 4번의 Copy**가 발생해야 한다.
이 중 유저 모드의 Application Buffer로 데이터를 복사하고 다시 Socket Buffer로 복사하는 과정인 2번과 3번 단계는 불필요한 절차다.

<img width="438" alt="스크린샷 2024-03-30 오후 4 50 32" src="https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/f0aa2366-a291-48ce-9f65-430eafeb1fb2">

또한 전체 프로세스에서 유저모드와 커널모드 간의 컨텍스트 스위칭이 몇회 발생했는지 살펴보면 **총 4회**가 발생한걸 알 수 있다. 
유저모드와 커널모드는 사용하는 정보가 완전 다르기 때문에 CPU가 컨텍스트 스위칭을 하는데 오버헤드가 크게 발생한다. 
굳이 유저 모드의 어플리케이션을 거쳐서 복사할 필요성이 없다면 커널 내부에서만 복사 작업이 진행되면 좋지 않을까. Unix, Linux 계열 OS에서 이 방법을 가능하게 하는게 **Zero Copy**라는 기술이다.

Zero Copy를 Java에서 사용하기 위해서는 `transferTo()` 메서드를 사용하면 된다.

```java
public void transferTo(long position, long count, WritableByteChannel target);
```

transferTo() 메서드는 데이터를 File Channel에서 메서드 파라미터로 들어온 WritableByteChannel로 전송한다. 
이때 내부적으로는 Unix와 Linux 기반 OS에서 제공하는 시스템콜인 `sendFile()` 메서드를 사용해서 수행한다.

![스크린샷 2024-03-30 오후 1 20 03](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/3e61f263-5de8-4260-ac4d-3995cf2db7af)

1. 디스크에서 커널 영역의 Read Buffer에 데이터가 복사된다.
2. sendFile() 시스템 콜을 통해 같은 커널 영역의 Socket Buffer에 데이터가 복사된다.
3. NIC Buffer로 데이터가 복사된다.

위의 과정을 거칠때 Context Switching은 어떻게 일어나는지 살펴보자.

<img width="451" alt="스크린샷 2024-03-30 오후 5 03 06" src="https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/425592cb-86bd-4350-904c-5b4d8490a0e2">

처음 카프카에서 transferTo() 메서드를 호출할때 커널 모드로 변환되는 Context Switching이 1회 발생한다.
이후 sendFile() 시스템 콜을 통해 디스크에서 커널의 Read Buffer로 DMA 엔진을 사용해 읽은 뒤, Socket Buffer로 데이터를 복사하는 작업까지 수행된다.
다음으로는 DMA 엔진이 Socket Buffer에 저장된 데이터를 NIC Buffer로 복사한다. 작업이 종료된 다음에는 유저 모드로 다시 전환되는 Context Switching이 1회 더 발생한다.
이 작업을 통해서 전통적인 작업에서는 총 4회의 Context Switching이 발생하던걸 2회로 줄였다. 또한 총 4번의 데이터 Copy 작업도 3번으로 줄였다. 

근데 자세히 보면 커널의 Read Buffer에서 Socket Buffer로 데이터를 복사하는 작업도 불필요해 보인다. 아예 Read Buffer가 NIC Buffer로 데이터를 직접 보내줄 수는 없을까. 
만약 NIC가 gather 연산을 지원한다면 더 최적화 가능하다.
리눅스 커널 2.4 버전 이후부터 Socket Buffer Descriptor가 이 요구사항이 가능하도록 변경됐다.

<img width="378" alt="스크린샷 2024-03-30 오후 5 16 24" src="https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/4e568585-f4c4-4649-a3d9-bf66260fec80">

처음에 DMA 엔진이 파일을 읽어서 커널의 Read Buffer에 넣는건 똑같다. Socket Buffer에는 데이터의 위치와 길이를 나타내는 디스크립터들만 추가된다.
DMA 엔진은 데이터를 Read Buffer에서 곧바로 NIC Buffer로 보내버리기 때문에 CPU를 사용하는 복사 작업을 제거했다.

<img width="582" alt="스크린샷 2024-03-30 오후 5 27 51" src="https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/b6ad1632-d057-4d4c-ba9f-3c4f9a66edc3">

전송하려는 파일의 크기와 전송 시간 간의 상관관계를 나타낸 표를 가져왔다.
데이터 규모와 상관없이 Zero Copy 방식이 일반 파일 전송 방식보다 최소 2배 이상의 성능을 내는걸 알 수 있다.

## 결론

분산 이벤트 스트리밍 플랫폼으로써 높은 처리량을 보장해야 하는 카프카가 디스크에 메세지 세그먼트를 저장하는 방식을 사용하면서 어떻게 성능 최적화를 하는지 궁금했었다. 
그리고 카프카는 이를 해결하기 위해 시스템 내부적으로 다양한 최적화 과정을 거쳤다는걸 알 수 있었다. 
이번 기회를 통해 카프카 API를 사용하는걸 넘어 카프카 자체에 대한 이해도를 조금 더 높힐 수 있는 기회였다고 생각한다.

## Reference

- [https://docs.confluent.io/kafka/design/file-system-constant-time.html](https://docs.confluent.io/kafka/design/file-system-constant-time.html)
- [https://developer.ibm.com/articles/j-zerocopy/](https://developer.ibm.com/articles/j-zerocopy/)


[nodejs]: https://nodejs.org/
[starter]: https://github.com/cotes2020/chirpy-starter
[pages-workflow-src]: https://docs.github.com/en/pages/getting-started-with-github-pages/configuring-a-publishing-source-for-your-github-pages-site#publishing-with-a-custom-github-actions-workflow
[latest-tag]: https://github.com/cotes2020/jekyll-theme-chirpy/tags
