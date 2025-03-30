---
title: 동시성 이슈를 해결하기 위한 JPA 낙관적 락 사용
date: 2023-12-05 00:20:00
categories: [Project]
tags: [spring]
pin: false
img_path: '/assets/img/'
---

서비스의 사용자가 증가하면 예상치 못한 여러 문제들이 발생할 수 있습니다. 오늘은 그중 분산 서버에서 특정 기능에 동시 접근했을때 발생할 수 있는 동시성 이슈를 알아보고 이를 해결하는 과정을 공유하고자 합니다.

## 문제 상황

<img width="234" alt="분산락" src="https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/92f45ed3-3dee-4c51-978f-dd53560ad876">

Melly 서비스는 소규모 그룹 간의 추억 공유를 중시하기 때문에 그룹의 최대 인원 수를 **10명**으로 제한해놨습니다. 만약 현재 그룹의 인원수가 10명일때 신규 참여 요청이 들어오면 아래의 예외를 반환합니다.

```json
{
    "status": 409,
    "code": "GROUP-005",
    "message": "그룹의 인원은 최대 10명 입니다.",
    "errors": [],
    "reason": null
}
```
일반적인 상황에서는 인원수를 검증하는 로직이 잘 동작할 것입니다. 하지만 분산 서버 환경에서 서로 다른 요청이 동시에 들어오면 어떻게 될까요? 

로컬 환경에서 서로 다른 포트로 두 어플리케이션을 실행시킨 뒤 동시에 그룹 참여 API를 호출한 결과, 제한 인원 10명을 무시한 채 **11명까지 인원이 증가**하는 현상을 발견했습니다. 왜 이런 현상이 발생했을까요?

```java
public void joinGroup(final Long userId, final Long groupId){
    User user = userReader.findById(userId); // 유저 정보 조회
    UserGroup userGroup = groupReader.findById(groupId); // 그룹 정보 조회
    groupValidator.isMaximumGroupMember(groupId); // 현재 인원수가 10명인지 체크
    groupValidator.isDuplicatedJoin(user.getId(), userGroup.getId()); // 중복 참여 인원 여부 체크
    groupAndUserWriter.save(GroupAndUser.of(user, userGroup)); // 그룹에 속한 유저 추가
}
```
위의 코드는 Melly 프로젝트에서 그룹에 참여하는 로직입니다. 4번 라인의 `isMaximumGroupMember()` 메서드 내부를 자세히 살펴보겠습니다.

```java
public class GroupValidator {

    private static final int GROUP_MEMBER_MAX_COUNT = 9;

    private final GroupAndUserReader groupAndUserReader;

    public void isMaximumGroupMember(Long groupId) {
        if (groupAndUserReader.countGroupMembers(groupId) > GROUP_MEMBER_MAX_COUNT) {
            throw new BusinessException(ErrorCode.PARTICIPATE_GROUP_NOT_POSSIBLE);
        }
    }
}    
```
`isMaximumGroupMember()` 메서드 내부에는 그룹에 속한 인원수를 DB에서 조회한 뒤 최대 인원수와 비교하는 조건문이 포함되있습니다. 만약 현재 그룹 내 인원수가 9명인 상황에서 서로 다른 서버의 요청이
동시에 해당 조건문으로 들어온다면 둘 다 조건을 통과하게 됩니다. 물론 하나의 서버에서 두 쓰레드가 동시에 해당 조건문으로 들어오는 경우도 마찬가지로 조건을 통과하게 됩니다. 

보통 서비스를 운영한다면 로드밸런서를 사용하는 분산 서버 환경이고, 이로 인해 동시성 이슈가 많이 발생하기에 분산 환경을 예시로 계속 진행하겠습니다.

## 락 구현 방식 선택
락 구현 방식에는 여러가지가 있지만 현재 기능의 특성이 무엇인지 파악하는게 락 선택의 중요한 기준이라 생각합니다.

그룹 참여 기능의 경우 인원 제한을 10명으로 걸어놨기에 트래픽이 많이 들어오지 않습니다. 또한 보통 그룹에 들어올때는 한번에 몰려 들어오는 것이 아닌 한명씩 시간차를 두고 들어오는 경우가 대부분이기 때문에 동시성 문제가 발생할 문제가 크지는 않습니다.

하지만 저희는 만약의 상황을 대비해야 하고, 이때 적합한 락 방식이 낙관적 락이라 생각했습니다.

