---
title: Istio의 mTLS를 사용한 제로 트러스트 구축
author: jemlog
date: 2024-06-25 00:20:00
categories: [Istio]
tags: [istio, network]
pin: false
image:
  path: '/assets/img/istio_back.png'
---

최근 서비스 운영 환경은 쿠버네티스 클러스터 내에 마이크로서비스를 배포하는 방식이 메인스트림이 되었다고 생각합니다. 그만큼 사용자의 요청이 인그레스를 통해 들어오면 클러스터 내에서 여러 서비스 간 네트워크 통신을 거쳐 응답이 반환되는 구조입니다. 보통 클러스터로 진입하기 이전까지는 HTTPS 암호화 통신이 보장되지만 클러스터 내부로 들어오면 평문 데이터 통신을 하게 됩니다. 

만약 악의적인 사용자가 인그레스 게이트웨이를 거치지 않고 클러스터 내부로 접근해서 트래픽을 캡쳐한다면 중요한 데이터가 외부로 노출될 위험이 있습니다. 따라서 쿠버네티스 클러스터 내에서도 데이터를 암호화하고 인증받은 클라이언트만 API를 호출할 수 있도록 해야합니다. 

쿠버네티스 클러스터 내의 트래픽 통제를 위해 서비스 메시라는 아키텍쳐를 도입할 수 있습니다. 이때 Istio를 많이 사용하고, Istio의 mTLS 기능을 통해 보안성을 높힐 수 있습니다. Istio에는 클라이언트와 서버간의 인가 정책을 설정하는 기능들도 존재하지만 이번 포스팅에서는 mTLS에만 집중하도록 하겠습니다. 

## mTLS

mTLS는 기존의 TLS 방식과 다르게 네트워크 연결의 양쪽 끝에 있는 양 당사자가 올바른 개인 키를 가지고 있는지 확인하여 그들이 주장하는 당사자인지 확인하는 방법이다. 기존의 TLS는 서버 인증서를 통해 서버의 신뢰성만 확인하면 되는 방식이었지만, mTLS의 경우 클라이언트 인증서를 가지고 클라이언트의 신뢰성까지 보장해야 하는 방식이다.

그렇다면 TLS가 사용되는 모든 영역에서 mTLS를 사용하면 안전성을 강화할 수 있는게 아닐까? 하지만 일반 웹 브라우저 환경에서 mTLS를 사용하는건 부적합하다고 할 수 있다. 서비스를 사용하는 사용자는 수만명이 될 수 있는데, mTLS를 적용하기 위해서는 모든 사용자가 자신을 인증하기 위한 개별적인 인증서를 가지고 있어야 하기 때문이다. 따라서 mTLS는 server to server 환경에서 신뢰성을 보장하는데 사용하기 적합하다고 생각한다. 

## mTLS의 동작 방식

mTLS는 기존의 TLS 동작 플로우에 비해 몇단계가 추가된다.

<img width="671" alt="스크린샷 2024-06-17 오후 11 42 54" src="https://github.com/jemlog/tech-study/assets/82302520/dba91a32-2f90-4011-a452-44dc205e8906">

1. 클라이언트가 서버에 연결한다
2. 서버가 TLS 인증서를 제시한다
3. 클라이언트가 서버의 인증서를 확인한다
4. 클라이언트가 TLS 인증서를 제시한다
5. 서버가 클라이언트의 인증서를 확인한다
6. 서버가 액세스 권한을 부여한다
7. 클라이언트와 서버가 암호화된 TLS 연결을 통해 정보를 교환한다

## Istio의 Envoy 프록시 기반의 mTLS

그렇다면 Istio에서는 mTLS를 어떻게 적용하는지 살펴보자. Istio는 파드에 사이드카 형태로 Envoy 프록시를 주입한다. 이 Envoy 프록시를 통해 Istio는 트래픽 통제, 인증/인가, 모니터링/로깅/트레이싱을 수행한다. Envoy 프록시가 실질적으로 mTLS 연결을 맺는 역할을 하는데, Envoy 프록시가 존재할때 클라이언트와 서버의 mTLS 통신 과정은 아래와 같다. 

