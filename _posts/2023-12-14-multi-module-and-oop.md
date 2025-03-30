---
title: 멀티 모듈 분리와 객체 지향 설계에 대한 고민
author: jemlog
date: 2023-12-06 20:55:00 +0800
categories: [Project]
tags: [spring]
pin: false
image:
  path: '/assets/img/spring.webp'
---

프로젝트를 리팩토링하면서 좋은 구조는 무엇일지에 대해 고민을 많이 했습니다. 

전체 구조를 여러번 수정하면서 제 나름대로 내린 결론은 **확장에 열려있고 적절한 격리를 통해 유지 보수 시 코드 변경 지점을 최소화 하는게** 좋은 구조라 생각하게 됐습니다.

지금부터 제 생각을 프로젝트에 녹여본 경험을 공유하고자 합니다.

## 멀티 모듈 구조 도입

### Melly 프로젝트 멀티 모듈

<img width="633" alt="스크린샷 2023-11-07 오후 7 37 00" src="https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/3b509f07-e7f4-4553-b81b-9a38638490af">

저는 프로젝트를 구성하는 기능의 종류를 기준으로 멀티 모듈을 분리했습니다. 1차적으로 비슷한 기능끼리 대분류를 한 뒤, 의존성과 역할을 기반으로 모듈을 세분화했습니다.

이 방식을 통해 **라이브러리 의존성간의 응집성**을 높이고 **모듈별로 독립적인 확장**을 해나갈 수 있습니다. 모듈별 의존성을 Gradle의 **Implementation**으로 주입하면 Compile 타임에 모듈을 사용하는 컨슈머 모듈에 의존성이 침범되는걸 방지할 수 있습니다.

구조를 개선하면서 참고한 프로젝트 중에는 도메인 별로 db 모듈을 모두 분리하고(ex. db-core-user, db-core-memory) 아예 Persistance 계층과 순수 도메인 모듈을 분리해서 설계한 경우도 있었습니다. 물론 가장 이상적인 방식이라고 생각합니다. 하지만 아직 프로젝트가 성숙하지 않은 상황에서
모듈을 엄청 세분화하면 오히려 코드 양이 급격히 늘어나고 관리가 더 힘들어질 수도 있습니다.

따라서 서비스의 규모와 상황에 따라 필요할때 모듈 분리를 진행하는게 좋다고 생각합니다. 

```bash
├── client:client-auth            # OAuth 리소스 서버와 통신하는 Client 모듈 (현재 OpenFeign 의존성 사용)
├── core:core-api                 # 모바일 클라이언트와 통신하는 API 모듈    
├── storage:db-core               # MySQL 기반의 저장소 모듈
├── storage:db-redis              # Redis 기반의 인메모리 저장소 모듈
├── infra:file                    # 파일 저장소 모듈 (현재 S3 의존성 사용)     
├── infra:mail                    # 메일 서비스 모듈 (현재 Java Mail 의존성 사용)
├── infra:notification            # 알림 서비스 모듈 (현재 FCM 의존성 사용) 
└── support:logging               # 로깅 모듈          
```
전체 구조는 위와 같이 분리를 했습니다. 지금부터는 몇 가지 세부 모듈에서 고민한 내용을 말씀드리겠습니다.

## Client 모듈 객체 지향 설계 고민 사항

현재 프로젝트는 OAuth 인증을 도입했고 카카오,네이버,구글 그리고 애플 로그인을 지원합니다. OAuth 인증 수단은 이후에 추가/삭제가 발생할 수 있습니다. 이때 변경사항이 발생해도 코드 수정을 최소화 할 수 있는 구조를 만들기 위해 고민했습니다.

### Factory 객체를 사용한 인증 클라이언트 조회

아래의 도식은 OAuth 인증 기능의 객체 구성입니다.

![dsfd drawio](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/9ccfee01-73b5-4002-978b-5246ae53c1dd)


OAuth 제공사별 통신 클라이언트를 유연하게 선택하기 위해 `LoginClient`라는 인터페이스를 구현했습니다.
- supports : 클라이언트가 어떤 OAuth 제공사인지 판별하는 메서드
- getUserData : OAuth 제공사 리소스 서버에서 데이터를 받아오는 메서드

```java
public interface LoginClient {

  boolean supports(String provider);

  LoginClientResult getUserData(String accessToken);
}
```