낙관적 락은 동시성 문제가 발생 시, 재시도를 통해 갱신 손실은 방지해야 합니다. 하지만 재시도가 길어지는건 응답시간에 영향을 미칠 수 있기에 동시 요청으로 인해 재시도 횟수가 너무 많아질 것 같으면 차라리 락을 명시적으로 잡는게 더 성능상 유리하다고 생각합니다.

저는 낙관적락의 default 재시도 횟수와 대기 시간을 각각 5회와 500ms로 설정했습니다. 

그룹 참여 기능에서 저는 최대 5명 이상은 동시에 참여 요청이 올 일이 없을 것이라 판단했습니다. 따라서 저는 동시 요청자가 5명일때 낙관적 락의 default 설정값으로 처리가 가능한지를 테스트해봤습니다.

| 동시요청 | 낙관적락 적용시 성공 횟수 |
| -------- | ------------------------- |
| 1개      | 1개                       |
| 3개      | 3개                       |
| 5개      | 5개                       |

위의 결과를 보면 5개의 동시 요청에도 재시도 횟수 초과로 인한 갱신 손실 없이 요청이 모두 처리된 것을 확인했습니다. 따라서 그룹 참여 기능에는 낙관적 락을 적용하는 걸로 충분하다는 판단을 내렸습니다.

## AOP를 사용한 낙관적락과 재시도 로직 구현

지금부터는 코드 상에 낙관적락을 어떻게 구현했는지 살펴보겠습니다.

```java
@Slf4j
@Aspect
@Component
@Order(Ordered.LOWEST_PRECEDENCE - 1) // @Transaction보다 먼저 AOP가 호출
@RequiredArgsConstructor
public class OptimisticLockAspect {

    @Around("@annotation(cmc.mellyserver.common.aspect.lock.OptimisticLock)")
    public Object lock(final ProceedingJoinPoint joinPoint) throws Throwable {

        log.info(OPTIMISTIC_LOCK_AOP_ENTRY); // 낙관적 락 진입 확인

        // @OptimisticLock 어노테이션 값 획득
        MethodSignature signature = (MethodSignature)joinPoint.getSignature();
        Method method = signature.getMethod();
        OptimisticLock lock = method.getAnnotation(OptimisticLock.class);

        int retryCount = lock.retryCount();
        long waitTime = lock.waitTime();

        for (int i = 0; i < retryCount; i++) {

            try {
                return joinPoint.proceed();

            } catch (OptimisticLockingFailureException | CannotAcquireLockException ex) {
                log.info(OPTIMISTIC_LOCK_RETRY);
                Thread.sleep(waitTime);
            }
        }

        log.error(OPTIMISTIC_LOCK_ACQUIRE_FAIL);
        throw new BusinessException(ErrorCode.SERVER_ERROR);
    }
}
```

위의 코드가 낙관적 락을 적용하기 위한 어드바이저입니다. 하나씩 뜯어보겠습니다.

```java
@Order(Ordered.LOWEST_PRECEDENCE - 1) // @Transaction보다 먼저 AOP가 호출
```
AOP의 Order를 `Ordered.LOWEST_PRECEDENCE - 1`로 설정했는데요. 이게 어떤 의미인지 알아보겠습니다. Spring AOP는 여러 층으로 곂쳐서 사용할 수 있고, @Order 어노테이션을 통해 AOP 간 우선순위를 조정할 수 있습니다. 

우선순위는 숫자가 작을수록 높습니다. 즉, Order(2)보다 Order(1)이 우선순위가 높은 것이지요. Spring AOP의 default 우선순위는 `Ordered.LOWEST_PRECEDENCE`로써 **가장 낮은 우선순위**로 설정되있습니다. 정확한 값은 **Integer.MAX_VALUE**와 같은 값입니다.

제가 낙관적락 재시도 AOP의 우선순위를 조정한 것은 @Transactional 때문인데요. @Transactional도 default 우선순위가 적용되고 만약 AOP간 우선순위가 같은 경우 **@Transactional이 먼저 실행**되는걸 확인했습니다.

제가 낙관적락 재시도 AOP에 설정한 우선순위는 `Ordered.LOWEST_PRECEDENCE-1`로 @Transactional보다 먼저 실행됩니다. 왜 이런 구조가 되야하는걸까요?

<img width="507" alt="스크린샷 2024-01-20 오후 7 53 28" src="https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/fac19e4a-f103-4f45-b0f7-00ffdd627357">