<img width="381" alt="스크린샷 2024-06-18 오전 12 11 27" src="https://github.com/jemlog/tech-study/assets/82302520/5f64b403-e638-4fea-9424-fb765c9aa3b5">

- Istio는 클라이언트 서비스의 아웃바운드 트래픽을 로컬 사이드카 프록시에게 라우팅한다.
- 클라이언트의 Envoy 프록시는 서버의 Envoy 프록시와 mTLS 핸드쉐이크를 시작한다. 핸드쉐이크를 수행하는 동안 클라이언트 Envoy 프록시는 서버측 인증서에 내장된 서비스 어카운트가 타겟 서비스를 실행할 권한이 있는지 검증하기 위해 Secure Naming 체크를 수행한다.
- 클라이언트 Envoy 프록시와 서버 Envoy 프록시는 mTLS 커넥션을 맺는다. 그리고 Istio는 클라이언트 Envoy 프록시의 트래픽을 서버로 포워딩한다.
- 클라이언트 Envoy 프록시는 요청을 인증한다. 만약 인증에 성공하면 로컬 TCP 커넥션을 통해 백엔드 서비스로 트래픽을 포워딩한다.

참고로 Istio는 최소 TLS 버전으로 **TLS v1.2**를 사용해야 한다.

## Istio의 mTLS 인증서 설정 방식

위에서 살펴봤듯이 mTLS를 위해서는 클라이언트와 서버가 모두 mTLS 인증서를 가지고 있어야 한다. 만약 클라이언트와 서버가 각각 하나씩이라면 직접 양쪽에 인증서를 설치하면 된다. 하지만 MSA 환경에서는 수백 수천개의 파드가 동작할 수 있다. 모드 파드에 인증서를 설치하는건 너무나 비효율적인 과정이기에 개발자 대신 인증서를 설치하는 과정이 필요할 것이다. Istio는 아래의 과정을 거쳐서
mTLS 인증서를 Envoy Proxy에 설치한다.


<img width="401" alt="스크린샷 2024-06-17 오후 8 44 37" src="https://github.com/jemlog/tech-study/assets/82302520/7daa76ee-e08a-4e34-96e4-5234a945f2ea">

1. istiod는 CSR(certificate signing requests)를 받기 위한 gRPC 서비스를 제공한다.
2. 파드 내에 존재하는 Istio Agent는 개인키와 CSR을 생성한다. 이후 CSR을 서명하기 위해 credential과 함께 istiod로 전달한다.
3. istiod에 들어있는 CA(Certificate Authority)는 Istio Agent로부터 받은 CSR 속의 credentials를 검증한다. 검증에 성공하면 인증서를 만들기 위해 CSR에 서명한다.
4. 워크로드가 시작되면 Envoy 프록시는 Istio Agent에게 인증서와 키를 요구한다.
5. Istio Agent는 istiod로부터 인증서를 받아서 개인키와 함께 Envoy 프록시에게 전달한다.
6. Istio Agent는 지속적으로 워크로드의 인증서가 만료되었는지 여부를 모니터링한다.

## Istio mTLS 정책

Istio에는 PeerAuthentication을 통해 mTLS 모드를 설정할 수 있다. 

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
spec:
  mtls:
    mode: PERMISSIVE # mode 설정 가능
