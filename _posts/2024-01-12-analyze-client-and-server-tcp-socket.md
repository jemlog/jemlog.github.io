---
title: 클라이언트 소켓과 서버 소켓 운영 주의 사항 및 개선점 분석
date: 2024-01-10 00:20:00
categories: [Network]
tags: [tcp/ip]
pin: false
img_path: '/assets/img'
---

TCP에 대해 학습을 진행하던 중 클라이언트 입장에서의 소켓과 서버 입장에서의 소켓이 동작하는 방식이 달라진다는 걸 알게되었다. 서비스 운영에 있어서 중요한 고려 지점이 될 것이라 생각해 좀 더 깊게 파해쳐보기로 했다.

## 클라이언트 소켓과 서버 소켓의 차이

시스템의 규모가 확장되고 비즈니스가 복잡해질수록 다른 서버와의 통신도 잦아진다. 이때 다른 서버가 내 서버에 요청을 보내면 서버 소켓을 사용해서 요청을 받고, 내 서버가
다른 서버로 요청을 보낼때는 클라이언트 소켓을 사용한다. 그렇다면 서버 소켓과 클라이언트 소켓의 차이점은 무엇일까?

<img width="800" alt="스크린샷 2024-01-14 오후 6 40 46" src="https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/75319171-14ab-49ad-84cb-ca37d09a1501">


가장 큰 차이점은 **로컬 포트**를 사용하는 방식과 개수라고 생각한다. 

서버 소켓의 경우 어플리케이션을 실행할때 설정한 **로컬 포트 딱 하나만 사용**한다. 스프링 어플리케이션을 실행하는 경우 기본 포트인 8080 포트가 될 것이다. 외부에서 여러 클라이언트가 요청을 보내도 소켓 개수만 늘어날뿐 사용하는 로컬 포트의 개수는 단 하나이다.

반면 다른 서버로 요청을 보내는 클라이언트 소켓의 경우, 소켓을 생성할때 OS 상의 로컬 포트 중 랜덤으로 하나를 선택해서 소켓에 매핑한다. 즉, 소켓을 생성할때마다 매번 다른 로컬 포트가 사용된다.

그렇다면 내 서버에서 외부로 향하는 트래픽이 늘어날수록 소켓 생성과 비례해 사용되는 로컬 포트가 증가한다는 말인데, 이게 무슨 문제가 되는 것일까?

### 클라이언트 소켓과 TCP TIME_WAIT 상태의 관계

만약 하나의 요청을 처리하기 위해 소켓이 생성된 후, 응답을 받은 뒤 소켓을 닫는다고 가정해보자. 그러면 우리는 소켓이 할당됐던 로컬 포트를 바로 재사용할 수 있을까? 정답은 **안된다**이다. 

그 이유는 TCP 소켓에는 `TIME_WAIT` 상태가 있기 때문이다. TIME_WAIT는 TCP의 종료 과정인 4-way handshake에서 먼저 TCP 연결 해제를 희망하여 FIN 패킷을 상대방에게 보낸 쪽이 마지막에 진입하는 상태다. 이 상태의 지속시간 설정값은 `2MSL`이고 OS에 하드코딩 되어있는 값이기 때문에 설정값을 변경하는건 불가능하다.

<img width="641" alt="스크린샷 2024-01-14 오후 6 46 29" src="https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/de88e9d2-472f-47d2-a527-37cb37751fbd">

TIME_WAIT 상태에 빠진 소켓에 매핑된 로컬 포트는 TIME_WAIT 상태가 끝날때까지 다른 소켓에 할당되지 못한다. 그렇다면 트래픽이 많이 들어오는 상황에서 어떤 문제가 있는지 느낌이 오는가? 만약 동시접속자가 1000명이 들어오면 총 1000개의 로컬 포트가 소켓에 할당되고 TIME_WAIT가 끝나기 전까지는 로컬 포트 1000개를 사용하지 못하는 것이다. 

트래픽이 작은 서비스는 문제가 없겠지만 규모가 큰 서비스의 경우 로컬 포트 고갈로 더이상 클라이언트 소켓을 생성하지 못하는 상황에 다다를 것이다. 

### tw_reuse와 timestamp를 사용한 로컬 포트 빠른 재활용

그렇다면 우리는 TIME_WAIT가 진행되는 2MSL의 시간동안 무조건 기다려야 하는 걸까? 다행히도 이 시간을 기다리지 않아도 TIME_WAIT 상태인 포트를 재활용할 수 있는 Linux 옵션이 존재한다. 바로 `net.ipv4.tw_reuse` 속성이다.

