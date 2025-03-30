---
title: 카프카 프로듀서 내부 동작 흐름 분석
author: jemlog
date: 2024-02-17 00:20:00
categories: [Kafka]
tags: [kafka]
pin: false
image:
  path: '/assets/img/kafka.webp'
---

카프카의 프로듀서를 학습하는 과정에서 내부 동작 흐름을 디버깅 해본 기록을 남기고자 합니다.

## 해당 글에서 다루고자 하는 카프카 프로듀서의 구조

이번 글에서 다루고자 하는 부분을 먼저 거시적으로 살펴보고 가자. 카프카 프로듀서에서 생성된 메세지는 Serializer와 Partitioner를 거쳐 RecordAccumulator에 batch 단위로 적재된다.

<img width="848" alt="스크린샷 2024-02-18 오후 11 17 46" src="https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/ce913c77-a3e3-4143-9d92-8d1b0aef87cf">

## Serializer

카프카 프로듀서는 브로커로 메세지를 전송할때 **Byte 배열 형태로 직렬화** 해서 네트워크를 통해 전송한다. 따라서 프로듀서의 send() 메서드를 호출하면 가장 먼저 수행되는 작업은 메세지 Key와 Value의 직렬화 작업이다.
다음은 `KafkaProducer.send()`를 호출했을때 내부적으로 호출되는 doSend() 메서드다.

<img width="679" alt="스크린샷 2024-02-18 오후 5 36 12" src="https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/bfe724ba-a64f-4ac6-ab9b-77993c85e009">

코드 상에 빨간색으로 표시한 두 부분이 Key와 Value를 직렬화 하는 Serializer 로직이다. 이때 KeySerializer와 ValueSerializer는 처음 카프카 설정 초기화 시 등록한 Serializer들이 등록된다. 

<img width="665" alt="스크린샷 2024-02-18 오후 5 49 22" src="https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/65c93f36-d7e4-4784-a530-0f30a0137073">

만약 기본 타입 외에 직접 생성한 객체를 그대로 직렬화 하고 싶다면 직접 Serializer를 커스터마이징 해서 등록하면 된다.

## Partitioner

Serializer를 통해 메세지 직렬화가 끝났다면 이제 토픽의 어떤 파티션으로 메세지를 전송할지 결정해야 한다. 위에서 본 `doSend()` 에서 직렬화 코드 하단을 보면 `partition()`이라는 메서드가 있는걸 알 수 있다.  

![스크린샷 2024-02-18 오후 6 40 58](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/74808bf1-b487-4a0a-aa96-fa20c5899bc6)

외부에서는 어떤 기준에 의해 파티션이 결정되는지 알 수 없기 때문에 partition() 메서드 내부로 들어가보자.

![스크린샷 2024-02-18 오후 6 52 47](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/3fd0af63-e1c5-4d54-9442-539e6a6aead0)

3항 연산자를 통해 분기처리가 되어 있다. 코드 별 의미를 살펴보자.
- 1번 라인 : 만약 ProducerRecord를 생성할때 원하는 파티션을 직접 지정했다면 해당 파티션으로 전송한다.
- 2번 라인 : 파티션 정보를 입력하지 않았다면 KafkaProducer의 내부 파티션 결정 로직에 의해 결정된다.

카프카의 Partitioner는 디폴트로 등록되있는 걸 사용할 수도 있고, 사용자가 직접 커스텀 Partitioner를 등록해서 사용할 수도 있다. 먼저 카프카가 기본적으로 사용하는 Partitioner에 대해서 알아보자.

### 카프카 디폴트 Partitioner

카프카에 별도의 파티셔너를 등록하지 않으면 Default Partitioner가 사용된다. 내부의 `partition()` 메서드를 살펴보자.

![스크린샷 2024-02-18 오후 7 15 00](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/58d5da79-9ccb-430e-a1a7-4f3e4b723a2a)

Default Partitioner 내에서도 **Key의 유무**에 따라 파티션을 결정하는 방식이 달라진다. 

만약 Key가 없는 경우에는 `stickyPartitionCache`라는걸 사용해서 파티션을 결정한다. 이 부분에 사용되는 파티션 결정 방식은 카프카 버전에 따라 다른데, 카프카 2.4 버전 이전까지는 **라운드 로빈 방식**의 파티션 결정 방식이 사용됐고 2.4 버전 이후부터는 위의 코드에 나와있는 **stickyPartitionCache**가 사용된다.

