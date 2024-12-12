---
title: 카프카 트랜잭션 이해하기
author: jemlog
date: 2024-02-25 00:20:00
categories: [Kafka]
tags: [kafka]
pin: false
image:
  path: '/assets/img/kafka.webp'
---

이번 포스팅에서는 카프카의 트랜잭션에 대해 알아보고자 합니다. 저희가 흔히 사용하는 DB 트랜잭션의 경우 여러 DB 작업을 하나의 원자적 단위로 묶어주는 역할을 합니다. 카프카를 사용할때도 여러 이벤트 발행 작업이 하나의 원자적인 단위로 묶일 필요가 있는데, 이때 카프카 트랜잭션을 사용 가능합니다.

## 카프카 작업 간의 트랜잭션을 통한 원자성 보장

### 문제가 될 수 있는 상황

카프카 트랜잭션이 사용될 수 있는 케이스를 학습해보자. Kafka Confluent 사이트에 좋은 예시가 있어서 참고해서 진행해보겠다.

A가 B에게 송금하는 플로우를 고려해보자. 송금을 진행하는 어플리케이션에는 송금 이벤트를 컨슈밍하는 컨슈머와 A와 B의 계좌 상태를 업데이트 하는 프로듀서가 있다.

<img width="514" alt="스크린샷 2024-03-26 오전 11 22 26" src="https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/d9948541-ab00-4dba-aee0-3934e7417171">

이때 만약 A에게 계좌 금액을 더하는 프로듀서는 정상적으로 동작해서 브로커에 메세지를 저장했지만, B에게 계좌 금액 차감을 전달하는 프로듀서를 정상적으로 메세지를 전달하지 못하고 어플리케이션이 다운 되는 상황을 가정해보자.

아직 컨슈머가 오프셋을 커밋하지 못한 상태이기 때문에 새로 동작하는 어플리케이션은 다시 컨슈머가 동작할때 이전에 읽었던 오프셋을 다시 읽어온다. 프로듀싱 작업 또한 다시 실행될 것이고, A에게 계좌 금액을 더하는 메세지는 한번 더 전송되게 될 것이다.

Balance 토픽을 구독하는 컨슈머 입장에서는 두 개의 계좌 출금 이벤트가 파티션에 들어있기 때문에 이벤트를 중복 처리 하게 된다. 이렇게 두 개의 프로듀서 작업 간 원자성이 보장되지 않는 문제를 해결하기 위해 카프카 트랜잭션을 사용할 수 있다.

### 카프카 트랜잭션

카프카 트랜잭션은 프로듀서 설정에서 활성화 할 수 있다. 프로듀서 설정에서 `transactional.id`라는 항목을 설정하면 된다.

```kotlin
@EnableKafka
@Configuration
class KafkaProducerConfig {

    @Bean
    fun producerFactory(): ProducerFactory<String, Long> {
        val props = HashMap<String, Any>()
        props[ProducerConfig.TRANSACTIONAL_ID_CONFIG] = "tx-advance"
        return DefaultKafkaProducerFactory(props)
    }
}
```

이 값을 설정하고 나면 브로커에 **트랜잭션 코디네이터**라는게 생성된다.  이 코디네이터는 트랜잭션 메타데이터를 관리하고 전체 트랜잭션 플로우를 관리한다.

### 트랜잭션이 중간에 문제가 생기는 경우

만약 카프카 트랜잭션을 설정한 상태에서 문제가 생기면 어떤 플로우가 진행될까. 아래의 그림을 살펴보자.

![스크린샷 2024-03-26 오전 9 54 30](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/2a2eb84d-fdc9-41df-a8b6-dfb662e31694)

1. 컨슈머 어플리케이션은 초기에 트랜잭션 코디네이터에게 `transactional.id`와 함께 초기화 요청을 보낸다. 트랜잭션 코디네이터는 PID와 transaction epoch에 transactional.id를 매핑하고 다시 어플리케이션에게 돌려준다.
2. 컨슈머가 transfer 토픽으로부터 메세지를 읽어온다.
3. 프로듀서가 debit 이벤트를 balance 토픽에 발행하기 전에, 그 사실을 트랜잭션 코디네이터에게 알린다.
4. 이벤트를 balance 토픽에 기록한다.

