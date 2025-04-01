---
title: Istio network resilence
date: 2025-03-31 00:20:00
categories: [Kubernetes]
tags: [kubernetes]
pin: false
image:
  path: '/assets/img/istio_back.png'
---

## Timeout

Envoy proxy에서 타 서비스를 호출할때 타임아웃 지정할 수 있다. 이때 타임아웃이 너무 길면 상대편 서비스가 죽었을때 긴 레이턴시가 발생할 수 있고, 반대로 너무 짧으면 여러 서비스를 거쳐 응답이 돌아올때 미처 기다리기 못하고 부적절한 예외가 발생할 수 있다. 따라서 적절한 타임아웃을 찾는게 중요하다.

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - route:
    - destination:
        host: ratings
        subset: v1
    timeout: 10s
```

Istio는 VirtualService를 사용해서 Timeout을 지정할 수 있다. 모든 destination에 대해 일괄적으로 적용하는 것이 아니라 subset 별로 유연하게 타임아웃을 설정할 수 있다. 이때 만약 타임아웃을 설정하지 않으면 istio의 default 값은 timeout이 없는 것이다.

## Retry

Retry를 세팅하면 Envoy proxy가 보낸 초기 요청이 실패했을때 최대 몇번까지 재시도 할지 지정할 수 있다. 서비스는 과부화 되거나 일시적인 네트워크 장애가 발생해서 간헐적인 실패가 발생한다. 이때 바로 실패 처리를 하는 것 보다는 재시도를 할때 시스템의 가용성을 향상시킬 수 있다.

재시도 간의 backoff는 매번 다르고 구체적인건 Istio가 결정한다. 만약 retry를 설정하지 않으면 default로 총 2번 재시도한다. 

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - route:
    - destination:
        host: ratings
        subset: v1
    retries:
      attempts: 3 # 총 몇번 재시도 할 것인가
      perTryTimeout: 2s # 재시도마다 얼마의 timeout을 가질 것인가
```
Retry 또한 VirtualService를 사용해서 destination 마다 유연하게 적용 가능하다.

## Circuit Breaker

Istio는 host, subset 별로 개별적인 서킷 브레이커를 설정할 수 있다. 대표적으로는 최대 동시 커넥션 수, 최대 호출 실패 횟수 등을 지정할 수 있다.
Circuit Breaker를 사용하면 클라이언트가 요청 실패하는 호스트에게 계속 요청을 보내는 대신 Fail Fast 전략을 사용할 수 있다.

```yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  subsets:
  - name: v1
    labels:
      version: v1
    trafficPolicy:
      connectionPool:
        tcp:
          maxConnections: 100 # review service 워크로드들에 맺어질 수 있는 최대 동시 커넥션 수
```
Circuit Breaker는 Timeout, Retry와 다르게 **DestinationRule**에서 설정 가능하다.

## Fault Injection

Istio는 직접 장애를 주입할 수 있는 방법을 제공한다. 이를 통해 서비스의 장애 복구 능력을 검증할 수 있다. 주의할 점은 fault injection 설정은 같은 virtualService의 retry, timeout 설정과 함께 사용할 수 없다. Istio의 Fault Injection은 Application Layer에서 이뤄진다. 즉, HTTP error code 같은 최상위 레이어를 테스트 해볼 수 있다.

Istio Fault injection은 두가지 예외를 제공한다.
- Delay: 네트워크 지연이나 업스트림 서비스의 과부화를 테스트해볼 수 있다.
- Abort: 업스트림 서비스의 실패를 재현할 수 있다. Abort는 보통 HTTP Error나 TCP 커넥션 에러 형태로 진행된다.

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - fault:
      delay:
        percentage:
          value: 0.1 # 1000분의 1 확률로 5초의 delay 발생
        fixedDelay: 5s
    route:
    - destination:
        host: ratings
        subset: v1
```

## 주의점
Istio Failure recovey는 애플리케이션과 완전 분리되있다. 애플리케이션은 Envoy 사이드카가 실패를 핸들링하고 있다는 사실을 모른다. 즉, 애플리케이션에 실패 복구 정책을 구현했을때
충돌 발생 가능성을 예상해야 한다. 

만약 애플리케이션에서 2초의 타임아웃을 지정했지만 VirtualService에 3초의 타임아웃과 1번의 retry를 지정했다면, 애플리케이션이 이미 timeout이 난 뒤에는 Envoy의 타임아웃이나 재시도는 의미가 없다. 애플리케이션은 Istio에서 발생한 실패나 에러에 대한 적절한 Fallback을 핸들링해야 한다. 만약 로드밸런싱 풀의 모든 워크로드가 죽었다면 Envoy는 503을 반환한다. 이때 503 반환의 주체는 클라이언트 Envoy Proxy이다. 애플리케이션은 503에 대한 Fallback 로직을 만들어놔야 한다.

[nodejs]: https://nodejs.org/
[starter]: https://github.com/cotes2020/chirpy-starter
[pages-workflow-src]: https://docs.github.com/en/pages/getting-started-with-github-pages/configuring-a-publishing-source-for-your-github-pages-site#publishing-with-a-custom-github-actions-workflow
[latest-tag]: https://github.com/cotes2020/jekyll-theme-chirpy/tags