클라이언트 모듈에서는 `LoginClientFactory`라는 객체를 직접 사용합니다. `LoginClientFactory`는 find 메서드를 통해 원하는 Provider의 통신 클라이언트를 조회합니다. 이때 한번 조회한적이 있는 클라이언트는 빠르게 조회하기 위해 메모리 Map에 캐싱하는 방식을 채택했습니다.

```java
@Component
@RequiredArgsConstructor
public class LoginClientFactory {

   private final List<LoginClient> loginClientList;

   /*
   한번 생성된 Client 객체를 런타임 내에 메모리상에 캐싱합니다
   */
   private final Map<Provider, LoginClient> factoryCache = new HashMap<>();

   public LoginClient find(Provider provider) {
	LoginClient loginClient = factoryCache.get(provider);
	if (loginClient != null) {
		return loginClient;
 	}

	loginClient = loginClientList.stream()
		.filter(client -> client.supports(provider.name()))
		.findFirst()
		.orElseThrow();

	factoryCache.put(provider, loginClient);
	return loginClient;
   }
}
```

**개선 효과**

해당 구조를 사용하면 차후 **FaceBookLoginClient**가 추가되더라도 사용하는 쪽(ex. OAuthService)의 코드는 변경될 일이 없습니다. Client를 자유롭게 추가할 수 있는 **개방성**과 사용하는 쪽의 코드를 수정하지 않아도 된다는 **폐쇄성**을 통해 객체지향 원칙의 **OCP**(개방 폐쇄 원칙)을 준수합니다.

```java
@Transactional
public OAuthResponseDto login(OAuthLoginRequestDto oAuthLoginRequestDto) {

  LoginClient loginClient = loginClientFactory.find(oAuthLoginRequestDto.getProvider()); // Provider를 통해 원하는 통신 클라이언트를 선택
  LoginClientResult socialUser = loginClient.getUserData(oAuthLoginRequestDto.getAccessToken()); // 통신 클라이언트를 통해 유저 데이터를 조회
  
  ... 
}
```

### 멀티 모듈과 default 접근 지시자를 통한 사용한 인증 클라이언트 외부 노출 방지
현재 프로젝트는 통신 클라이언트로 OpenFeign을 사용하고 있습니다. 하지만 전체 서비스 입장에서는 통신 클라이언트가 RestTemplate인지 OpenFeign인지는 관심사가 아닙니다. 
사용하는 측에서는 통신 클라이언트를 감싸고 있는 Client 객체를 통해서만 외부와 통신을 진행하면 됩니다.

만약 멀티 모듈이 아닌 싱글 모듈로 프로젝트를 구성했다면 OpenFeign 의존성을 컴파일 타임에 프로젝트 어디서든 사용 가능합니다. 물론 의존성 관리에 대한 규칙을 잘 정해둔다면 상관 없겠지만, 프로젝트가 복잡해지면 언제 의존성이 코드베이스 전반에
흩어질지 모릅니다. 이를 방지하기 위해 Client 모듈을 분리해서 통신 클라이언트를 사용하는 모듈 (core-api) 쪽에서는 아예 컴파일타임에 OpenFeign을 import 할 수 없도록 구성했습니다. 

OpenFeign을 사용하기 위해서는 openfeign 인터페이스를 감싸고 있는 Client 객체(ex. KakaoLoginClient, NaverLoginClient)를 사용해야 합니다. 저는 OpenFeign 의존성을 Client 객체를 통해서만 사용하고 싶기에 OpenFeign 인터페이스의 접근 지시자를 package default로 변경했습니다.
이 구조를 통해 통신 클라이언트 기능이 필요한 외부 모듈은 어떤 구체 기술을 사용하는지에 상관 없이 Client 객체만 가지고 와서 사용하면 됩니다. 구체 기술의 변경이 필요하면 Client 모듈만 수정하면 됩니다.

```java
// package default로 접근 지시자를 설정
@FeignClient(name = "kakaoLogin", url = "${client.oauth.uri.kakao}")
interface KakaoLoginApi {

  @GetMapping(produces = MediaType.APPLICATION_JSON_VALUE, consumes = MediaType.APPLICATION_JSON_VALUE)
  KakaoResource call(@RequestHeader(value = "Authorization") String accessToken);
}
```

### 엔티티 대신 DTO 사용을 통한 모듈간 의존성 분리

Client 클래스는 기존에 `getUserData` 메서드 내에서 User 엔티티를 생성 후 반환하는 방식으로 구현됐었습니다.

