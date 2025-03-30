---
title: 쿠버네티스 클러스터 내부의 Service 동작 분석
date: 2024-06-15 00:20:00
categories: [Kubernetes]
tags: [kubernetes, network]
pin: false
image:
  path: '/assets/img/k8s-image.png'
---

쿠버네티스를 학습하면서 네트워크에 대한 부분이 어렵게 다가왔습니다. 특히 Spring Cloud 기반의 MSA 어플리케이션을 쿠버네티스 환경에 배포할때는 eureka service discovery의 역할을 쿠버네티스의 Service가 대신해줄 수 있다는걸 알게되면서 이 컴포넌트에 대한 깊은 이해가 필요하다고 생각했습니다.

## Service 오브젝트 사용 이유

먼저 Service라는 오브젝트를 왜 사용해야 하는지 생각해보자. 만약 클라이언트 파드와 서버 파드가 있을때 서버 파드의 IP 주소를 알고 있다면 우리는 직접 통신을 연결할 수 있다. 만약 같은 노드 내에 두 파드가 존재한다면 cbr0라는 브릿지 네트워크를 통해 바로 통신을 할 수 있다. 다른 노드에 위치한다고 해도 노드 간 통신에 사용되는 라우팅 테이블에 정보가 들어있기 때문에 통신이 가능하다. 

하지만 컨테이너 환경은 파드의 생성과 삭제가 빈번하게 일어난다. 파드는 그때마다 IP가 변경되기 때문에 특정 IP 하나를 대상으로 통신을 연결한다는건 불가능에 가깝다고 할 수 있다. 또한 여러개의 서버 파드가 실행되고 있다면 파드 간의 로드 밸런싱도 해줘야 할 것이다.

위의 문제를 가장 쉽게 해결할 수 있는 방법은 서버 파드 앞단에 Reverse Proxy를 배치하는 것이다. 사실 컨테이너 환경이 아니더라도 우리는 분산 서버 구축을 위해 Nginx나 AWS의 ALB 같은 서비스를 사용해왔다. 쿠버네티스에도 이런 Reverse Proxy의 역할을 수행하는 컴포넌트가 존재하는데, 이게 Service 오브젝트다.

Service는 아래의 yaml 파일을 통해 생성할 수 있다.
```yaml
apiVersion: v1
kind: Service
metadata:
  name: testapp
spec:
  ports:
  - port: 80 # Service의 포트 설정
    targetPort: 80 # 파드의 어떤 포트와 연결할 것인지 설정
  selector:
    app: testapp # 해당 label을 가진 파드들과 연결된다
```

Service가 생성될때 쿠버네티스의 api-server는 어떤 파드가 Service와 연결되는지 찾는다. 이때 Service의 label selector와 일치하는 파드를 탐색한다. 파드들을 찾고나면 api-server는 endpoint를 생성한다. endpoint는 연결되는 파드의 IP를 나타내는데, Service는 생성된 endpoint들과 매핑된다. testapp이라는 이름을 가진 파드에 할당된 endpoint들을 검색해보면 아래의 2개의 IP가 조회되는걸 알 수 있다.

```shell
$ kubectl get pods --selector=testapp -o jsonpath='{.items[*].status.podIP}'
10.244.0.64 10.244.0.63 # Service에 연결된 2개의 파드 IP                                                                                                                            
```
이제 생성된 Service를 조회해보자.
```shell
$ kubectl get service testapp
NAME      TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
testapp   ClusterIP   10.99.74.155   <none>        80/TCP    9h
```
ClusterIP라는 Type으로 생성됐고, `10.99.74.155`라는 IP가 할당됐다. 또한 80 Port가 할당되있다. 여기서 ClusterIP라는 타입으로 서비스가 생성되면
해당 IP로 클러스터 내에서 모두 조회가 가능하다는 의미이다. 이외에는 NodePort와 LoadBalancer 타입이 있는데, 이후에 자세히 알아보는 것으로 하자.

