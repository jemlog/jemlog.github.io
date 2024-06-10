---
title: 분산 캐시 서버 Fault Tolerance 개선기
author: jemlog
date: 2023-12-05 00:20:00
categories: [Project]
tags: [spring]
pin: false
img_path: '/assets/img/'
---

Melly 서비스는 Redis 기반의 분산 캐시를 적용해 조회 성능을 최적화했습니다. 하지만 기존의 캐시 로직은 Standalone의 캐시 서버가 죽지 않는다는 이상적인 가정 하에서 잘 동작하는 시스템이었습니다.
이번 글에서는 `Circuit Breaker`와 `Spring Actuator health check` 설정을 통해 분산 서버 환경에서 가용성 있는 캐시 서버를 나름대로 구축해본 기록을 공유하고자 합니다.

## 초기 문제 상황
스프링은 `@Cacheable` 어노테이션을 사용해서 간편하게 캐시 기능을 사용할 수 있습니다. @Cacheable는 먼저 캐시를 조회한 뒤 데이터가 없다면 DB를 조회하는 `Cache Aside` 전략을 사용합니다. 
하지만 만약 분산 캐시 서버가 장애로 인해 죽는다면 어떻게 동작할까요?

<img width="862" alt="스크린샷 2023-12-06 오후 5 16 46" src="https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/9302e40c-d5ec-4a15-b01d-5e8084aeae7b">

간단히 테스트 해보기 위해 캐싱 기능을 사용하는 API를 POSTMAN을 통해 호출해봤습니다. 그 결과 요청이 Pending 되는 현상이 발생했고, 몇초 동안 Pending되나 체크한 결과 **60초 이후**에 `QueryTimeoutException`을 반환했습니다. 왜 60초 이후에 예외가 반환되는지
체크하기 위해 레디스 클라이언트로 사용중인 Lettuce의 Configuration을 분석해봤습니다.

<img width="730" alt="스크린샷 2023-12-06 오후 5 23 55" src="https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/be90f5fc-aa6b-4376-b9ae-077239bedb88">

timeout 필드의 값으로 `RedisURI.DEFAULT_TIMEOUT`가 들어오는걸 알 수 있습니다. RedisURI로 들어가보겠습니다.

<img width="730" alt="스크린샷 2023-12-06 오후 5 24 18" src="https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/3b9e3d44-4412-4ca2-ac64-778bc358ecee">

DEFAULT_TIMEOUT으로 60초가 설정되어 있는걸 알 수 있습니다. 만약 타임아웃을 재정의하지 않으면 분산 캐시 서버가 죽은 상황에서 모든 요청은 Lettuce Client가 Redis 서버에 재연결할때까지 최대 60초를 대기해야 합니다. 해당 상황이 지속되면 어떤 일이 발생할까요?

사용자 요청을 처리하는 쓰레드가 톰캣 쓰레드풀로 재시간에 반환되지 못하고, **쓰레드 풀 고갈**로 인해 다른 기능들에도 장애가 전파되다가 곧 **전체 시스템 장애가 발생**할 것입니다. `Bulkhead` 기능을 사용해 캐시 서버 장애가 전체 쓰레드 고갈로 이어지는걸 막을 수 있다 생각하지만,
현재 프로젝트에서는 구현하지 못했고 아직 추가 학습이 필요하다 생각하기에 우선은 넘어가겠습니다.

## Command Timeout 설정을 통한 무한 로딩 방지

그렇다면 지나치게 긴 Timeout을 설정을 통해 단축시켜 보겠습니다.

```java
@Bean(name = "redisCacheConnectionFactory")
RedisConnectionFactory redisCacheConnectionFactory() {
    RedisStandaloneConfiguration redisStandaloneConfiguration = new RedisStandaloneConfiguration();
    redisStandaloneConfiguration.setHostName(host);
    redisStandaloneConfiguration.setPort(port);
        
    LettuceClientConfiguration lettuceClientConfiguration = LettuceClientConfiguration.builder()
        .commandTimeout(Duration.ofSeconds(1)) // 해당 부분에서 타임아웃을 설정
        .build();

    return new LettuceConnectionFactory(redisStandaloneConfiguration, lettuceClientConfiguration);
    }
```
`LettuceClientConfiguration`의 `commandTimeout` 값을 변경하면 타임아웃을 조절할 수 있습니다.