낙관적락에서 버전 충돌을 확인하는 시점이 언제인지 생각해보겠습니다. 바로 **트랜잭션이 커밋**되는 순간입니다. 만약 커밋 시 버전 충돌이 발생한다면 OptimisticLockingFailureException이 외부로 반환될 것 입니다. 
저희가 커스터마이징한 낙관적락 재시도 AOP는 해당 예외를 외부에서 캐치한 후 트랜잭션을 다시 실행해야 하기 때문에 @Transactional보다 우선순위가 높아야 하는 것입니다.



```java
MethodSignature signature = (MethodSignature)joinPoint.getSignature();
Method method = signature.getMethod();
OptimisticLock lock = method.getAnnotation(OptimisticLock.class);

int retryCount = lock.retryCount();
long waitTime = lock.waitTime();
```

@OptimisticLock 어노테이션의 값으로 재시도 횟수와 대기 시간을 지정할 수 있습니다. 위의 코드를 통해 어노테이션에서 값을 가져올 수 있습니다.

```java
for (int i = 0; i < retryCount; i++) {

    try {
        return joinPoint.proceed();

    } catch (OptimisticLockingFailureException | CannotAcquireLockException ex) {
        log.info(OPTIMISTIC_LOCK_RETRY);
        Thread.sleep(waitTime);
    }
}
```

재시도 횟수 만큼 루프를 돌며 타겟 메서드를 호출합니다. 만약 낙관적락 예외가 발생하면 catch문에서 잡은 뒤 대기 시간 만큼 기다린후 재시도를 진행합니다.
이때 CannotAcquireLockException이 무엇인지 바로 이해되지 않을 수 있는데, 조금 있다가 설명하겠습니다.

```java
log.error(OPTIMISTIC_LOCK_ACQUIRE_FAIL);
throw new BusinessException(ErrorCode.SERVER_ERROR);
```
만약 전체 재시도 횟수를 다 실패하면 예외를 반환하고 종료합니다.

## 낙관적 락에서 발생 가능한 데드락 문제 해결
제목만 보면 조금 의아하다는 생각이 들 수 있습니다. 저희는 분산락이나 DB 쓰기락에서 발생할 수 있는 데드락을 회피하기 위해서 JPA의 낙관적 락을 사용합니다. 근데 여기서 어떤 데드락이 발생할 수 있을까요?

![스크린샷 2023-12-27 오후 10 55 44](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/720c8a26-1159-47f5-a60d-ca27dd09c9c3)

낙관적락은 버전 업데이트 시 update를 사용합니다. 이때 요청이 병렬로 들어온다면(10ms 정도의 오차까지 생각) 해당 레코드에 대한 쓰기락으로 인한 데드락이 발생합니다. 현재 MySQL8.0을 사용중이고 InnoDB 스토리지 엔진은 데드락 발견 시 즉시 데드락 트랜잭션을 풀어버리는 작업을 진행합니다.

이때 데드락 쓰레드에 의해 버려진 트랜잭션은 데드락 예외를 반환하고 그대로 종료가 되버립니다. 재시도가 되지 않고 바로 갱신 손실이 발생하는 것입니다. 이 문제를 어떻게 해결할까 고민하다가 데드락 예외도 재시도 예외에 포함시키는 걸로 결정했습니다.

정확하게 어떤 예외가 반환되는지 체크하기 위해 에러를 로깅했고, CannotAcquireLockException이라는 예외가 반환되는걸 확인했습니다.

```java
catch (OptimisticLockingFailureException | CannotAcquireLockException ex) {
    log.info(OPTIMISTIC_LOCK_RETRY);
    Thread.sleep(waitTime);
}
```
catch문에 이 예외를 추가했습니다. 그렇다면 이 방법이 잘 동작하는지 확인해볼까요?

![스크린샷 2023-12-27 오후 11 02 28](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/aa252710-1c61-4ab0-b6b9-5096fc55f482)

조치 후의 로그를 살펴보겠습니다. 데드락은 똑같이 발생하고 있는데요. 여기서 데드락으로 인한 예외를 catch문에서 잡아서 다시 재시도하는 로직을 수행합니다. 

해당 방법으로 개선 후 병렬로 요청을 하는 상황에서도 데이터가 유실되는 현상을 방지할 수 있었습니다. 물론 지금 다루고 있는 기능의 핵심 요소인 **그룹 인원 수 10명 제한**을 판별하는 부분도 낙관적 락으로 정확하게 보장할 수 있게 됐습니다.