Service로 요청을 보낼때는 직접 ClusterIP를 호출할 수 있지만, Service의 이름을 사용해서 호출할 수도 있다. 이게 가능한 이유는 쿠버네티스 내에는 Service의 이름과 IP가 매핑된 CoreDNS가 존재하기 때문이다. CoreDNS는 kube-system 네임스페이스에서 실행되고 있는 Pod 중 하나이고 네임서버로 사용된다. 

```shell
$ kubectl get po -n kube-system
NAME                               READY   STATUS    RESTARTS      AGE
coredns-7db6d8ff4d-2wctg           1/1     Running   2 (41h ago)   2d11h
...
```
이렇게 kube-system 네임스페이스 내에 coredns 파드가 동작하고 있는걸 확인할 수 있다.

### 파드 IP와 대역이 다른 ClusterIP

여기서 이상한 부분이 하나 있다. ClusterIP와 endpoint의 IP 대역이 서로 다르다. 만약 클라이언트가 ClusterIP를 사용해서 서버를 호출하면 현재 네트워크 대역에서는 ClusterIP가 어디에 위치하는지 라우팅 정보가 등록되지 않았기 때문에 찾을 방법이 없다는 것이다. 하지만 실제로는 ClusterIP를 통해서도 서버 파드로 요청을 정상적으로 보낼 수 있다. 이게 어떻게 가능한 것일까? 사실 이 부분이 Service 오브젝트의 동작을 이해하는데 핵심인 부분이라 생각한다.

만약 Client Pod에서 다른 노드의 Server Pod를 호출하는 과정을 생각해보자. 
1. Pod가 Service 이름으로 호출하면 CoreeDNS에서 이름에 매핑된 ClusterIP를 반환한다
2. Pod의 veth에서는 해당 ClusterIP 정보를 알지 못하기 때문에 쿠버네티스의 cbr0로 트래픽이 올라간다
3. cbr0은 단순히 트래픽 전달의 역할을 하기 때문에 노드의 네트워크 인터페이스인 eth0로 트래픽이 올라간다
4. 이 부분에서 **ClusterIP가 요청하고자 하는 서버 파드의 IP로 직접 변환되서 노드 밖으로 나간다**

여기서 ClusterIP를 파드 IP로 변환해주는 작업이 핵심인데, 이 역할을 하는게 Kube-Proxy이다.

## Kube-Proxy

kube-proxy는 클러스터 내의 모든 노드에 존재하는 컴포넌트다. Kube-Proxy는 목적지 IP 변환(DNAT) 작업에 노드의 iptables와 netfilter를 사용한다. 어떤 과정을 거쳐서 Kube-Proxy가 Service 정보를 iptables에 등록하는지 살펴보자.

위에서 Service가 생성되고 endpoint와 매핑되는 과정을 살펴봤다. 해당 작업이 완료된 후에 쿠버네티스의 api-server는 Kube-Proxy에게
서비스 생성에 대한 정보를 전파한다. Kube-Proxy는 이 변경 사항들을 노드의 iptables에 NAT Rule로 반영한다. 이 NAT Rule을 통해 Service IP를 파드 IP로 DNAT한다. Service와 그에 연결된
파드는 언제나 동적으로 변할 수 있다. 그때마다 Kube-Proxy는 api-server로부터 변경사항을 전달받아 실시간으로 iptables의 Rule을 변경한다.

만약 파드의 IP는 설정했는데, 실제 파드가 Unhealthy 상태가 된다면 어떻게 될까? 서비스가 요청을 로드밸런싱해도 파드가 죽었기 때문에 트래픽을 
처리하지 못할 것이다. Kube-Proxy에게 파드들의 상태를 알려주는 작업이 필요한 것이다. 이 역할을 Kubelet이 수행한다. Kubelet은 쿠버네티스의 Node Agent로써 Endpoint 파드들의 Health를 지속적으로 모니터링하다가 UnHealthy 파드가 생기면 api-server에 이 사실을 알린다. api-server는 다시 이 정보를 Kube-Proxy에게 전파하고, Kube-Proxy는 업데이트 된 정보를 기반으로 iptables를 수정해서 로드밸런싱 대상에서 unhealthy 파드를 제거한다.