## Circuit Breaker를 통한 Fallback 구현
타임아웃을 짧게 설정했으니 이제 다시 캐시 서버를 죽이고 API를 호출해보겠습니다.
```json
{
    "status": 500,
    "code": "COMMON-001",
    "message": "서버에서 처리할 수 없습니다.",
    "errors": null,
    "reason": "Redis command timed out"
}
```
이번에는 저희가 설정한 command timeout만큼 짧게 pending된 후, 위의 예외를 반환합니다. 이제 요청을 처리하는 쓰레드가 긴 시간 대기하지 않기 때문에, 캐시를 사용하지 않는 기능들로 예외가 전파되지 않을 것입니다.

하지만 캐시 서버에 장애가 발생했다고 사용자에게 예외를 반환하는게 가용성 측면에서 옳은 조치일까요? 사용자의 불편함을 최소화하기 위해서는 캐시 서버가 죽었을때 DB에서라도 데이터를 조회 후 유저에게 제공해주는게 맞는 방법이라 생각합니다.
또한 Command Timeout이 발생할때까지 대기한다는건, 장애가 발생한 캐시 서버로 요청을 보냈다는걸 의미합니다. 이는 캐시 서버의 회복을 더디게 만들기 때문에, 캐시 서버에 장애가 발생했다는걸 인지한다면 `Fail Fast`, 즉 빠르게 실패라 판단하고 Fallback 로직을 실행하는게 좋습니다.

위의 요구사항을 해결할 수 있는 방법이 무엇인지 고민했고, `Resilence4j` 기반의 `Circuit Breaker`를 도입하기로 결정했습니다.

<img width="464" alt="스크린샷 2023-12-06 오후 8 23 56" src="https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/f1495382-785e-45a2-95bb-9d8353d272f9">

### Circuit breaker
서킷 브레이커는 전기 회로 차단기의 개념을 가져온 기술로 3가지의 상태로 이루어져있습니다.
- **CLOSED** : 로직이 정상 동작하고 있는 상태입니다.
- **OPEN** : 일정 횟수 이상 예외나 Slow Call이 발생해서 장애가 발생한 리소스로의 접근이 차단된 상태입니다.
- **HALF_OPEN** : 상태를 CLOSED로 바꿀지, OPEN 상태를 유지시킬지 트래픽을 조금 흘려서 테스트 해보는 상태입니다.

Spring의 `@Cacheable`과 서킷 브레이커를 조합해서 사용하는 방법을 고민해봤고, 아래의 코드처럼 구현했습니다. 

```java
@Bean
public CacheManager customCacheManager(
    @Qualifier("redisCacheConnectionFactory") RedisConnectionFactory connectionFactory,
    CircuitBreakerFactory circuitBreakerFactory) {
    
    // ... RedisCachemanager 설정
    
    CircuitBreaker circuitBreaker = circuitBreakerFactory.create(CACHE_CIRCUIT);

    /* Circuit Breaker 설정 */
    return new CustomCacheManager(redisCacheManager, circuitBreaker);
}
```
첫번째로 CacheManager를 커스터마이징 해서 내부에 서킷 브레이커 인스턴스를 넘겨줬습니다.
```java
public class CustomCacheManager implements CacheManager {

    private final RedisCacheManager globalCacheManager;
    private final CircuitBreaker circuitBreaker;

    public CustomCacheManager(RedisCacheManager globalCacheManager,
        CircuitBreaker circuitBreaker) {
        this.globalCacheManager = globalCacheManager;
        this.circuitBreaker = circuitBreaker;
    }

    @Override
    public Cache getCache(String name) {
        return new CustomCache(globalCacheManager.getCache(name), circuitBreaker);
    }
}
```
두번째로는 Cache를 재정의해서 내부에 서킷 브레이커 인스턴스를 넘겼습니다.
```java
@Slf4j
public class CustomCache implements Cache {

    private final Cache globalCache;

    private final CircuitBreaker circuitBreaker;

    public CustomCache(Cache globalCache, CircuitBreaker circuitBreaker) {
        this.globalCache = globalCache;
        this.circuitBreaker = circuitBreaker;
    }
    
    @Override
    public ValueWrapper get(Object key) {
        return circuitBreaker.run(() -> (globalCache.get(key)), (throwable -> fallback()));
    }
    
    private ValueWrapper fallback() {
        log.error("글로벌 캐시 다운, Fallback 메서드 실행");
        return null;
    }
}
```
이 부분이 핵심 로직입니다. `get()` 메서드를 호출할때 예외가 발생하면 `fallback` 메서드를 호출하고 null을 반환합니다.
만약 ValueWrapper로 null이 반환되면 스프링은 캐시에 데이터가 없다고 판단하고 AOP의 타겟 메서드를 호출해 DB에서 데이터를 직접 조회합니다.