```java
public class KakaoClient implements LoginClient {

  private static final String AUTH_PREFIX = "Bearer ";

  private final KakaoLoginApi kakaoLoginApi;
  
  ...
  
  /*
  기존 코드는 getUserData 메서드가 User 엔티티를 반환함 
  */ 
  @Override 
  public User getUserData(String accessToken) {
     return kakaoLoginApi.call(AUTH_PREFIX + accessToken).toResult();
  }
}
```
이렇게 구현하기 위해서는 Compile 타임에 User 엔티티가 필요했기에 client-auth 모듈에서 db-core 모듈을 implementation했습니다.

```gradle
dependencies {

    implementation 'org.springframework.cloud:spring-cloud-starter-openfeign'
    implementation 'io.github.openfeign:feign-hc5' // Open Feign의 클라이언트로 Apache HttpClient5를 사용합니다

    ...

    implementation project(":storage:db-core") // User 엔티티가 Compile 타임에 필요함
}
```

client-auth 모듈은 db-core 모듈 중 User 엔티티 하나만 필요함에도 모듈 전체를 컴파일 타임에 포함해야 합니다. 이는 결과적으로 컴파일 타임을 증가시킵니다. 따라서 User 엔티티를 반환하는 대신 DTO를 반환함으로써 db-core 모듈에 대한 의존성을 제거했습니다.


## Infra 모듈 객체 지향 설계 고민 사항

Infra 디렉토리는 File, Mail, Notification 3개의 모듈로 세분화됩니다. 기능에 따라 모듈을 분리함으로써 라이브러리 의존성을 응집할 수 있고 독립적인 확장이 가능합니다. Infra 모듈을 구현할때는 **구체 기술의 유연한 확장**에 집중했습니다.

<img width="452" alt="스크린샷 2023-11-13 오전 10 47 42" src="https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/c3473715-2534-45ed-ae39-18e719b7ed2b">


### 개선 내용

3가지 모듈 모두 같은 고민 사항에 초점을 두고 제작했기에 예시로 **Mail 모듈**을 분석합니다.

Mail 모듈은 외부와 소통하는 인터페이스이자 부가 로직을 추가할 수 있는 `EmailService`와 메일 전송 기능만을 수행하는 `EmailSendClient`로 구성되있습니다. EmailSendClient는 Java Mail 기반이 될 수도 있고 AWS SES 기반이 될 수도 있기에 인터페이스를 통해 다형성을 구현했습니다.

```java
@Component
@RequiredArgsConstructor
public class EmailService {

  private final EmailSendClient emailSendClient;

  public void send(String subject, String content, String... to) {
      emailSendClient.sendMail(subject, content, to);
  }

}
```
구현 객체인 `JavaMailSendClient`의 경우 **Package Default** 접근 지시자를 통해 외부에서 구체 기술에 의존하는걸 방지했습니다.

```java
// Package Default 접근 지시자로 외부에서 임의로 JavaMailSendClient에 접근하는걸 막음
@Component
@RequiredArgsConstructor
class JavaMailSendClient implements EmailSendClient {

  private final JavaMailSender mailSender;

  @Override
  public void sendMail(String subject, String content, String... receivers) {
	log.info("{} send mail to {}", subject, receivers);
	mailSender.send(createMessage(subject, content, receivers));
  }
}
```

## 결론

지금까지 프로젝트의 의존성을 통제하고 확장에 열린 구조를 만들기 위한 과정을 몇 가지 모듈을 예시로 살펴봤습니다. 프로젝트의 규모가 커질수록 코드 또한 관리하기
쉬운 방향으로 끊임없이 성장해야 한다고 생각합니다. 개발자는 사용자 경험을 향상시키기 위해 기존의 기능을 개선하고 새로운 기능들도 추가해야 합니다. 

저는 여기에 추가적으로 **동료들의 경험을 향상시키는 노력**도 해야 한다고 생각합니다. 이를 위해서는 현재의 팀원들과 미래의 팀원들 모두가 코드를 잘 유지보수하고 개선해갈 수 있는 유연하고 가독성 높은 코드를 고민하는게 중요하다고 생각합니다.


[nodejs]: https://nodejs.org/
[starter]: https://github.com/cotes2020/chirpy-starter
[pages-workflow-src]: https://docs.github.com/en/pages/getting-started-with-github-pages/configuring-a-publishing-source-for-your-github-pages-site#publishing-with-a-custom-github-actions-workflow
[latest-tag]: https://github.com/cotes2020/jekyll-theme-chirpy/tags
