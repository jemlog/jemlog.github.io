---
title: DB 쓰기락 적용을 통한 동시성 문제 해결 
date: 2023-12-14 20:55:00 +0800
categories: [Project]
tags: [spring]
pin: false
img_path: '/assets/img'
---

오늘은 프로젝트에 DB 쓰기락을 적용해서 동시성 문제를 해결한 경험을 공유하겠습니다. 

## 문제 상황

현재 서비스에서는 댓글에 좋아요를 누를 수 있는 기능을 제공합니다. 이때 여러명의 사용자가 동시에 좋아요 버튼을 누르면 동시성 문제로 인해 **갱신 손실**이 발생할 수 있습니다.

<img width="336" alt="스크린샷 2023-12-19 오후 1 22 05" src="https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/ff2498b8-1236-4b80-ae97-64e86360a094">


## 문제 해결 방법 선정

현재 서비스가 단일 서버로 운영된다고 생각해보겠습니다. 그렇다면 가장 간단한 해결 방법은 Java 모니터락 기반 **Synchronized** 키워드나, **Java Lock API**를 사용하는 것입니다.
하지만 현재 멜리 서비스는 인스턴스 2개를 로드밸런서에 연결해서 부하를 분산하고 있기 때문에 두 서버간의 동기화가 필요합니다.

그렇다면 다음으로 생각해볼 수 있는 해결 방법에는 뭐가 있을까요? 저는 JPA 기반의 **비관적락&낙관적락** 그리고 **분산락**이 떠올랐습니다.

각각의 락 매커니즘에 대해 간단히 알아보고 넘어가겠습니다.
- **낙관적 락** : 모든 요청은 동시성 문제가 발생하지 않는다고 가정합니다. 엔티티를 수정 시도할때 처음 조회한 엔티티와 현재 시점의 엔티티의 버전이 다르면 트랜잭션을 롤백합니다.
- **비관적 락** : 모든 요청은 동시성 문제가 발생한다고 가정합니다. 수정하고자 하는 엔티티를 조회할때 DB의 `for update` 쓰기락을 획득합니다.
- **분산 락** : 원하는 데이터 자체에 락을 거는 것이 아니라 외부 공통 저장소에 락을 건 후, 락을 획득한 서버만 로직을 실행하도록 만듭니다.

**비관적락**은 직접 DB에 쓰기락을 거는 작업이기 때문에 락을 제대로 반환하지 않으면 데드락이 발생할 수 있습니다. 또한 MySQL의 InnoDB 스토리지 엔진은 인덱스 기반의 레코드락을 제공하기 때문에 인덱스 설계를 제대로 하지 못하면 오히려 많은 레코드를 잠궈버린다는 단점이 있습니다.

**분산락**은 하나의 쓰레드가 락을 획득한 뒤에 그 다음 요청들은 모두 락을 획득할때까지 블로킹이 됩니다. 따라서 락을 획득한 서버가 제대로 락을 반환하지 않으면 데드락이 발생할 수 있습니다. 하지만 DB 레코드 자체에는 락을 걸지 않기에 DB 데드락으로부터 자유롭다는 장점이 있습니다.

**낙관적락**은 CAS(Compare And Set) 연산을 구현한 것으로, 실제 락 대신 version을 통해 동시성을 관리하지만 동시 요청이 많아서 버전 충돌이 많이 발생하면 재시도 횟수가 많아져서 부하와 응답시간이 증가하게 됩니다.

저는 **비관적락**과 **낙관적락** 중에서 비교를 통해 적절한 락 방식을 선택해보겠습니다. 분산락을 사용하지 않은 이유에 대해서는 마지막 문단에서 다뤄보겠습니다.

## 낙관적 락과 비관적락 성능 비교

좋아요 추가/삭제 기능은 다수의 동시 요청이 발생 가능하기 때문에 비관적락이 적합할 것이라고 직관적으로 생각했습니다. 하지만 현재 서비스에서 낙관적 락이 어느정도 동시성 문제를 해결해줄 수 있다면 직접 락을 획득하지 않는 방법을 선택하는 것도 괜찮아보였습니다.

이론만으로는 합리적인 결정을 할 수 없기에 직접 테스트를 진행했습니다.

테스트는 로컬에서 진행하며 배포 환경과 동일하게 두 개의 인스턴스를 8080,8081 포트에서 실행했습니다. 이번 테스트의 핵심은 **요청을 두개의 인스턴스에 병렬로 동시에 보내는 것**입니다. 그래야 락 획득을 위한 대기 상태나 락 획득 실패로 인한 재시도를 제대로 관찰할 수 있기 때문입니다.
```shell
#!/bin/bash

# 각 요청에 사용할 다른 Authorization 토큰들
tokens=(
"Bearer eysfes.dfdd ..."
"Bearer eysfas.dfsh ..."
"Bearer eysfds.d13a ..."
"Bearer eysfgs.dfss ..."
"Bearer eysfhs.grgr ..."
...
)

# 병렬로 요청을 보낼 URL 목록
urls=(
"http://localhost:8080/api/comments/1/like"
"http://localhost:8081/api/comments/1/like"
"http://localhost:8080/api/comments/1/like"
"http://localhost:8081/api/comments/1/like"
"http://localhost:8080/api/comments/1/like"
...
)

# 각 URL에 대한 요청을 병렬로 보내기
for i in "${!urls[@]}"; do
  url="${urls[$i]}"
  token="${tokens[$i]}"
  # Authorization 헤더를 포함하여 curl 요청 보내기
  curl -s -X POST -H "Authorization: $token" -o /dev/null "$url" &
done
# 모든 백그라운드 작업이 완료될 때까지 대기
wait
```
위와 같이 병렬 요청 스크립트를 작성해서 테스트를 진행했습니다. 현재 프로젝트는 한명의 사용자가 좋아요를 중복으로 할 수 없기 때문에 서로 다른 유저를 사용하기 위해서 인증 토큰 여러개를 준비했습니다. 
테스트는 **동시 요청 5회**부터 순차적으로 증가하며 진행했습니다. 이때 낙관적 락의 **재시도 횟수는 5회**, **interval은 500ms**로 설정했습니다.