```yaml
resilience4j:
  circuitbreaker:
    configs:
      cacheCircuit:
        slidingWindowType: COUNT_BASED
        minimumNumberOfCalls: 80 # 판단을 위한 최소 요청 횟수
        slidingWindowSize: 100 # 슬라이딩 윈도우 사이즈
        failureRateThreshold: 80 # OPEN이라고 판단하는 실패율
        waitDurationInOpenState: 30s # OPEN 상태에 머물러 있는 시간
        permittedNumberOfCallsInHalfOpenState: 50 # HALF_OPEN 상태에서 상태 전이를 판단하는 요청 횟수
        automaticTransitionFromOpenToHalfOpenEnabled: true # OPEN에서 HALF_OPEN으로 자동으로 넘어가는지 여부
        registerHealthIndicator: true # Actuator의 Health Indicator를 등록
    instances:
      cacheCircuit:
        base-config: cacheCircuit
```
서킷 브레이커 인스턴스에 대한 설정은 yml 파일에서 관리합니다.

### Circuit Breaker 적용 후
<img width="730" alt="스크린샷 2023-12-06 오후 8 51 40" src="https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/ae450309-98f8-4f16-ae8e-4f266ba85d58">

서킷 브레이커를 적용한 후에는 캐시 서버 장애가 발생했을때, Fallback 메서드가 호출되고 실제로 DB에서 SELECT 쿼리가 발생합니다.

<img width="730" alt="스크린샷 2023-12-06 오후 8 54 35" src="https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/ffdc8659-42d9-4dbc-a64f-ed82b5ac8f5a">

또한 예외 발생 횟수가 임계치를 넘어서면 서킷의 상태가 `OPEN`으로 변경됩니다. 이때부터는 Redis 서버가 장애라 판단하고 Command Timeout을 기다리지 않은 채 DB 쿼리를 진행합니다.
캐시 서버가 정상적으로 복구되고 Lettuce Client가 커넥션을 다시 생성하면 `HALF_OPEN` 상태에서 정상 트래픽을 몇번 흘린 후 `CLOSED` 상태로 돌아갑니다.

## Redis pub/sub을 통한 분산 서버간 Circuit State 전파
지금까지 스프링 캐시에 서킷 브레이커를 도입하는 과정을 살펴봤는데요. 위의 개선사항은 현재 서버가 `단일 인스턴스`인 경우만 고려한 케이스입니다. 우리는 서비스가 성장함에 따라 많은 트래픽을 수용해야 하고 필수적으로 
서버 스케일 아웃을 진행합니다. 따라서 지금부터는 `분산 환경`에서 서킷 브레이커를 사용할때의 문제점과 개선방안을 분석해보겠습니다.

위의 그림처럼 로드밸런싱되는 분산 서버 환경에서는 모든 서버 인스턴스가 한번에 서킷 브레이커의 상태를 OPEN으로 변경하지 않습니다. 애초에 서킷 브레이커 인스턴스는 공유 자원이 아닌 각각의 서버 인스턴스에 종속된 요소이기 때문입니다.
만약 Redis 서버의 장애가 확실시 되서 더이상 트래픽을 보내지 말아야 하는 상황에서도 로드밸런서는 부하 분산을 하며 계속 Redis 서버로 트래픽을 흘려보내게 됩니다.