TW_REUSE 옵션을 활성화하면 로컬 포트가 부족할때 TIME_WAIT 상태인 로컬 포트를 재활용할 수 있도록 만들어준다. 이때 통신을 하는 양측 모두 TCP timestamp 옵션이 활성화되있어야 한다.

TW_REUSE가 동작하는 방식은 TIME_WAIT 상태에 들어간 소켓의 진입 시간을 기록한 후, 새로운 소켓을 생성해야 하는데 로컬 포트가 부족한 경우 생성된지 현재 시간으로부터 1초 이상 지난 TIME_WAIT 포트를 재활용한다. 따라서 이 기능을 사용하기 위해서는 TCP TIMESTAMP가 필수적이다.

<img width="532" alt="스크린샷 2024-01-14 오후 9 12 49" src="https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/d8d4523c-a610-4759-8148-f4e36d2e0e6d">

아래의 명령어를 통해서 TW_REUSE를 활성화할 수 있다.

```shell
sysctl -w net.ipv4.tcp_tw_reuse="1" # 1로 설정해서 활성화
```


### 커넥션 풀을 사용한 로컬 포트 고갈 방지

위의 tw_reuse를 사용하면 로컬 포트가 고갈되는걸 어느정도 방지할 수 있을 것이다. 하지만 조금 더 근본적인 해결책을 찾아야 한다. 

TIME_WAIT 상태의 소켓이 꾸준히 증가하는건 다른 서버에 요청을 할때마다 커넥션을 새로 생성하고, 이를 위한 소켓이 새로 생성되면 로컬 포트가 할당되기 때문이다. 그러면 우리는 어떤 방법을 사용할 수 있을까. 어플리케이션에서 **커넥션풀**을 생성해서 정해진 개수 만큼만 계속 재사용하면 된다.

어플리케이션과 관련된 부분이기 때문에 직접 구현을 해서 자세히 살펴보자. 테스트 방법은 클라이언트 어플리케이션은 로컬 환경에서 동작시키고 서버 어플리케이션은 EC2에서 동작시키는 방법으로 실행하겠다. `Spring Cloud OpenFeign`과 `Apache Http Cient 5`를 통신 클라이언트로 사용하고, 각 서버의 소켓 상태를 `netstat`로 분석해보겠다.

먼저 httpclient에 커넥션풀을 사용하지 않은 상태에서 테스트를 진행해보자. 

```yaml
spring:
  cloud:
    openfeign:
      client:
        config:
          default:
            connectTimeout: 3000
            readTimeout: 3000
            loggerLevel: full
      httpclient:
        hc5:
          enabled: true
```

위의 설정을 통해 openfeign과 apache httpClient를 사용할 수 있다.

```shell
#!/bin/bash

# 병렬로 요청을 보낼 URL 목록
# 총 20개 할당
urls=(
"http://localhost:8080/users"
"http://localhost:8080/users"
"http://localhost:8080/users"
"http://localhost:8080/users"
"http://localhost:8080/users"
"http://localhost:8080/users"
... 
)

for i in "${!urls[@]}"; do
  url="${urls[$i]}"
  curl -s -X GET -o /dev/null "$url" &
done

wait
```
정확한 테스트를 위해 요청을 병렬로 보내는 스크립트를 작성했다. 총 20번의 요청을 병렬로 보내고 저의 예상으로는 클라이언트 소켓도 총 20개가 ESTABLISH 상태로 생성되있어야 한다. 스크립트를 실행한 결과를 살펴보자.

<img width="800" alt="스크린샷 2024-01-14 오후 10 30 21" src="https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/678a0521-1c13-479a-825a-5933fa89f0f3">

예상대로 총 20개의 소켓이 ESTABLISHED 상태로 유지되는걸 알 수 있다. 만약 동시 요청 개수가 몇천개 몇만개가 된다면 로컬 포트 고갈로 이어질 것이다. 그렇다면 이번에는 커넥션풀을 적용해보자.

```yaml
httpclient:
  hc5:
    enabled: true         
    pool-reuse-policy: fifo
    pool-concurrency-policy : strict
  max-connections: 10 # httpClient의 전체 커넥션 풀 사이즈, default 200
  max-connections-per-route: 5 # route(url) 별 최대 커넥션 개수, 확인이 필요한 부분, default 50
```

<img width="800" alt="스크린샷 2024-01-14 오후 10 35 21" src="https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/9cded33c-e960-4def-91d9-47f8ba701f69">