```

Istio의 mTLS 모드에는 3가지가 존재한다.

- PERMISSIVE(default) : 워크로드가 mTLS 와 평문 트래픽을 모두 허용
- STRICT : 워크로드가 mTLS 트래픽만 허용
- DISABLE : mTLS가 비활성화. 사용 권장되지 않음

여기서 PERMISSIVE가 기본값으로 설정되어있는걸 알 수 있는데, 해당 모드에서는 클라이언트가 mTLS를 지원하지 않더라도 들어오는 트래픽을 모두 허용하겠다는 뜻이다.
PERMISSIVE 모드는 쿠버네티스 클러스터에 점진적으로 mTLS를 적용하는 단계에 적절하다고 생각한다. 하지만 완전한 제로 트러스트를 구축하기 위해서는 모든 클라이언트와 서버가 mTLS를 사용하도록
STRICT 모드를 사용해야 한다.

mTLS가 정상적으로 적용됐는지는 Kiali라는 Istio 모니터링 도구를 사용하면 편리하게 확인할 수 있다. 현재 foo와 bar이라는 네임스페이스를 만든 후, foo에는 Istio의 Envoy 프록시를 주입하고, bar에는 Envoy 프록시를 주입하지 않았다. 트래픽을 전송하면 Kiali 대시보드 상에 어떻게 나오는지 보자.

![340785824-1f632789-c05d-464f-a3e6-359205cab982](https://github.com/CMC11th-Melly/Melly_Server/assets/82302520/9789e15c-6b32-40f4-a11a-503189da71ef)

위의 사진을 보면 foo 네임스페이스 내부적인 연결에는 자물쇠가 걸려있고, foo에서 bar로 가는 트래픽에는 자물쇠가 없는걸 알 수 있다. 자물쇠 존재 여부로 mTLS가 적용됐는지를 파악할 수 있다. 
현재 Istio의 mTLS 정책은 `PERMISSIVE`이기에 평문 데이터를 요청으로 받는게 허용된다. 한번 직접 호출해보겠다. 

```shell
$ kubectl exec "$(kubectl get pod -l app=sleep -n bar -o jsonpath={.items..metadata.name})" -c sleep -n bar -- curl http://httpbin.foo:8000/ip -s -o /dev/null -w "bar에서 foo로 plainText 전송 : %{http_code}\n"

bar에서 foo로 plainText 전송 : 200
```

위의 결과에서 숫자는 HTTP 상태코드를 나타낸다. 요청을 보내면 정상적으로 200이 반환되는걸 알 수 있다. 하지만 세밀한 제로 트러스트를 구축하기 위해서는 `STRICT` 모드로 변경해야 한다.

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
spec:
  mtls:
    mode: STRICT
```
모드를 변경한 다음 똑같이 요청을 보내보자.

```shell
$ kubectl exec "$(kubectl get pod -l app=sleep -n bar -o jsonpath={.items..metadata.name})" -c sleep -n bar -- curl http://httpbin.foo:8000/ip -s -o /dev/null -w "bar에서 foo로 plainText 전송 : %{http_code}\n"

bar에서 foo로 plainText 전송 : 000
command terminated with exit code 56
```
이번에는 다른 결과가 나온걸 알 수 있다. foo 측에서는 상대방도 mTLS를 사용하는 경우에만 트래픽을 허용하도록 설정했기 때문에 평문 데이터를 트래픽이 차단된 것이다. Istio는 이런 방식으로 안전한 데이터 암호화를 지원한다.

## 결론

지금까지 Istio가 mTLS를 통해 쿠버네티스 클러스터 내에서 제로 트러스트를 구현하는 방법을 살펴봤다. mTLS 이외에도 클라이언트와 서버가 서로 API를 호출할 수 있는 대상을 제한하는 Authorization 정책을 통해 제로 트러스트를 구현할 수 있다. 다음에는 이 방법들에 대해서도 한번 정리를 해보고자 한다.

## Reference

- [https://istio.io/latest/docs/concepts/security/](https://istio.io/latest/docs/concepts/security/)
- [https://www.youtube.com/watch?v=4sJd6PIkP_s](https://www.youtube.com/watch?v=4sJd6PIkP_s)


[nodejs]: https://nodejs.org/
[starter]: https://github.com/cotes2020/chirpy-starter
[pages-workflow-src]: https://docs.github.com/en/pages/getting-started-with-github-pages/configuring-a-publishing-source-for-your-github-pages-site#publishing-with-a-custom-github-actions-workflow
[latest-tag]: https://github.com/cotes2020/jekyll-theme-chirpy/tags
