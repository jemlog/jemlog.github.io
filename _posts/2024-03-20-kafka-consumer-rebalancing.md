---
title: 카프카 컨슈머 리밸런싱
author: jemlog
date: 2024-03-20 00:20:00
categories: [Kafka]
tags: [kafka]
pin: false
image:
  path: '/assets/img/kafka.webp'
---

카프카는 처리량 조절을 위해 컨슈머 어플리케이션 개수를 유동적으로 조절할 수 있다. 이때 컨슈머 개수가 변동될때마다 각각의 컨슈머가 할당받는 파티션도 변경되는데, 이를 컨슈머 리밸런싱이라고 한다. 이번 글에서는 컨슈머 리밸런싱에 대해서 알아보고자 한다. 

### 컨슈머 리밸런싱이 발생하는 경우

우선 어떤 경우에 리밸런싱이 발생하는지 알아보자. 크게 2가지 경우에 리밸런싱이 발생한다.

- Consumer Group에 컨슈머가 추가/삭제 되는 경우
- 브로커에 파티션이 추가되는 경우

만약 리밸런싱이 필요한 상황이 발생하면 브로커의 GroupCoordinator가 Consumer Group의 리더 컨슈머에게 리밸런싱 명령을 내린다. 이때 리밸런싱이 발생하는 동안에는 일시적으로 STW(Stop The World)가 발생하는데, 카프카 프로듀서는 그 순간에도 브로커로 계속 메세지를 
전송한다. 따라서 컨슈머는 리밸런싱 이후에 최신 데이터를 따라가는 작업을 수행해야 하는데, 이때문에 리밸런싱이 잦아지면 성능상 좋지 않다.

![스크린샷 2024-03-27 오전 8 04 44](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/91facb6f-162d-4832-b676-1df7e0cfb151)

### 리더 컨슈머 선출과 파티션 분배 과정

리밸런싱은 컨슈머 그룹 내에서 이뤄진다. 이때 컨슈머 그룹에는 파티션 할당을 담당하는 리더 컨슈머가 할당된다. 리더 컨슈머가 어떤 과정으로 선출되고 어떤 역할을 하는지 알아보자.

1. Consumer Group의 컨슈머가 브로커에 처음 접속 요청 시 Group Coordinator가 생성된다.
2. 가장 빨리 그룹에 조인한 컨슈머를 리더 컨슈머로 지정한다.
3. 리더 컨슈머는 파티션 할당 전략에 따라 컨슈머들에게 파티션을 할당한다
4. 할당을 마친 후에는 할당된 파티션 정보를 Group Coordinator에게 전달한다
5. 정보 전달 하고 난 뒤에 개별 컨슈머들이 할당된 파티션에서 데이터를 읽는다.

### HeartBeat Thread

HeartBeat 스레드는 컨슈머가 최초로 poll()을 호출할때 생성된다. 여기서 알아두면 좋은 점은 컨슈머는 최초 poll()에서는 메세지를 가져오지 않는다는 점이다. 최초 poll()에서는 브로커와 메타데이터를 교환하고 HeartBeat 스레드를 생성하는 작업을 한다.

HeartBeat 스레드는 일정 시간마다 브로커에게 HeartBeat를 전송한다. 이때 설정한 제한 시간 내에 HeartBeat가 오지 않으면 브로커는 컨슈머가 죽었다 판단하고 컨슈머 리밸런싱을 명령한다. HeartBeat 스레드와 관련된 설정값들이 어떤게 있는지 알아보자.

- **heartbeat.interval.ms** (default 3s)
  - HeartBeat 스레드가 heartbeat를 전송하는 간격이다. session.timeout.ms 보다 낮게 설정되어야 하며 보통 3분의 1 정도를 권장한다
- **session.timeout.ms** (default 45s)
  - 브로커가 개별 컨슈머로부터 HeartBeat가 오기를 기다리는 시간이다. 이 시간이 경과하면 컨슈머가 죽었다 판단하고 리밸런싱을 명령한다
- **max.poll.interval.ms** (default 300s)
  - 컨슈머가 poll()을 한번 호출한 후 다음 poll()을 호출하기까지의 시간
  - 제한시간 안에 poll()을 호출하지 않으면 브로커는 컨슈머가 죽었다 판단하고 리밸런싱을 명령한다


### Static Group Membership

리밸런싱이 발생하면 컨슈머가 맡고 있던 파티션이 계속 변경된다. 이때 만약 컨슈머가 죽었다 재기동 되더라도, 일정 시간 안에만 살아났다면 처음에 할당 받았던 파티션을 그대로 유지하고 싶다면 어떻게 해야 할까? 이를 위해서 **Static Group Membership**이라는게 있다.

