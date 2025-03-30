---
title: TCP 통신 시 발생 가능한 예외 케이스 분석
date: 2024-01-10 00:20:00
categories: [Network]
tags: [tcp/ip]
pin: false
img_path: '/assets/img'
---

보통 네트워크는 예측할 수 없는 구간이라고 한다. 클라이언트와 서버가 물리적으로 떨어져있는 만큼 중간 노드(ex. NAT, Load Balancer, Switch)들을 경유하고 보안 장비(ex. SG, WAF)를 거치면서 패킷이 필터링 될 수도 있기 때문이다. 또한 서버와 어플리케이션 자체의 상태에 따라서도 에러 처리가 모두 다르다. 

문제가 발생하면 고려해야 하는 경우의 수가 많은 구간이기 때문에 대표적인 케이스들에 대해 인지하고 있는게 문제 해결에 도움이 될 것이라 생각한다. 오늘은 기본적인 문제 상황 몇개를 살펴 보고자 한다. 

## 포트를 정상적으로 리스닝 하지 않는 케이스

만약 서버는 활성화된 상태에서 내부의 어플리케이션만 죽어서 Port를 리스닝하지 않는 상태라면 어떤 에러가 발생할까? 브라우저에서 요청을 보냈을때 이런 화면이 뜨면서 `ERR_CONNECTION_REFUSED`라는 메세지가 나타난다.

<img width="800" alt="스크린샷 2024-01-11 오전 12 09 50" src="https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/203b0dd4-2ff2-40cc-9149-189a4d06bbbb">

그렇다면 내부적으로는 어떤 패킷들이 오고 갈까? `curl`을 통해 요청을 보낸 뒤 WireShark로 패킷을 캡쳐해봤다.

<img width="800" alt="스크린샷 2024-01-11 오전 12 09 50" src="https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/f6e67fd3-594f-4bc8-aaa4-aca468a3c663">

먼저 로컬 PC에서 SYN 패킷과 함께 커넥션을 요청한다. 하지만 서버 상에는 8080포트가 리스닝되고 있지 않기에 OS는 바로 RST 패킷을 반환한다. 로컬 PC는 RST 패킷을 받고 바로 커넥션 연결을 포기한다.

## 서버가 죽은 케이스

그렇다면 서버 자체가 죽은 상황에서는 패킷이 어떻게 동작할까? AWS 클라우드 환경이라면 EC2 서버가 죽은 경우가 될 것이다. 현재 EC2를 종료시킨 뒤 다시 요청을 전송해보겠다.

<img width="800" alt="스크린샷 2024-01-11 오전 12 09 50" src="https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/59e9224f-3176-4712-a20d-840f69822995">

이번에는 RST 패킷이 응답으로 전송되는 대신 `SYN Retransmission`이 계속 발생한다. SYN Retransmission은 초기에는 **1초** 주기로 재전송되다가 이후에는 지수적으로 **interval이 증가한다**. 

재시도가 꽤 오랜 시간 지속되는걸 알 수 있는데 당연히 서비스 운영 환경에서는 이정도 시간을 기다릴 수 없다. 따라서 어플리케이션에서 커넥션 타임아웃을 설정해야 하고 보통 첫 재시도 1초를 감안하여 **3초** 정도로 설정한다.

### Ping을 사용한 IP 테스트

네트워크에 문제가 발생했을때 애플리케이션보다 먼저 체크해야 하는건 서버의 활성화 여부일 것이다. 서버의 상태는 서버 노드의 고유한 주소인 IP를 테스트 함으로써 판단 가능하다. 보통 `ping` 명령어를 사용해 원격지 서버의 상태를 체크한다.

ping은 **ICMP**라는 프로토콜을 사용한다. ICMP는 네트워크 장치에서 네트워크 통신 문제를 진단하는데 사용하는 네트워크 계층 프로토콜이다. traceroute와 ping 명령어 모두 ICMP 프로토콜을 사용한다.

```shell
ping www.naver.com
```

ping 사용법은 아주 간단하다. 그냥 ping 명령어 뒤에 요청하고자 하는 도메인이나 IP를 명시하면 된다. 

<img width="500" alt="스크린샷 2024-01-11 오전 12 09 50" src="https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/b3483087-2225-4fb2-bced-bad61d83c787">