### StickyPartition

라운드 로빈 방식은 직관적으로 이해가 된다. 그렇다면 StickyPartition 방식은 내부적으로 어떻게 동작하는걸까?

우선 라운드 로빈 방식으로 메세지를 각각의 파티션에 할당된 배치에 균등하게 분배하게 되면 배치를 충분히 채우지 못하고 브로커에 전송하게 될 가능성이 있다.  

이때의 비효율을 개선하기 위해 StickyPartition은 **특정 파티션으로 전송되는 하나의 배치에 메세지를 빠르게 먼저 채워서 보내는 방식**을 사용한다.

반면 Key가 존재하는 경우에는 해당 키를 해싱한 뒤, 토픽에 존재하는 전체 파티션의 개수로 모듈러 연산을 진행해서 파티션을 결정한다. 

## RecordBatch

지금까지 메세지를 직렬화 하고 파티션을 결정하는 작업을 수행했다. 그렇다면 이제 메세지를 바로 브로커로 보내게 되는 걸까? 만약 하나의 메세지를 처리할때마다 바로 네트워크를 통해 전송하게 된다면, 대규모 데이터를 처리하는 카프카의 속도가 매우 저하될 것이다. 
카프카는 메세지를 단건으로 전송하는 대신 **배치로 묶어서 한꺼번에 전송**한다. 위에서 계속 봐왔던 `doSend()` 메서드를 계속 보도록 하자. 

![스크린샷 2024-02-18 오후 10 04 42](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/6d95cb75-82bd-48eb-a970-42f1751da1f1)

해당 부분에서 유심히 봐야 할 부분은 **메세지 사이즈 검증**, **메세지 배치 추가** 크게 두 가지이다. 먼저 메세지 사이즈 검증 로직부터 살펴보자.

![스크린샷 2024-02-18 오후 10 03 31](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/ac86b83b-6a07-4742-8c11-62f890a8467e)

위의 코드를 보면 maxRequestSize와 totalMemorySize 두 값을 사용하여 메세지의 사이즈를 검증한다. 두 변수는 카프카 설정 상으로 각각 `max.request.size`와 `buffer.memory`이다.

- max.request.size : 한개의 메세지의 사이즈를 제한한다
- buffer.memory : RecordAccumulator의 전체 메모리 사이즈를 제한한다

만약 두가지 제한 중 한가지라도 걸린다면 **RecordTooLargeException**이 반환된다. 사이즈 제한을 통과했다면 이제 RecordAccumulator의 Batch에 메세지를 추가한다. `accumulator.append()` 메서드의 내부를 살펴보자.

![스크린샷 2024-02-18 오후 10 44 32](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/d8e498c4-2340-4708-9f74-ea9a2acf1cc1)

내부 로직을 보면 Deque<ProducerBatch>를 조회하는 부분이 나오는데, RecordAccumulator가 어떤 구조로 되어 있는지 글을 쓰며 참고만 naver D2 아티클의 이미지를 참고해서 살펴보자

<img width="507" alt="스크린샷 2024-02-18 오후 11 26 55" src="https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/87fd8e0f-ce75-47d8-988a-e3bfcccc8255">

RecordAccumulator는 내부적으로 batches 라는 Map을 가지고 있고, 이 Map의 Key는 파티션 정보, Value는 Batch들을 담는 Deque가 들어온다. 



`append()` 메서드는 먼저 batches 변수에서 파티션에 해당하는 Deque를 찾아온다. 이때 만약 Deque가 존재하지 않는다면 생성한 후 반환한다. 그 다음으로는 해당 Deque의 마지막 Batch에 메세지를 넣는 작업을 수행하는데, 이 부분은 tryAppend() 메서드를 조금 더 자세히 보자.

![스크린샷 2024-02-18 오후 10 58 52](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/c2f9f18c-d428-49ff-b949-a8cc0123360f)