```kotlin
@Configuration
class KafkaConsumerConfig {

    @Bean
    fun consumerFactory(): ConsumerFactory<String, Long> {
        val props: MutableMap<String, Any> = HashMap()
        props[ConsumerConfig.GROUP_ID_CONFIG] = "consumer-group-1"
        props[ConsumerConfig.GROUP_INSTANCE_ID_CONFIG] = "1" // 해당 설정으로 사용 가능
        return DefaultKafkaConsumerFactory(props)
    }
}
```

위와 같이 설정하면 해당 컨슈머는 파티션 1번을 초기에 할당 받을 수 있다. 이때 여러 컨슈머 어플리케이션이 같은 그룹 인스턴스 아이디를 설정하면 에러가 발생한다. 하나의 컨슈머 인스턴스마다 고유한 아이디 값이 할당되는 것이다. 
만약 컨슈머가 죽었더라도 **session.timeout.ms** 안에만 살아난다면 같은 파티션을 할당 받을 수 있다.

![스크린샷 2024-03-27 오전 7 56 03](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/0ff8286e-308c-4111-8c8e-694658b42642)

## 컨슈머 리밸런싱 모드

마지막으로 컨슈머가 리밸런싱을 진행할때 어떤 방식을 사용해서 진행하는지 살펴보자. 그다음에 크게 2가지 모드에 따라 재할당을 진행한다. Eager 모드와 Cooperative 모드가 있다.

### Eager Mode

Eager Mode는 컨슈머 리밸런싱을 진행할때 기존의 파티션과 컨슈머 간의 모든 연결을 해제하고 새롭게 재할당하는 과정을 거친다. 재할당이 진행되는 동안 어떤 컨슈머도 파티션에서 메세지를 소비하지 못하기 때문에 컨슈머 그룹에 참여한 컨슈머가 많을 경우 성능이 떨어질 수 있다.
Eager Mode에도 몇가지 종류가 있는데 하나씩 알아보도록 하자.

#### Range

- default로 사용되는 방식이다
- 서로 다른 2개 이상의 토픽을 Consumer들이 구독할때 토픽별로 동일한 파티션을 Consumer에 할당
- 여러 토픽에서 동일한 Key로 되어 있는 파티션을 동일 Consumer에 할당해서 데이터 처리를 용이하게 해준다

![스크린샷 2024-03-27 오전 7 36 43](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/056a416c-f4f2-404a-bdd7-18d1f7c49661)


#### Round Robin

- 균등하게 분배한다.
- 파티션의 개수와 Consumer의 개수가 달라지면 같은 Consumer로 같은 Key의 메세지가 들어간다는 보장은 없다.

![스크린샷 2024-03-27 오전 7 36 43](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/056a416c-f4f2-404a-bdd7-18d1f7c49661)

#### Sticky

- 최초 할당된 파티션과 Consumer의 매핑을 기억해두었다가 리밸런싱이 수행될때 원래의 매핑을 유지하도록 구현

![스크린샷 2024-03-27 오전 7 44 30](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/8a15c60c-bce1-47aa-80a3-edd2c46585bf)

### Cooperative Mode

리밸런싱이 필요한 일부를 위해서 전체 할당을 해제한다는건 비효율적인 면이 있어보인다. 이를 해결하기 위해 Cooperative Mode가 등장했는데, Eager 모드와는 다르게 영향을 받는 파티션과 컨슈머의 연결만 해제 후 재연결한다.
많은 컨슈머를 가진 컨슈머 그룹에서 리밸런싱 시간이 오래 걸리는 경우 좋은 방법이다.

#### Cooperative Sticky

- 기본적으로 Sticky 전략을 따르지만 리밸런싱 시 전체 매핑을 취소하지 않고 변경된 부분만 재연결 시도한다.

## 결론

트래픽에 따라 유연하게 스케일아웃이 가능한 최근 환경에서 컨슈머의 추가/삭제로 인한 리밸런싱 작업 또한 빈번하게 발생할 것이다. 리밸런싱이 어떤 경우에 발생하고 어떤 기준으로 재할당이 되는지 정확히 알고 있어야 STW 시간이 길어져서 메세지를 효율적으로 소비하지 못하는 현상을
잘 방지할 수 있을 것이다.

[nodejs]: https://nodejs.org/
[starter]: https://github.com/cotes2020/chirpy-starter
[pages-workflow-src]: https://docs.github.com/en/pages/getting-started-with-github-pages/configuring-a-publishing-source-for-your-github-pages-site#publishing-with-a-custom-github-actions-workflow
[latest-tag]: https://github.com/cotes2020/jekyll-theme-chirpy/tags