## 재시도를 구현할 수 있는 다른 방법과 비교

지금까지 낙관적락에서 재시도를 구현하기 위해 스프링 AOP를 적용해봤습니다. 그렇다면 재시도 로직은 AOP를 사용해서만 구현할 수 있을까요?

다른 방법으로 **Spring Retry**를 통해서 재시도를 할 수 있습니다. 한번 Spring Retry를 적용했을때도 정상 동작하는지 살펴볼까요?

```java
    @Retryable(retryFor = {OptimisticEntityLockException.class,
                            CannotAcquireLockException.class},
        maxAttemptsExpression = "5", // 총 5번 재시도
        backoff = @Backoff(delay = 500L) // 재시도 간격, 500ms
    )
    public void joinGroup(final Long userId, final Long groupId) {
       // 그룹 참여 로직...
    }
    
    // 총 5회 반복이 실패했을때 실행되는 메서드
    // retry하는 메서드의 파라미터를 똑같이 넣어줘야합니다.
    @Recover
    void recover(OptimisticLockingFailureException e, Long userId, Long groupId) {
        log.info("optimistic lock eventually fail...");
        throw new BusinessException(ErrorCode.SERVER_ERROR);
    }
```
위의 코드는 AOP를 적용했을때와 똑같이 5회 반복 후 최종 실패하면 예외를 반환합니다. 똑같은 조건에서 테스트를 진행해본 결과, 낙관적락이 정상적으로 동작했고 10명 인원 제한도 잘 지켜졌습니다.

그렇다면 굳이 AOP 코드를 추가할 필요 없이 스프링이 제공해주는 기능을 사용하면 되는거 아닐까요? 저는 몇가지 이유를 고려해서 **AOP를 사용한 낙관적락 재시도 구현**을 결정했습니다.

### 비즈니스 로직에 부가기능을 수행하는 메서드 침범

Spring retry를 사용하면 recover를 위해서 별도의 메서드를 작성해야합니다. 이 메서드는 비즈니스 로직과는 연관이 없는, 낙관적락 재시도 실패 처리를 위한 **부가 기능**입니다. 저희가 Spring AOP를 통해 제거하고자 했던 **재시도 템플릿 코드의 일부**라 볼 수 있죠.

만약 현재 @Retryable을 적용한 코드가 통신 클라이언트를 사용해서 외부와 통신하는 부분에서 재시도가 필요한 부분이었다면 고민없이 적용했을 것입니다. 통신을 담당하는 부분은 비즈니스 로직이 아니기 때문이죠. 따라서 저는 어노테이션 하나만 붙여서 비즈니스 로직에 대한 침범을 최소화 할 수 있는 AOP가 더 적절한 방법이라 생각했습니다.

### @Retryable 메서드의 파라미터에 따라 recover 메서드 개수 증가

recover 메서드는 @Retryable을 적용한 메서드의 파라미터를 그대로 사용해야합니다. 그 말은 재시도하는 메서드들의 파라미터가 다르다면 각각을 위한 recover 메서드를 따로 만들어야 한다는 의미입니다. 안그래도 비즈니스 로직에 recover 메서드가 침범되는게 불만이었는데 개수까지 늘어난다면 안되겠죠? 
그래서 저는 최종적으로 AOP를 사용한 낙관적락 재시도를 선택했습니다.


## 더 나아가기
Melly 프로젝트에 낙관적락을 AOP 기반으로 구현해보면서 많은걸 배울 수 있었습니다. 무엇보다 상황에 맞는 기술을 고르는 시야를 넓힐 수 있었다고 생각합니다. 절대적으로 좋은 기술은 없습니다. 현재 만들고 있는 서비스에 가장 적합한 기술이 현재 상황에서는 가장 좋은 기술이라고 생각합니다.

그만큼 앞으로도 기술을 학습할때 장단점을 확실히 학습하고 현재 해결해야 하는 문제에 대입해 보는 자세를 꾸준히 길러야 겠다고 다짐했습니다.


[nodejs]: https://nodejs.org/
[starter]: https://github.com/cotes2020/chirpy-starter
[pages-workflow-src]: https://docs.github.com/en/pages/getting-started-with-github-pages/configuring-a-publishing-source-for-your-github-pages-site#publishing-with-a-custom-github-actions-workflow
[latest-tag]: https://github.com/cotes2020/jekyll-theme-chirpy/tags