Kube-Proxy의 mode에는 `userspace`, `iptables`, `ipvs`가 있다.

### Userspace Mode

userspace 모드에서는 netfilter를 통해 들어오는 service 트래픽을 kube-Proxy로 라우팅한다. kube-proxy가 요청들어온 트래픽을 실제 server pod의 ip:port로 변환해서 eth0한테 전달한다. Kube-proxy가 직접 Proxy로써 NAT 역할을 할때는 kube-proxy 자체가 단일 장애 지점이 될 수 있었기 때문에 안정성이 떨어진다.

### Iptables Mode

linux 커널의 netfilter가 실질적으로 service IP를 발견하고 그것을 실제 매핑된 Pod로 전달하는 기능을 수행하고, kube-proxy는 이 netfilter의 rule을 수정하는 역할을 한다. iptables mode에서는 netfilter가 실질적인 proxy 역할을 수행하고, netfilter는 linux kernel의 기능이기 때문에 서버가 살아있는 한은 동작을 보장받을 수 있다. kube-proxy는 마스터 api-server의 정보를 수신해서 클러스터 변화 감지한다. 지속적으로 iptables를 업데이트해서 netfilter 규칙 최신화한다. 새로운 service 생성되면 kube-proxy는 알림을 받고 그에 맞는 규칙 생성한다.

iptables은 기본적으로 방화벽을 위해 탄생한 기능이다. 쿠버네티스 환경에서 수많은 Service와 파드가 iptables 모드는 table을 탐색할때 순차 탐색을 사용한다. 이 순차 알고리즘은 Service와 Endpoint 개수가 늘어나면 O(n)의 시간복잡도로 인해 Lookup 횟수가 증가한다. 또한 iptables는 별도의 로드밸런싱 알고리즘을 제공하지 않고 random 방식만을 사용한다.

### IPVS Mode

ipvs는 로드밸런싱을 위해 특별히 디자인된 리눅스 기능이다. ipvs 모드에서는 Kube-Proxy가 iptables 대신 ipvs에 Rule을 삽입한다. ipvssms 내부적으로 해시를 사용해서 시간복잡도 O(1)의 최적화된 Lookup 알고리즘을 사용한다. 따라서 내부적으로 Rule이 많아져도 항상 상수 시간의 성능을 보여준다는 장점이 있다. 또한 ipvs는 Round Robin, Least Connection 등의 다양한 로드밸런싱 알고리즘을 제공한다. 

이런 많은 장점이 존재하지만, ipvs는 모든 리눅스 시스템에 아직 존재하지는 않는다. 반면 iptables는 모든 리눅스 시스템의 핵심 특징이고, Service 개수가 엄청나게 많지 않다면 iptables 모드도 좋은 성능을 보여준다.

이번 포스팅에서는 iptables Mode에 집중해서 알아보도록 하자.

## iptables & netfilter

그 전에 iptables와 netfilter가 어떤 장치인지 간단히 알아보자. 

iptables는 Kernel의 패킷 필터링 프레임워크인 netfilter와 함께 사용되는 방화벽이다. iptables는 Linux Kernel 네트워킹 스택의 패킷 필터링 hook들과 상호 소통하며 동작한다.
이 Kernel hook들은 netfilter 프레임워크로 알려져있다.
네트워킹 레이어를 지나가는 모든 패킷들은 이 hook들을 트리거한다, 그래서 프로그램들이 트래픽과 중요한 순간에 상호 소통하도록 만든다.

netfilter에 어떤 hook들이 있는지 알아보자.