위의 사진처럼 요청이 보내지고 packet loss가 0%라면 원격지 서버는 잘 살아있다는 말이다. 만약 ping이 실패하면 생각해볼만한 원인은 크게 **3가지**가 있다.

1. ping을 보내는 클라이언트가 네트워크에 연결이 안 되있다.
2. AWS의 Security Group에서 ping을 막고 있다.
3. 서버가 죽은 상태이다.

1번째는 클라이언트가 대표적인 도메인인 구글이나 네이버에 ping을 한번 쏴봄으로써 확인이 가능하다. 만약 구글에 ping이 안 보내지면 클라이언트 자체에 문제가 있다고 판단할 수 있다. 만약 ping이 잘 보내진다면 이제 2번째 이유를 고민해봐야한다.

AWS SG는 화이트리스트 기반 보안 솔루션으로써 ping의 프로토콜인 ICMP도 기본적으로 막고 있다. 따라서 ping 테스트를 성공하려면 원격지 서버 앞단 SG의 보안그룹에 ICMP를 추가해줘야 한다.

<img width="800" alt="스크린샷 2024-01-13 오후 8 16 31" src="https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/9257e332-bb4a-467d-8215-c0f64dca7ed7">

SG의 보안그룹 추가는 간단하다. 위의 사진처럼 ICMP를 허용으로 등록하면 이제 SG에서 ping 테스트가 막힐 일은 없다. 그 다음 3번째는 위에서 살펴본 서버 예시를 생각하면 된다.

## 패킷이 서버로 들어오기 전에 막히는 케이스

서버가 죽었을 경우 SYN Retransmission이 발생하는걸 위에서 확인했다. 그렇다면 SYN Retransmisson이 발생하는 상황은 이거 하나만 있을까? 만약 재전송이 발생하길래 서버를 확인해봤는데 정상 동작하고 있다면 뭘 더 확인해볼 수 있을까?
필자는 **패킷 자체가 서버로 들어왔는지** 여부를 체크해봐야 한다고 생각한다. 

EC2 서버에서 `tcpdump`를 떠서 패킷이 서버로 들어왔는지를 체크할 수 있다.

```shell
tcpdump tcp port 8080 -w tcpdump.log # 8080 port로 들어온 패킷을 분석해서 tcpdump.log 파일로 출력
```
이때 tcpdump 파일을 분석했을때 기록이 남아있지 않다면 서버로 아예 패킷이 들어오지 않았다고 판단할 수 있다.

```shell
scp -i [로컬에 있는 pem key] ec2-user@[서버 퍼블릭 IP]:[EC2 내의 tcpdump 파일 경로] [파일을 다운받고자 하는 로컬의 경로]
scp -i test.pem ec2-user@13.11.23.250:/home/ec2-user/tcpdump.log ./
```
로컬의 wireshark에서 분석하려면 EC2 내의 덤프 파일을 로컬로 가지고 와야 한다. `scp` 명령어를 통해서 로컬로 파일을 복사할 수 있다.

그렇다면 서버로 패킷이 들어오기 전에 막히는 케이스는 대표적으로 뭐가 있을까? 필자는 AWS의 Security Group을 떠올렸다. 현재 사용하고 있는 8080 포트를 SG에서 막은 후 다시 요청을 보내보겠다.

<img width="800" alt="스크린샷 2024-01-11 오전 12 09 50" src="https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/cbdab9ee-2dc0-48b3-aa52-5eb5530745f6">

서버가 죽었을 경우와 똑같이 SYN Retransmisson이 발생하는걸 알 수 있다.

## 결론

TCP 통신을 하면서 발생할 수 있는 예외 상황은 매우 다양할 것이다. 그리고 이 부분은 경험으로 채워야 하는 부분이라 생각하기에 앞으로 문제를 겪을때마다 내용을 보강할 예정이다.







[nodejs]: https://nodejs.org/
[starter]: https://github.com/cotes2020/chirpy-starter
[pages-workflow-src]: https://docs.github.com/en/pages/getting-started-with-github-pages/configuring-a-publishing-source-for-your-github-pages-site#publishing-with-a-custom-github-actions-workflow
[latest-tag]: https://github.com/cotes2020/jekyll-theme-chirpy/tags
