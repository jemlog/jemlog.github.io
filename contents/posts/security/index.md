---
title: "@AuthenticationPrincipal에 null값 들어오는 문제 해결"
description:
date: 2023-06-03
update: 2023-06-03
tags:
- project
- spring security
series: "spring"
---

@Authentication을 통한 인증 유저를 가져오는 과정에서 겪은 문제와 제가 해결한 방법에 대해 공유하고자 합니다. 
![img1](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/e6adb87a-708e-4ba2-8d61-b3318ef30f73)

@AuthenticationPrincipal 어노테이션의 사용법을 구글에 검색했을때, 많은 블로그에서 UserDetailsService를 구현한 CustomUserDetailsService의 loadUserByUsername 메서드에서 반환해준 값을 파라미터로 직접 받아 사용할 수 있다고 알려줬습니다. 
![img2](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/bc0ed9b2-f97a-46b1-abbb-482902e1aa12)


loadUserByUsername 메서드에서 반환하는 UserPrincipal 객체입니다. UserDetails를 구현하는 방식으로 만들었습니다. 전체 코드 중 일부를 가져온 것이기 때문에  '이런 객체를 반환하도록 만들었구나' 정도만 생각해주면 감사하겠습니다. 
![img3](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/cea8d971-2707-4c54-a18d-3089639e4d61)

이제 Controller에서 @AuthenticationPrincipal을 통해 loadUserByUsername 메서드가 반환해준 userPrincipal 객체에 접근해보려고 합니다. 제 예상대로라면 인증 성공의 결과로 DB에 저장된 유저의 ID가 정상적으로 출력되야했습니다. 하지만 결과는 **null**이 나왔습니다. 

전혀 예상하지 못한 결과였기 때문에 @AuthenticationPrincipal 관련 글들을 정말 많이 찾아봤습니다. 저는 마땅한 해결책을 얻지 못했고, 결국 @AuthenticationPrincipal 어노테이션이 어떤 원리로 파라미터에 값을 넣어주는지 직접 분석해보기로 마음 먹었습니다. 
![img4](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/07333824-bbd8-4721-ac41-98aabc198a57)

우선 @AuthenticationPrincipal 어노테이션 자체를 살펴보겠습니다. 이 어노테이션이 붙은 파라미터에 값을 주입해주는 ArgumentResolver의 이름이 AuthenticationPrincipalArgumentResolver인 것을 알 수 있습니다. 한번 내부로 들어가보겠습니다. 
![img5](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/2ac65cf4-8774-4f66-82f3-ec951a07fc6e)

이 어노테이션에서 중점적으로 봐야할 부분은 2군데 입니다. 

1\. supportParameter 메서드를 통해 @AuthenticationPrincipal이라는 어노테이션이 있는지 체크합니다.

2\. 위의 supportParameter의 값이 true라면 resolveArgument에서 파라미터에 값을 주입해줍니다.

위의 그림에도 볼 수 있듯이, resolveArgument에서는 SecurityContextHolder.getContext().getAuthentication()의 값을 가져와서 파라미터에 주입해 주는 걸 알 수 있습니다. 엄밀히 말하면 CustomUserDetailsService의 loadUserByUsername의 반환값과는 다른 값을 제공해 주는 것이였습니다. 

이제 문제 해결의 실마리가 잡힌 것 같습니다. SecurityContextHolder에 인증 정보를 넣어주는 로직을 찾아가보겠습니다. 
![img6](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/c66a37e9-86bb-4d85-a16e-c8f40e699568)

현재 진행중인 프로젝트에서 JWT와 OAuth2 방식의 로그인을 구현하고 있기에 OnceRerRequestFilter를 상속한 TokenAuthenticationFilter를 사용하고 있습니다. 아래의 tokenProvider의 getAuthentication 메서드를 통해 어떤 객체가 생성 되는지를 확인해보겠습니다. 
![img7](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/9fe61737-9db1-433f-bb37-513d8b960223)

드디어 @AuthenticationPrincipal 사용 시 파라미터가 제대로 주입 되지 않는 이유를 발견했습니다. 저는 인증 객체를 저장 하는 과정에서 매번 DB에 접근해서 User 정보를 가져오는 대신, 토큰에서 추출한 정보만으로 인증 객체를 만들었습니다. UsernamePasswordAuthenticationToken에 인자로 들어가는 principal은 loadUserByUsername의 반환 타입인 UserPrincipal과 내부 데이터도, 타입 자체도 다른 걸 알 수 있습니다.

문제를 해결하기 위해 getAuthentication 메서드를 수정해보겠습니다. 
![img8](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/d41cf262-a26a-4726-b46f-31d781006cd4)

이제 SecurityContextHolder 내부에는 제가 CustomUserDetailsService의 loadUserByUsername 메서드가 반환한 UserPrincipal 객체가 저장됩니다. 실제 테스트를 진행했을때도 값이 정상적으로 출력됐습니다. 

이번 이슈를 해결해는 과정을 통해 직접 코드 내부를 분석해보는 과정이 얼마나 중요한지 느끼게 되었습니다. 또한 리팩토링을 하는 과정에서 제 코드에 대한 이해도도 훨씬 높아지게 되었습니다. 제 글이 저와 같은 문제로 고민하고 있는 분들에게 도움이 됐으면 좋겠습니다.