따라서 분산 환경에서는 모든 서버의 서킷 브레이커 인스턴스의 상태를 `일괄적으로 동기화 시킬 수 있는` 수단이 필요합니다. 어떤 방법을 사용할 수 있을까 고민하던 중, 분산 환경에서의 로컬 캐시를 동기화 하는 방법으로 `Redis Pub/Sub`을 사용하는 아티클이 떠올랐습니다.
토픽을 구독하고 있는 서버 모두에게 빠르게 서킷 OPEN 메세지를 발행할 수 있겠다 생각하여 바로 실행에 옮겼습니다.

```java
@Slf4j
@Configuration
public class CircuitBreakerConfig {

    @Bean
    public RegistryEventConsumer<CircuitBreaker> myRegistryEventConsumer(CircuitBreakerEventPublisher redisPublisher) {

        return new RegistryEventConsumer<CircuitBreaker>() {
            @Override
            public void onEntryAddedEvent(EntryAddedEvent<CircuitBreaker> entryAddedEvent) {

                CircuitBreaker.EventPublisher eventPublisher = entryAddedEvent.getAddedEntry().getEventPublisher();
                
                // 서킷의 상태가 변경되는 이벤트를 감지합니다.
                eventPublisher.onStateTransition(event -> {
                    log.info("onStateTransition {}", event.getStateTransition());
                    publishCircuitOpenTopic(event, redisPublisher);
                });
            }
        };
    }
}    
```
그렇다면 어떻게 서킷이 OPEN 상태로 바뀌었다는걸 인지할 수 있을까요? Resilence4j에는 위의 코드처럼 특정 이벤트를 캐치할 수 있는 `EventConsumer`가 있습니다.
저는 이중에서 `onStateTransition` 이벤트를 잡았습니다.

근데 상태가 변경되는 이벤트가 OPEN만 존재하는게 아니라 `HALF_OPEN`으로 바뀌는 것도 있고 `CLOSED`로 바뀌는 것도 있습니다. 이중 어떻게 OPEN으로 바뀌는것만 잡아낼까요? 
`publishCircuitOpenTopic()` 메서드를 살펴보겠습니다.

```java
    private void publishCircuitOpenTopic(CircuitBreakerOnStateTransitionEvent event,
        CircuitBreakerEventPublisher redisPublisher) {
        if (openStateSpreadEnabled(event)) {
            redisPublisher.publish(new ChannelTopic(CIRCUIT_OPEN), event.getCircuitBreakerName());
        }
    }
    
    private boolean openStateSpreadEnabled(CircuitBreakerOnStateTransitionEvent event) {
        return !event.getStateTransition().getFromState().name().equals(OPEN_STATE) 
              && event.getStateTransition().getToState().name().equals(OPEN_STATE);
    }
```

`openStateSpreadEnabled()`라는 메서드를 통해 조건을 판별 후 토픽을 발행할지 결정하는데요. 저는 `1.현재 서킷의 상태가 OPEN이 아니면서`, `2.OPEN으로 상태가 변하는 경우만` 토픽을 발행하도록 조건문을 설계했습니다. 여기서 1번째 조건을 만든 이유는
서킷 브레이커 인스턴스의 상태를 OPEN으로 변경시키는 과정에서 `OPEN -> OPEN`인 경우도 이벤트로 감지되기 때문입니다. 이렇게 되면 **모든 서버 인스턴스가 끝도 없이 토픽을 발행, 구독하는 현상이 발생**해서 첫번째 조건을 추가했습니다.