- `NF_IP_PRE_ROUTING` : 이 훅은 어떤 income 트래픽이 네트워크 스택에 들어오자마자 트리거된다. 이 훅은 패킷이 어디로 갈지 경로를 결정하기 전에 실행된다.
- `NF_IP_LOCAL_IN` : 이 훅은 income 패킷이 local로 들어도로록 라우딩 된 다음에 트리거된다.
- `NF_PI_FORWARD` : 이 훅은 income 패킷이 다른 호스트로 빠져나가도록 결정된 뒤 트리거된다.
- `NF_IP_LOCAL_OUT` : 이 훅은 로컬에서 만들어진 outbound 트래픽이 네트워크 스택을 hit 하자마자 발생
- `NF_IP_POST_ROUTING` : 이 훅은 outgoing이나 forward 트래픽에 의햇 트리거된다. 밖으로 나간다고 결정되자마자.

이 훅을 사용하는 커널 모듈들은 훅이 호출되면 어떤 순서로 동작할지를 결정하는 Order를 제공해야 한다. 각자 순서되로 모듈이 호출되고 나면 netfilter에게 패킷이
어떤 동작을 수행해야 하는지에 대한 decision을 반환한다.

iptables에 저장되는 Rule은 chain에 소속된다. chain들은 netfilter hook에 연계되서 수행된다. iptables에는 여러 테이블이 존재하는데 하나의 테이블에는 여러 chain이 존재할 수 있다. 만약 chain에 등록된 Rule이 match되면 target이 됐다고 한다. 이때 3가지 동작 중 하나가 수행된다.

- `Terminating` : 해당 chain에서의 평가를 종료하고 netfilter hook에게 통제권을 돌려준다. 결과에 따라서 패킷을 drop하거나 다음 stage로 packet을 허용한다.
- `Non-Terminating` : 중간에 종료되지 않고 끝까지 수행된 후 netfilter에게 결과를 돌려준다.
- `Jump Target` : 매칭되면 추가적인 처리를 위해 다른 chain으로 이동한다. iptables는 관리자가 사용자 정의 chain을 만들 수 있도록 하는데, 오직 jump target을 통해서만 도달 가능하다. 

## iptables를 사용한 Service 동작 원리

이제 Service에서 iptables가 어떻게 사용되는지 분석해보자. iptables에 여러 chain이 등록되겠지만, 여기서 중요한 chain은 `KUBE-SERVICE`, `KUBE-SVC-*` 그리고 `KUBE-SEP-*`이다.

- `KUBE-SERVICES` : Service 패킷의 시작점이다. 목적지 IP:Port와 대응되는 `KUBE-SVC-*` chain으로 전달한다.
- `KUBE-SVC-*` : 해당 chain은 마치 로드밸런서처럼 동작하여 `KUBE-SEP-*` chain으로 패킷을 전달한다.
- `KUBE-SEP-*` : 이 chain은 Service의 IP:Port를 Pod의 IP:Port로 변환하는 DNAT를 수행한다.

쿠버네티스는 Packet Filtering과 NAT를 수행하기 위해 iptables에 `KUBE-SERVICES`라는 사용자 정의 chain을 생성한다. 이 chain을 추가하면 PREROUTING과 OUTPUT chain을 통과하는 모든 트래픽이 `KUBE-SERVICES`로 보내진다.

```shell
$ iptables -t nat -L PREROUTING
Chain PREROUTING (policy ACCEPT)
target     prot opt source               destination         
KUBE-SERVICES  all  --  anywhere             anywhere             /* kubernetes service portals */
DOCKER_OUTPUT  all  --  anywhere             host.minikube.internal 
DOCKER     all  --  anywhere             anywhere             ADDRTYPE match dst-type LOCAL
```

처음 패킷이 네트워크 스택에 들어왔을때 트리거되는 PREROUTING chain을 검색하면 연관된 chain들이 조회된다. 현재 도커 기반의 minikube 환경에서 진행하고 있기 때문에 도커와 관련된 사용자 정의 chain도 보인다. 여기서 주목해야할 chain은 `KUBE-SERVICES`이다.

```shell
$ iptables -t nat -L KUBE-SERVICES
Chain KUBE-SERVICES (2 references)
target     prot opt source               destination         
...
KUBE-SVC-V2OKYYMBY3REGZOG  tcp  --  anywhere             10.99.74.155         /* default/testapp cluster IP */ tcp dpt:http
...
```
`KUBE-SERVICES` chain은 `KUBE-SVC-*` chain과 연결되있다. 해당 chain은 로드밸런서의 역할을 하며 연결된 `KUBE-SEP-*` chain으로 트래픽을 전달한다.