이때 만약 컨슈머 오프셋을 커밋하기 전에 어플리케이션이 다운되고 재시작되는 경우를 고려해보자. 트랜잭션이 없는 상황에서는 Balance 토픽의 데이터를 중복 소비하게 되는 문제가 있었다. 트랜잭션을 사용하는 상황에서는 어떻게 해결되는지 살펴보자.

1. 어플리케이션이 재시작하면 다시 transactional.id와 함께 초기화 요청을 보낸다. 이때 트랜잭션 코디네이터는 pending중인 이전 트랜잭션이 있다는걸 확인하고 해당 트랜잭션을 취소한다. 그때 메세지를 전송했던 balance 토픽의 파티션에는 `Abort` 마커를 추가한다. 이후 새로운 트랜잭션을 시작한다.
2. Balance 토픽을 구독하는 컨슈머 어플리케이션이 `isolation.level=read_commited`로 설정되어 있다면 Abort 마커 이전의 메세지는 무시하고 새롭게 들어온 메세지만 읽게 된다.

정리하자면, `read_commited` 설정값을 가진 컨슈머는 이전 트랜잭션 취소에 의해 Abort된 메세지는 무시해버리는 방식으로 메세지의 중복 소비를 방지한다. 이를 통해 트랜잭션 범위 내의 여러 프로듀서들 간의 원자성도 보장할 수 있다.

### 트랜잭션이 정상적으로 수행된 경우

그렇다면 트랜잭션이 정상적으로 수행된 경우를 살펴보자.

<img width="515" alt="스크린샷 2024-03-26 오전 10 11 24" src="https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/cc524c03-0efa-486b-a6f5-d796496ff6ab">

1. 컨슈머 어플리케이션은 초기에 트랜잭션 코디네이터에게 `transactional.id`와 함께 초기화 요청을 보낸다. 트랜잭션 코디네이터는 PID와 transaction epoch에 transactional.id를 매핑하고 다시 어플리케이션에게 돌려준다.
2. 컨슈머가 transfer 토픽으로부터 메세지를 읽어온다.
3. 프로듀서가 debit 이벤트를 balance 토픽에 발행하기 전에, 그 사실을 트랜잭션 코디네이터에게 알린다.
4. 계좌 출금 이벤트를 Balance 토픽의 파티션 P0에 전송한다
5. 계좌 입금 이벤트를 Balance 토픽의 파티션 P1에 전송한다
6. 프로듀서 작업이 성공적으로 종료된 상황에서 프로듀서가 직접 __consumer_offset을 커밋한다. 프로듀서 작업이 완료됐다는걸 보장하기 위해 트랜잭션 프로듀서가 직접 작업을 수행한다.
7. 트랜잭션이 완료됐다는 사실이 브로커의 트랜잭션 코디네이터에게 전달된다.
8. 트랜잭션 코디네이터는 __transaction_state 토픽에 커밋 마커를 찍고, 프로듀서가 메세지를 발행한 파티션(P0, P1)과 __consumer_offset 파티션들에도 커밋 마커를 찍는다.
9. Balance 토픽을 구독하는 컨슈머 어플리케이션은 `read_commited`로 설정되있다면 파티션의 커밋 마커가 찍힌 데이터만 읽는다.

위의 과정을 거쳐서 카프카는 하나의 트랜잭션 내에서 동작하는 프로듀서들과 컨슈머의 __consumer_offset 커밋 작업까지 원자적으로 묶을 수 있게 된다.

## 카프카와 DB 트랜잭션 간의 원자성 보장

위에서는 카프카 작업 간의 원자성을 보장해주는 카프카 트랜잭션의 역할에 대해 알아봤다. 아마 카프카를 사용하면 DB 트랜잭션과 함께 사용하는 경우가 많이 있을 것이다. 이번에는 카프카 트랜잭션과 DB 트랜잭션이 어떻게 묶여서 사용되는지 살펴보자. 

Spring을 사용하면 @KafkaListener를 통해서 메세지를 컨슈밍할 수 있다. 이때 @KafkaListner가 붙은 메서드 범위 내에서 카프카 트랜잭션과 함께 DB 트랜잭션을 사용하려면 어떻게 해야할까?
사용법은 간단하다. 컨슈머 메서드 위에 똑같이 @Transactional 어노테이션을 붙여주기만 하면 된다.