자, 이제 Redis pub/sub을 실행하는 `publisher`와 `subscriber` 코드를 살펴보겠습니다.
```java
@Service
@RequiredArgsConstructor
public class CircuitBreakerEventPublisher {

    private final RedisTemplate redisTemplate;

    public void publish(ChannelTopic topic, String message) {
        redisTemplate.convertAndSend(topic.getTopic(), message);
    }
}
```
publisher 코드는 단순합니다. RedisTemplate을 사용해서 토픽을 발행합니다.
```java
@Service
public class CircuitBreakerEventSubscriber implements MessageListener {

    private final Resilience4JCircuitBreakerFactory circuitBreakerFactory;

    private final RedisTemplate<String, Object> redisTemplate;

    public CircuitBreakerEventSubscriber(Resilience4JCircuitBreakerFactory circuitBreakerFactory,
        RedisMessageListenerContainer redisMessageListenerContainer, RedisTemplate redisTemplate) {
        this.circuitBreakerFactory = circuitBreakerFactory;
        this.redisTemplate = redisTemplate;

        // 토픽 등록
        ChannelTopic channelTopic = new ChannelTopic(CIRCUIT_OPEN);
        redisMessageListenerContainer.addMessageListener(this, channelTopic);
    }
    
    @Override
    public void onMessage(Message message, byte[] pattern) {

        // 1. 서킷 인스턴스를 가져오기 위한 registry를 불러온다.
        CircuitBreakerRegistry circuitBreakerRegistry = circuitBreakerFactory.getCircuitBreakerRegistry();
        // 2. 구독한 메세지를 역직럴화해서 서킷 브레이커 인스턴스 명을 얻는다.
        String circuitBreakerName = (String)redisTemplate.getValueSerializer().deserialize(message.getBody());
        // 3. 해당 이름으로 서킷 브레이커 인스턴스를 찾는다.
        CircuitBreaker circuitBreaker = circuitBreakerRegistry.circuitBreaker(circuitBreakerName);
        // 4. 상태를 OPEN으로 변경한다.
        circuitBreaker.transitionToOpenState();
    }
}
```
subscriber는 구독한 메세지로부터 서킷 브레이커 인스턴스 이름을 파싱한 후, 해당 이름을 가진 인스턴스의 상태를 `OPEN`으로 변경합니다.

기능을 만들었으니 정상적으로 동작하는지 실험을 해봐야겠죠? 클라우드에 4~5개의 서버 인스턴스를 생성해서 테스트 환경을 구축하는건 현재로써는 비용 부담이 있습니다. 따라서 로컬의 8080~8084 포트에 서버 인스턴스를 개별적으로 띄우는 방식으로 테스트를 진행했습니다.

![스크린샷 2023-12-07 오전 12 48 47](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/bf4296c6-6d0b-4465-9ced-4c45e20a3d25)

다음으로 캐시 서버를 강제로 종료시킨 뒤, 서킷이 OPEN될때까지 요청을 보냈습니다.

![스크린샷 2023-12-07 오전 12 56 36](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/d3de1e05-e70f-4d3d-9a50-d62a6ee9580b)

5개의 서버 중 하나의 서버에서 서킷 브레이커가 OPEN된 후, 토픽을 발행했습니다.

![스크린샷 2023-12-07 오전 12 58 56](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/4faf7e91-ae99-4da5-870c-c3065e2b252b)

나머지 4개의 서버는 구독한 메세지를 받은 후, 자신의 서킷 브레이커 인스턴스 상태를 OPEN으로 변경했습니다.

<img width="1511" alt="스크린샷 2023-12-07 오전 12 58 19" src="https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/caba67b3-587b-4963-b8ce-b18f46a739b3">

각각의 서버에서 Actuator metric을 확인한 결과 5개 서버의 서킷 브레이커 상태가 모두 OPEN으로 변경된걸 알 수 있습니다.

> CLOSED 상태는 전파를 안해도 될까?

지금까지 서킷 브레이커의 OPEN 상태를 전파하는 방법을 살펴봤습니다. 근데 CLOSED 상태는 따로 전파를 할 필요가 없을까요? 저는 CLOSED 상태는 각각의 서버 인스턴스가 자연스럽게 회복해야 한다고 생각합니다.

만약 모든 서버의 서킷 브레이커가 CLOSED 상태로 한번에 바뀌면 대량의 트래픽이 Redis 서버로 유입될 수 있습니다. 따라서 각각의 서버가 조심스럽게 서킷을 CLOSE하는게 시스템에 더 안전한 방법이라 봅니다. 


## 커스텀 헬스 체크 엔드포인트 설정을 통한 로드밸런싱 장애 방지
드디어 마지막 개선사항이네요. 하나의 기능을 제대로 많드려면 정말 많은 고민이 필요한 것 같습니다! 이번에는 Spring Actuator 부분을 살펴보고자 합니다.