위의 코드를 보면 deque.peekLast() 메서드를 통해 deque의 마지막 배치를 확인하는걸 알 수 있다. 이때 Deque 내에서 배치를 찾는다면 `last.tryAppend()` 단계로 넘어가고, 만약 배치가 하나도 없다면 null을 반환한다. 우선은 배치를 찾은 경우의 last.tryAppend()의 내부 동작을 살펴보자.

![스크린샷 2024-02-18 오후 11 04 37](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/41f3133f-92c7-48b4-a895-7d4cdef38438)

`recordsBuilder.hasRoomFor()` 메서드를 통해 **현재 Batch에 메세지를 넣을 공간이 있는지 확인**하는 분기문이 수행된다. 만약 현재 배치에 공간이 없다면 메세지를 저장하는 대신 null을 반환한다. 

위의 내용을 종합해봤을때, Deque에 Batch가 한개도 없거나, 조회한 Batch에 메세지를 넣을 공간이 더이상 없을때 null이 반환된다. null이 반환되는 경우에는 **새로운 배치를 생성**한 뒤, **Deque에 적재**하는 작업을 수행하는데 해당 로직을 살펴보자.

![스크린샷 2024-02-18 오후 10 58 13](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/8b17eda4-3e8b-4657-81ab-b8128f300d65)

위의 코드를 보면 ProducerBatch를 새로 생성한 후, 그 내부에 메세지를 tryAppend() 메서드를 사용해 적재한 뒤에 마지막으로 Deque에 적재하는걸 볼 수 있다.

## Sender

마지막으로 브로커로 메세지를 전송하는 Sender를 살펴보자. Sender는 이전의 과정과 다르게 Sender Thread라는 별도의 쓰레드로 동작한다. 

<img width="682" alt="스크린샷 2024-03-18 오전 1 12 03" src="https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/0dffff6e-e46b-4687-9166-bedd4737106a">

`runOnce()`라는 메세지를 반복한다. 

<img width="525" alt="스크린샷 2024-03-18 오전 4 01 03" src="https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/d72223ec-6d41-47b8-925e-99533e15304a">

`runOnce()` 내부에서는 `sendProducerData()` 메서드를 호출합니다.

<img width="772" alt="스크린샷 2024-03-18 오전 4 07 28" src="https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/01d46b6a-8e08-4f4d-a54d-44096cb4d988">

해당 메서드 내부적으로는 accumulator의 drain() 메서드를 호출한다.

![스크린샷 2024-03-27 오전 9 24 14](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/a3f3e602-d9f9-40bc-a287-4ea96925d53a)

drain() 메서드에서는 현재 브로커 노드들의 정보를 순회하며 각각의 노드로 보내야 하는 메세지를 수집하는 과정을 거친다. 이때 노드별로 할당된 파티션 정보를 얻어와서 TopicPartition을 생성한다.

![스크린샷 2024-03-27 오전 9 33 56](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/1c6a381f-f6b9-4886-9d39-ebf72d91c92b)

우리는 앞에서 RecordAccumulator가 내부적으로 batches라는 Map을 가지고 있고 Key가 TopicPartition, Value가 batch들을 모아둔 Deque인걸 확인했다. 이 Deque의 데이터를 `getDeque(topicPartition)` 메서드에서 이제 조회하게 된다.

![스크린샷 2024-03-27 오전 9 36 57](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/030c683b-cc86-4559-872a-70885627aa11)

Deque에서 가장 첫번째 Batch를 가져온 후 노드별 응답 리스트인 ready에 추가한다. 이 과정은 while문을 통해 계속 반복되는데, `max.request.size`가 가득 찰때까지 반복하게 된다.

![스크린샷 2024-03-27 오전 9 39 05](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/6f89cb9c-c905-43a6-a62e-860d1889d40d)

각 노드마다 위의 작업이 끝나면 모두 합쳐서 브로커로 전송하는 과정을 거치게 된다.

## Reference
- [https://d2.naver.com/helloworld/6560422](https://d2.naver.com/helloworld/6560422)


[nodejs]: https://nodejs.org/
[starter]: https://github.com/cotes2020/chirpy-starter
[pages-workflow-src]: https://docs.github.com/en/pages/getting-started-with-github-pages/configuring-a-publishing-source-for-your-github-pages-site#publishing-with-a-custom-github-actions-workflow
[latest-tag]: https://github.com/cotes2020/jekyll-theme-chirpy/tags