테스트 결과 예상대로 동시 요청이 20개임에도 총 5개의 소켓만 ESTABLISHED 된걸 알 수 있다. 요청이 아무리 많아도 생성되는 소켓 개수를 조절할 수 있는 것이다.

그리고 실험을 진행하는 중 한가지 알아낸 점이 있다. 커넥션풀 사이즈를 10으로 설정해도 어플리케이션이 처음 시작할때 **10개가 미리 채워지지 않는다**는 점이다. httpClient의 커넥션풀의 경우 초기에는 빈 상태이다가 트래픽이 들어오면 그때 커넥션을 생성한 후에 풀에 넣어서 재활용을 한다. 


### 서버 소켓과 TIME_WAIT 상태의 관계

그렇다면 서버 소켓의 경우 TIME_WAIT 상태의 지장을 받지 않는가? TCP TIME_WAIT와 관련된 여러 아티클들을 분석한 결과 **별로 지장이 없다**로 결론을 냈다. 

애초에 서버 소켓은 로컬 포트를 하나만 사용하기 때문에 로컬 포트 고갈이라는 문제에서 자유롭다. 또한 약 65535개가 존재하는 로컬 포트와 다르게 소켓은 몇십만개를 생성할 수 있다. 따라서 TIME_WAIT가 서버 소켓에 영향을 미칠 가능성은 적다.

그래도 서버 소켓과 TIME_WAIT 간의 관계에서 주의해야 하는 속성이 있는지 찾아보자. 

```shell
net.ipv4.tcp_max_tw_buckets = 4096
```
해당 속성은 TIME_WAIT 상태의 소켓 개수를 제한하는 파라미터이다. Amazon Linux EC2에서 해당 설정값을 확인한 결과 **4096개**가 나왔다. 만약 이 수치보다 더 많은 TIME_WAIT 상태의 소켓이 생성되려고 하면 어떻게 될까?
이후에 생성되는 소켓들은 TIME_WAIT 상태로 넘어가지 않고 곧바로 파괴되버린다. `/var/log/messages` 디렉토리에 로그 메세지가 남는다.

```shell
TCP : time wait bucket table overflow
```

TIME_WAIT 상태는 TCP의 안전한 종료를 위해 존재하기 때문에 급작스럽게 소켓이 종료되면 데이터 유실이 발생하거나 혼란이 올 수 있다. 따라서 해당 값은 넉넉하게 설정하는게 중요하다. 또한 애초에 time_wait 상태가 지나치게 많다는건 불필요한 연결 맺기와 끊기가 많다는 의미이기 때문에 해결방법을 찾아야 한다.


## 결론
소켓 통신에 대한 여러가지를 학습하고 글로 풀어봤다. 여러 서비스가 네트워크로 연계되서 동작하는 MSA 환경이 주류가 된 만큼 네트워크 통신에 대한 이해도는 날이 갈수록 중요해질 것이라 생각한다.

솔직히 아직 큰 규모의 프로젝트를 경험해보지 않았기에 소켓과 관련된 문제들을 당장 경험해볼 기회는 없다고 생각한다. 하지만 이런 네트워크 예외 상황에 대한 개인적인 호기심이 크고, 잘 동작하는 것 처럼 보이는 네트워크 환경 속에서 어떤 문제들이 발생할 수 있을까 상상해보는게 재밌게 느껴진다. 그리고 정답이 아닐지라도 문제에 대한 해결방법이 나의 지식 속에서 발견될때 스스로 성장했다는 작은 쾌감을 느끼는 것 같다. 그것이 물론 앞으로 더 나아갈 원동력이 되준다.

앞으로 네트워크에 대한 경험치를 쌓아가면서 이 글도 지속적으로 수정해나갈 계획이다.



## Reference
- [https://brunch.co.kr/@dreaminz/5](https://brunch.co.kr/@dreaminz/5)
- [https://meetup.nhncloud.com/posts/55](https://meetup.nhncloud.com/posts/55)
- [https://brunch.co.kr/@alden/3](https://brunch.co.kr/@alden/3)
- [https://docs.likejazz.com/time-wait/](https://docs.likejazz.com/time-wait/)

[nodejs]: https://nodejs.org/
[starter]: https://github.com/cotes2020/chirpy-starter
[pages-workflow-src]: https://docs.github.com/en/pages/getting-started-with-github-pages/configuring-a-publishing-source-for-your-github-pages-site#publishing-with-a-custom-github-actions-workflow
[latest-tag]: https://github.com/cotes2020/jekyll-theme-chirpy/tags