보통 클라우드 환경에서는 AWS의 로드밸런서를 사용해서 분산 서버를 구축할껀데요. 이때 로드밸런서는 타겟 그룹의 헬스체크를 위해 PING을 보내게 되고 그 경로는 보통 Actuator의 헬스체크 API가 됩니다. 이때 Actuator는 단순히 API가 잘 보내지는지만 체크할까요? 
Actuator의 헬스체크는 내부적으로 꽤 복잡한 과정을 거칩니다. 

```json
    "db": {
      "status": "UP",
      "components": {
        "dataSource": {
          "status": "UP",
          "details": {
            "database": "MySQL",
            "validationQuery": "isValid()"
          }
        }
      }
    }
    "mail": {
      "status": "UP",
      "details": {
        "location": "smtp.gmail.com:587"
      }
    }
    "ping": {
      "status": "UP"
    }
    "redis": {
      "status": "UP"
    }
```
위의 JSON 파일은 `/actuator/health`를 호출했을때 나오는 결과값입니다. 실제로는 더 많은 값들이 나오지만 중요한 것들 몇 가지만 추려봤습니다.

Spring Acuator는 health check 과정에서 연결된 `외부 리소스들과의 커넥션`을 모두 체크합니다. 
커넥션이 잘 맺어져 있다면 UP을 반환하고, 커넥션에 문제가 있다면 DOWN을 반환합니다. 여기서 중요한건 **단 한개의 외부 리소스라도 커넥션에 실패하면 전체 health check 결과가 DOWN으로 결정**됩니다.

<img width="500" alt="스크린샷 2023-12-06 오후 8 54 35" src="https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/578db2bf-a0a0-4022-ae41-5342aa9dc396">

로드밸런서는 health check를 진행하고 DOWN이 반환되면 해당 인스턴스가 장애 상황이라 판단하고 트래픽을 보내지 않습니다. 만약 이 조건 하에서 캐시 서버에 장애가 발생하고 커넥션을 맺지 못한다면 어떻게 될까요? 

분명 저희는 가용성을 보장하기 위해 캐시 서버 장애에 대한 Fallback을 모두 구현했지만, 로드밸런서는 서버 장애라 판단하고 트래픽을 전송하지 않습니다. 타겟 그룹 내의 모든 서버는 Health Check의 결과로 DOWN을 반환할 것이고 결국
어떤 인스턴스로도 트래픽을 보내지 못하는 장애 상황이 발생할 것입니다.

여기서 우리는 어플리케이션에 대한 헬스 체크의 의미를 생각해봐야 합니다. 특히 최근 쿠버네티스 환경에서는 파드의 시작 시점에 startup probe를 수행하고 이후 지속적으로 readiness probe와 liveness probe 헬스체크를 수행하는 만큼 헬스체크는 
중요하다고 할 수 있습니다. 

저희가 헬스체크를 통해 알고 싶은건 외부 컴포넌트와 연결이 잘 되있는지 여부가 아닌 어플리케이션 자체가 살아있는지입니다. 외부 컴포넌트의 장애는 그 컴포넌트에 대한 모니터링을 통해 감지하면 되는 것입니다.

따라서 저는 기존의 Spring Actuator의 health check를 사용하지 않고, 커스텀한 health check API를 만들었습니다. 

```java
@GetMapping("/health")
public String health(){
   return "UP";    
}
```

위와 같은 간단한 API만 생성하면 어플리케이션이 살아있는지 판단 가능합니다.


## 결론
지금까지 분산 환경에서 어떻게 가용성을 보장할 수 있을지에 대한 저의 고민을 풀어봤습니다. 아마 현업에서 가용성을 보장하는 방식은 제 생각과 다를수도 있고 훨씬 더 복잡할 것입니다.
그치만 평소 생각해보지 못했던 이슈에 대해 고민해보고 그 과정에서 서킷 브레이커 같은 새로운 기술을 도입해볼 수 있었다는 점에서 성장의 기회였다고 생각합니다.


[nodejs]: https://nodejs.org/
[starter]: https://github.com/cotes2020/chirpy-starter
[pages-workflow-src]: https://docs.github.com/en/pages/getting-started-with-github-pages/configuring-a-publishing-source-for-your-github-pages-site#publishing-with-a-custom-github-actions-workflow
[latest-tag]: https://github.com/cotes2020/jekyll-theme-chirpy/tags