```kotlin
@Transactional
@KafkaListener
fun save(String message){
   // ... 작업 처리
}
```

이때 카프카 트랜잭션과 DB 트랜잭션의 커밋 순서는 어떻게 될까?

DB 트랜잭션이 먼저 커밋된 다음 카프카 트랜잭션이 커밋된다. 여기서 고려해봐야 하는 부분은 DB 트랜잭션은 커밋이 됐는데 카프카 트랜잭션이 커밋에 실패해서 재시도를 해야 하는 경우다. 만약 멱등성이 보장되지 않는 DB 연산이 수행된다면 데이터가 중복 처리 될 가능성이 있다.
따라서 이 경우에는 데이터 저장의 멱등성을 별도로 보장해야 한다.

대표적으로 메세지에 DB에서 유니크 키로 사용할 수 있는 값을 담아서 전송하는 멱등키 방식이 있다. 데이터 생성 일자를 나타내는 타임스탬프 같은 값을 유니크 키로 사용하면 데이터 중복 저장을 방지할 수 있을 것이다.

또 다른 방법으로는 Upsert(데이터가 없을 경우 Insert, 아니면 Update) 연산을 지원하는 DB라면 이 방식을 사용해 데이터 중복 저장을 방지할 수도 있을 것이다.

### 카프카 프로듀서와 DB 트랜잭션 묶기

위의 경우는 카프카 컨슈머와 DB 트랜잭션을 묶는 경우를 살펴봤다. 이번에는 프로듀서와 DB 트랜잭션을 묶는 경우를 살펴보자.

이 경우는 간단하다. 그냥 `@Transactional` 어노테이션만 붙여주면 알아서 카프카 프로듀서의 트랜잭션도 DB 트랜잭션과 연계되서 수행된다.
```kotlin
@Transactional
public void someMethod(String in) {
    this.jdbcTemplate.execute("insert into mytable (data) values ('" + in + "')");
    this.kafkaTemplate.send("topic2", in.toUpperCase());
}
```

이 경우에도 컨슈머 때와 같이 DB 트랜잭션이 먼저 커밋되고 카프카 트랜잭션이 이후에 커밋된다. 만약 카프카 트랜잭션을 먼저 커밋하고 싶다면 어떻게 해야 할까? 아래와 같이 중첩 구조를 사용해서 DB 트랜잭션과 카프카 트랜잭션을 분리하면 된다.

```kotlin
@Transactional("dstm") 
public void someMethod(String in) {
    this.jdbcTemplate.execute("insert into mytable (data) values ('" + in + "')"); 
    sendToKafka(in); 
} 
@Transactional("kafkaTransactionManager") 
public void sendToKafka(String in){ 
    this.kafkaTemplate.send("topic2", in.toUpperCase()); 
}
```

## 결론

지금까지 카프카 트랜잭션의 동작 방식과 스프링 카프카의 트랜잭션을 DB 트랜잭션과 연계하는 방법을 알아봤다. MSA 환경에서는 하나의 작업 단위 내에서 여러 이벤트를 발행하는 일이 많을 것이다. 그만큼 이벤트 발행 간의 원자성 보장 또한 중요하다고 본다. 
실제 카프카를 사용해서 프로젝트를 진행하며 카프카 트랜잭션을 사용해야 하는 여러 케이스들을 경험해보고 싶다.

## Reference

- [https://developer.confluent.io/courses/architecture/transactions/](https://developer.confluent.io/courses/architecture/transactions/)
- [https://docs.spring.io/spring-kafka/reference/kafka/transactions.html](https://docs.spring.io/spring-kafka/reference/kafka/transactions.html)
- [https://www.youtube.com/watch?v=7_VdIFH6M6Q](https://www.youtube.com/watch?v=7_VdIFH6M6Q)


[nodejs]: https://nodejs.org/
[starter]: https://github.com/cotes2020/chirpy-starter
[pages-workflow-src]: https://docs.github.com/en/pages/getting-started-with-github-pages/configuring-a-publishing-source-for-your-github-pages-site#publishing-with-a-custom-github-actions-workflow
[latest-tag]: https://github.com/cotes2020/jekyll-theme-chirpy/tags