테스트를 진행한 결과는 아래와 같습니다.

| 동시요청 | 낙관락 반영                        | 비관적락 반영 |
| -------- | ---------------------------------- | ------------- |
| 3회      | 3개                                | 3개           |
| 5회      | 5개                                | 5개           |
| 7회      | 7개                                | 7개           |
| 10회     | <span style="color:red">7개</span> | 10개          |

동시 요청이 10개 이상 들어오는 구간부터 낙관적 락의 재시도가 5회 실패해서 요청이 버려지는 현상이 발생했습니다. 

재시도 횟수를 5회보다 더 늘리는 방법도 있겠지만 이 이상으로 재시도가 늘어나면 응답 시간이 길어지므로 오히려 낙관적 락이 적합하지 않은 상황이라 생각했습니다.

결론적으로 인기가 많은 메모리의 댓글에는 동시 요청자가 집중될 수 있기에 낙관적 락보다는 **DB 락 기반의 비관적락**을 사용하는게 적절하다고 판단했습니다.

## 비관적락 코드 적용

비관적락을 도입하기로 결정했다면 코드를 작성해보겠습니다. 비관적락은 JPA를 사용해서 간단하게 사용할 수 있습니다.
```java
public interface CommentRepository extends JpaRepository<Comment, Long> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @QueryHints({@QueryHint(name = "javax.persistence.lock.timeout", value = "3000")})
    @Query("select c from Comment c where c.id = :commentId")
    Comment findByPessimisticLock(@Param("commentId") Long commentId);
    
}
```

`LockModeType.PESSIMISTIC_WRITE`를 사용하면 **SELECT FOR UPDATE** 문을 통해 쓰기락을 획득합니다. 락 대기 타임아웃은 @QueryHint로 작성 가능합니다. 
아래의 코드의 `commentReader.findByIdWithLock(commentId)` 내부에서 `findByPessimisticLock()`을 호출하면 끝입니다.

```java
    @Transactional
    public void saveCommentLike(final Long userId, final Long commentId) {

        Comment comment = commentReader.findByIdWithLock(commentId); // 비관적락을 사용해서 Comment 가져오기
        commentLikeValidator.validateDuplicatedLike(commentId, userId);
        comment.addLike();
        commentLikeWriter.save(userId, comment);
    }
```

## 분산락으로 바라본 오버엔지니어링에 대한 생각

비관적락을 사용한 좋아요 기능은 분산락으로도 처리가 가능합니다. 실제 분산락을 적용해서 비관적락과 똑같이 성능 비교를 해봤을때 정상적으로 동작하는걸 확인했습니다. 분산락을 적용하는 코드를 다 작성하고 사용 가능하도록 만들었지만 결과적으로는 DB 쓰기락을 적용했습니다.

우선 분산락이 가장 이상적으로 사용되는 경우는 **하나의 트랜잭션 내에서 여러 테이블에 대한 락을 획득**해서 데드락의 위험이 커지는 경우라 생각합니다. 여러 테이블에 직접 락을 획득하는 대신 하나의 분산락만 획득 하는 것이지요. 하지만 현재 댓글 좋아요 기능은 오직 댓글 테이블 하나에만 락을 획득합니다.
따라서 분산락을 적용했을때의 메리트가 전혀 없습니다. 

또한 분산락에 Redis 같은 외부 데이터베이스를 사용하는 경우 장애 상황에 대한 대비를 따로 해줘야 합니다. 레디스 장애로 인한 서비스 중단을 막기 위해서는 분산락을 위한 클러스터를 만들어줘야 하는데, 그건 오버엔지니어링으로 다가올 수 있습니다. 결론적으로 현재 상황에는 여러 락 매커니즘 중 DB 락 기반의 비관적락이 가장 적절하다고 생각했습니다.

개발자로서 새로운 기술을 적극적으로 학습하고 서비스에 적용하는 것도 중요한 자세입니다. 하지만 그것보다 더 중요한건 현재 상황에 가장 비용이 적으면서 효율적인 방법을 찾아서 **오버엔지니어링**을 막는 것이라 생각합니다. 





[nodejs]: https://nodejs.org/
[starter]: https://github.com/cotes2020/chirpy-starter
[pages-workflow-src]: https://docs.github.com/en/pages/getting-started-with-github-pages/configuring-a-publishing-source-for-your-github-pages-site#publishing-with-a-custom-github-actions-workflow
[latest-tag]: https://github.com/cotes2020/jekyll-theme-chirpy/tags
