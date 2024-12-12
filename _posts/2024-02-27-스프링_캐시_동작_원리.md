---
title: 스프링 캐시 동작 과정 분석
author: jemlog
date: 2024-01-05 00:20:00
categories: [Spring]
tags: [spring]
pin: false
image:
  path: '/assets/img/spring.webp'
---

프로젝트에 스프링 캐시를 도입하는 과정에서 어떤 방식으로 내부 구현이 되어 있는지 궁금해졌습니다. @Cacheable 어노테이션을 사용해서 캐시 조회 기능을 사용했을때 어떻게 동작하는지 위주로 분석해보겠습니다.

캐시 호출 시 가장 먼저 동작하는 클래스는 `CacheAspectSupport`이다. 해당 클래스는 AOP를 사용해 스프링 캐시의 구체 기술을 쉽게 적용할 수 있도록 도와주는 역할을 수행한다. 

![스크린샷 2024-03-21 오전 9 28 25](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/30ad1b65-34d9-4f6f-bae2-214a29dfaef7)

해당 클래스 내의 `execute()` 메서드가 호출되면 `findCacheValue()`라는 내부 메서드가 호출되서 캐시값을 가져온다. `findCacheValue()` 메서드를 좀 더 자세히 살펴보자.

![스크린샷 2024-03-21 오후 12 57 38](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/67c5d261-7bf0-4742-8473-6e4132b455ee)

findCachedValue() 메서드 내부에 findInCaches()라는 메서드가 하나 더 있는걸 볼 수 있다. 해당 메서드 위를 보면 Key를 생성하는 generateKey() 메서드가 있는데, 여기서 만들어진 Key를 사용해서 
캐시값을 조회한다. findInCaches() 메서드 내부로도 한번 들어가보자.

<img width="711" alt="스크린샷 2024-03-21 오후 12 18 25" src="https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/ea8ad48e-beac-4d86-99b6-0af97782328c">

context에서 cache 객체를 조회하는걸 볼 수 있다. 현재 Redis를 분산 캐시로 사용하고 있기에 RedisCache가 반환된다. 

이후 하단의 doGet() 메서드 부분에서 cache 객체를 사용한 캐시 조회가 이뤄진다.

![스크린샷 2024-03-21 오전 1 48 29](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/c7996de2-0df2-4f8d-92c9-d23b7ff1c557)

doGet() 메서드 내에서는 파라미터로 들어온 Cache 객체를 사용해서 조회하는 로직이 있다. get() 메서드의 동작을 살펴보자.

![스크린샷 2024-03-21 오전 11 18 08](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/04d4337d-dbd4-4126-95b4-e892e1664e94)

이 부분에서는 lookup() 부분을 유심히 보자. lookup()은 Redis로부터 네트워크 통신을 통해 직접 데이터를 가져오는 과정과 역직렬화 과정이 포함되어 있다. toValueWrapper()는 응답값을 ValueWrapper로 단순히 래핑하는 역할을 수행하므로 건너뛰겠다.

![스크린샷 2024-03-21 오후 12 14 56](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/b576e869-99f1-479e-aaee-114f6d3f3360)

빨간색 박스 표시한 두 부분이 핵심 로직이다. 첫번째 박스에서는 직접 레디스로부터 캐시 데이터를 읽어오는 역할을 한다. 만약 데이터가 없다면 Cache Miss로 null이 반환된다. 

다음 박스에서는 레디스에서 가져온 JSON 데이터를 객체로 역직렬화 하는 과정을 거친다.

<img width="586" alt="스크린샷 2024-03-21 오후 12 17 22" src="https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/f10a1718-1422-40e8-aabc-d220020165da">

<img width="515" alt="스크린샷 2024-03-21 오후 12 17 10" src="https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/0cb2ecea-dd37-4b1d-b734-4617dc5ad416">

ValueSerializer로는 캐시를 설정할때 등록한 Serializer인 GenericJackson2JsonRedisSerializer가 사용되는걸 알 수 있다.

<img width="700" alt="스크린샷 2024-03-21 오후 12 18 49" src="https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/ae820da8-0722-47f6-b0a8-eb5fafb7f95c">

스프링 캐시는 1차적으로 캐시를 조회한 뒤 캐시 미스가 발생하면 DB에서 조회한 데이터를 캐시에 추가하는 작업까지 함께 수행한다. 이 과정은 evaluate() 메서드를 통해 수행되는데 내부를 살펴보자.

![스크린샷 2024-03-21 오후 12 35 49](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/d6da4ce3-bc28-4ff8-87e7-006ce6e3a90a)

만약 cacheHit이 null일 경우 추가할 데이터를 collect한 다음, 캐시에 실제로 추가하는 작업을 수행하는걸 알 수 있다.

마지막으로 모든 작업을 완료한 뒤 execute 메서드에서 실제 값을 반환한다.

![스크린샷 2024-03-21 오후 12 39 45](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/fd4fce3e-fa53-4fc0-bd47-c3bd68b4bb71)

## 결론

이번 포스팅에서는 스프링 캐시가 내부적으로 어떻게 동작하는지 @Cacheable 어노테이션 기반으로 분석해봤다. 스프링은 캐시, 비동기 등의 구체 기술들을 어노테이션 하나만 붙여서 적용할 수 있도록 AOP를 통해 최적화를 많이 시켜놨다. 따라서 한번쯤은 단순히 사용하는걸 넘어서
이 기술들이 어떻게 동작하는지 구체적인 방식을 살펴볼 필요가 있다고 생각한다.

[nodejs]: https://nodejs.org/
[starter]: https://github.com/cotes2020/chirpy-starter
[pages-workflow-src]: https://docs.github.com/en/pages/getting-started-with-github-pages/configuring-a-publishing-source-for-your-github-pages-site#publishing-with-a-custom-github-actions-workflow
[latest-tag]: https://github.com/cotes2020/jekyll-theme-chirpy/tags