```shell
$ iptables -t nat -L KUBE-SVC-V2OKYYMBY3REGZOG 
Chain KUBE-SVC-V2OKYYMBY3REGZOG (2 references)
target     prot opt source               destination         
KUBE-MARK-MASQ  tcp  -- !10.244.0.0/16        10.99.74.155         /* default/testapp cluster IP */ tcp dpt:http
KUBE-SEP-MUU44H3RRYQLTQ3I  all  --  anywhere             anywhere             /* default/testapp -> 10.244.0.63:80 */ statistic mode random probability 0.33333333349
KUBE-SEP-UGFU6IF6MA43DDMH  all  --  anywhere             anywhere             /* default/testapp -> 10.244.0.64:80 */ statistic mode random probability 0.50000000000
```
`KUBE-SEP-*` chain의 가장 하단에 DNAT 항목이 있고 주석에는 `10.244.0.63:80`으로 트래픽을 전달한다는 내용이 있다. 우리가 찾고자 했던 ClusterIP를 파드 IP로 변경시키는 부분이 여기서 수행이 되는 것이다. 

```shell
$ iptables -t nat -L KUBE-SEP-MUU44H3RRYQLTQ3I
Chain KUBE-SEP-MUU44H3RRYQLTQ3I (1 references)
target     prot opt source               destination         
KUBE-MARK-MASQ  all  --  10.244.0.63          anywhere             /* default/testapp */
DNAT       tcp  --  anywhere             anywhere             /* default/testapp */ tcp to:10.244.0.63:80
```
파드의 endpoint를 조회하면 `10.244.0.63:80`이 존재하는걸 알 수 있다.

```shell
$ kubectl get ep testapp
NAME      ENDPOINTS                       AGE
testapp   10.244.0.63:80,10.244.0.64:80   15m
```

위의 절차에 따라 NAT된 파드 IP가 라우팅 테이블의 지시에 따라 목적지 파드로 올바르게 향하게 되는 것이다.

## 결론
이번 포스팅에서는 쿠버네티스 클러스터 내부에서 Service 오브젝트가 어떻게 동작하는지 알아봤다. Service에 대해 추상적인 느낌이 있었는데, 이번 기회를 통해서
자세히 알 수 있었던 기회였다. 다음 포스팅에서는 쿠버네티스 클러스터 외부에서 Service에 접근하고자 할때 사용할 수 있는 방법인 NodePort와 LoadBalancer 그리고 Ingress에 대해 정리해보고자 한다.

## Reference
- [https://coffeewhale.com/k8s/network/2019/05/11/k8s-network-02/](https://coffeewhale.com/k8s/network/2019/05/11/k8s-network-02/)
- [https://coffeewhale.com/packet-network3](https://coffeewhale.com/packet-network3)
- [https://www.digitalocean.com/community/tutorials/a-deep-dive-into-iptables-and-netfilter-architecture](https://www.digitalocean.com/community/tutorials/a-deep-dive-into-iptables-and-netfilter-architecture)
- [https://medium.com/@amroessameldin/kube-proxy-what-is-it-and-how-it-works-6def85d9bc8f](https://medium.com/@amroessameldin/kube-proxy-what-is-it-and-how-it-works-6def85d9bc8f)
- [https://www.digitalocean.com/community/tutorials/a-deep-dive-into-iptables-and-netfilter-architecture](https://www.digitalocean.com/community/tutorials/a-deep-dive-into-iptables-and-netfilter-architecture)



[nodejs]: https://nodejs.org/
[starter]: https://github.com/cotes2020/chirpy-starter
[pages-workflow-src]: https://docs.github.com/en/pages/getting-started-with-github-pages/configuring-a-publishing-source-for-your-github-pages-site#publishing-with-a-custom-github-actions-workflow
[latest-tag]: https://github.com/cotes2020/jekyll-theme-chirpy/tags
