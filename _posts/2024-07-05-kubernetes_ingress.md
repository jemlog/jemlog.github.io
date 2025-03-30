---
title: 로드밸런서와 Ingress를 사용한 쿠버네티스 클러스터 외부 통신
date: 2024-07-05 00:20:00
categories: [Kubernetes]
tags: [kubernetes]
pin: false
image:
  path: '/assets/img/k8s-image.png'
---

[이전 포스팅](https://jemlog.github.io/posts/%EC%BF%A0%EB%B2%84%EB%84%A4%ED%8B%B0%EC%8A%A4_%EC%84%9C%EB%B9%84%EC%8A%A4_%EB%8F%99%EC%9E%91_%EB%B6%84%EC%84%9D/)에서는 쿠버네티스 클러스터 내부 통신을 담당하는 Service 오브젝트의 동작원리에 대해 깊게 살펴봤습니다. 
클라이언트가 서비스를 이용하기 위해서는 결국 클러스터 외부에서 접속을 시도해야 합니다. 이때 클러스터의 가장 앞단에서 외부 트래픽을 보안/수정 처리를 한 후 내부 Service로 라우팅하는 역할을 하는 컴포넌트가 존재하는데, 이를 **인그레스 컨트롤러**라고 합니다.

이번 포스팅에서는 인그레스와 관련된 요소들이 어떤게 있는지 살펴보고, 외부 트래픽이 어떤 흐름으로 클러스터의 내부 파드까지 전달이 되는지 분석해보겠습니다. 인그레스는 클러스터에 유입되는 모든 트래픽을 관리하는 만큼 인그레스 컨트롤러(Nginx) 자체가 제공하는 기능에 대한 깊은 이해가 필요합니다. 이 부분은 실무를 진행하면서
경험치를 쌓아나가야 한다고 생각하기에, 이번에는 쿠버네티스 오브젝트와 관련된 부분만 학습을 진행해보겠습니다.

## Ingress

우선 인그레스의 가장 기본이 되는 **Ingress**라는 오브젝트에 대해 알아보자. Ingress는 특정 Rule에 따라 외부 트래픽이 어떤 내부 Service와 매칭될지를 명시한 스펙이다.

<img width="300" alt="스크린샷 2024-07-04 오전 3 32 56" src="https://github.com/jemlog/tech-study/assets/82302520/875627d4-3c42-4806-b6da-b3b8e9e4a76b">

위 그림은 Ingress의 구조를 개략적으로 나타낸 것이다. `host` 필드에는 들어오는 요청의 도메인을 설정할 수 있고, `name` 필드에는 요청이 어떤 내부 Service와 매칭되는지를 명시한다. 아래는 자세한 Ingress Spec이다.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress 
spec:
  ingressClassName: nginx # 연결하고자 하는 ingressClass의 이름 명시
  rules:
  - host: test.first.com # 해당 도메인으로 요청이 들어오면 동작한다
    http:
      paths:
      - path: / # 모든 경로에 대해 적용
        pathType: Prefix
        backend:
          service:
            name: first_svc # 연결하고자 하는 Service의 이름 명시
            port:
              number: 80 # 서비스의 Port 명시

  - host: test.second.com
    http:
      paths:
      - path: "/"
        pathType: Prefix
        backend:
          service:
            name: second_svc
            port:
              number: 80

  tls:
    - hosts:
        - test.first.com # 해당 도메인으로 요청이 들어오면 tls 적용
      secretName: test-tls # tls 인증서가 저장된 Secret 오브젝트
```

중요한 내용들 위주로 살펴보자. IngressClassName 필드를 사용해서 IngressClass와 연결할 수 있다. 이때 IngressClass는 Ingress Controller와 Ingress를 연결하는 매개체 역할을 한다.
IngressClass에 대한 자세한 내용은 뒤에서 알아보겠다.

Ingress는 Virtual Name 기반으로 여러개의 도메인에 대한 분기도 가능하다.
위의 Spec에서는 test.first.com과 test.second.com으로 도메인을 분기했다.
만약 path 조건까지 매칭되면 설정한 Service 이름과 포트가 반환된다.

마지막으로 tls 필드를 통해서 Ingress에서 TLS Termination을 수행하도록 만들 수 있다. 이때 TLS 인증서는 Secret 오브젝트에 미리 등록해놔야 한다.

Ingress에 대한 더 자세한 내용은 [공식문서](https://kubernetes.io/docs/concepts/services-networking/ingress/)를 참고해보자.


## IngressClass

지금까지 Ingress에 대해 알아봤다. 근데 Ingress만 있으면 외부와 통신을 수행할 수 있을까?

Ingress는 사실 Rule을 표현한 것에 불과하기 때문에 실제 트래픽을 받고 라우팅을 하는 추가적인 컴포넌트가 필요하다. 이 역할을 하는게 **Ingress Controller**다.
Ingress와 Ingress Controller를 연결해주는 중간 매개체가 필요한데, 이 역할을 수행하는게 `IngressClass`이다.

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: nginx # Ingress는 해당 이름을 사용해서 Ingress Controller와 매핑된다
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true" # 해당 IngressClass를 default로 사용한다.
spec:
  controller: k8s.io/ingress-nginx # ingress-nginx라는 ingress controller를 연결한다
```

annotations를 보면 **is-default-class : true**가 설정되어 있다. Ingress는 ingressClassName 필드를 통해 특정 ingressClass에 연결할 수 있는데, 만약 지정하지 않은 경우에는 자동으로 default ingressClass에 매핑된다.
단, default ingressClass는 전체에서 딱 하나만 설정 가능하다.

## Ingress Controller

이제 직접 외부 트래픽을 받는 주체인 인그레스 컨트롤러에 대해 알아보도록 하자. 인그레스 컨트롤러는 직접 설치를 해야하고, 대표적으로 Nginx를 인그레스 컨트롤러로 사용할 수 있다. 

인그레스 컨트롤러를 처음 접하면, 트래픽이 Ingress Controller -> Ingress -> Service -> Pod 순서로 들어간다고 생각할 수 있다. 하지만 인그레스 컨트롤러는 직접 Pod로 트래픽을 전송한다. 이게 어떻게 가능한걸까? 아래 그림을 순서대로 살펴보자.

<img width="876" alt="스크린샷 2024-07-04 오전 5 04 05" src="https://github.com/jemlog/tech-study/assets/82302520/d9f926b1-e8a7-42f9-b9f3-fc48baec4441">

1. 인그레스 컨트롤러는 kube-apiserver로부터 모든 네임스페이스에 존재하는 Ingress 정보를 모두 조회한다.
2. 획득한 Ingress에서 Service 이름을 추출한다.
3. Service 이름과 매핑된 EndpointSlice들을 조회한다.
4. EndpointSlice에 할당된 Pod IP들을 가져와서 직접 Pod로 트래픽을 전송한다.

결론적으로 트래픽은 인그레스 컨트롤러로 들어온 뒤 바로 Pod로 향한다. 따라서 원래 Pod 로드밸런싱을 담당하던 Service의 역할은 사라진다. 대신 인그레스 컨트롤러가 자체적으로 로드밸런싱을 수행한다.

Nginx 인그레스 컨트롤러는 아래와 같이 helm chart로 편리하게 설치할 수 있다. 
자세한 내용은 [Nginx Ingress Controller Helm 저장소](https://artifacthub.io/packages/helm/ingress-nginx/ingress-nginx)를 참고하자.

```shell
$ helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
$ helm repo update

$ helm install ingress-nginx ingress-nginx/ingress-nginx -n ingress-nginx --create-namespace
```

위의 과정을 거치면 `ingress-nginx` 네임스페이스에 Nginx 인그레스 컨트롤러 파드가 하나 생성된다. 이제 위에서 살펴본 **IngressClass**와 **Ingress**를 연동하면 정상적으로 외부 트래픽을 내부 파드로 전송할 수 있다.

## Istio Ingress Gateway

지금까지는 쿠버네티스 자체적으로 제공하는 Ingress 기능을 사용하는 인그레스 컨트롤러를 살펴봤다.
만약 쿠버네티스 클러스터에 Istio 서비스 메시를 도입한 상태라면 Nginx 인그레스 컨트롤러 대신 Istio에서 기본으로 제공해주는 Istio 인그레스 게이트웨이를 사용할 수 있다.

<img width="773" alt="스크린샷 2024-07-04 오후 12 58 01" src="https://github.com/jemlog/tech-study/assets/82302520/11641e45-3af9-4442-aa20-d19280d7c0f1">

외부 트래픽을 서비스 메시 내의 Service로 전달하기 위해서는 크게 **Gateway**와 **VirtualService**가 필요하다.

Gateway는 앞에서 살펴봤던 Ingress 오브젝트와 비슷한 역할을 수행한다고 할 수 있다. Istio Ingress gateway를 연결하고 어떤 호스트와 포트를 수신할지를 결정한다. 

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: jeminapp-gateway
  namespace: default
spec:
  selector:
    istio: ingress
  servers:
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
        - "jeminapp.com"
    - port:
        number: 443
        name: https
        protocol: HTTPS
      tls:
        mode: SIMPLE # enables HTTPS on this port
        credentialName: jeminapp-com-credential # fetches certs from Kubernetes secret
      hosts:
        - "jeminapp.com"
```

VirtualService는 Gateway를 통해 들어온 트래픽을 실제 Service 오브젝트로 라우팅하는 역할을 수행한다. **spec**의 **hosts**와 **gateways** 필드를 통해
어떤 gateway를 통해 어떤 domain이 들어오면 동작할지 판별한다. 아래의 예시에서는 jeminapp.com의 모든 경로로 들어오는 트래픽을 myapp이라는 Service로 라우팅한다. 

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: jeminapp-virtualservice
  namespace: default
spec:
  hosts:
    - "jeminapp.com"
  gateways:
    - default/jeminapp-gateway # namespace/gateway
  http:
    - match:
        - uri:
            prefix: / # all traffic
      route:
        - destination:
            host: myapp # kubernetes service name
            port:
              number: 80 # service port
```

## MetalLB를 사용한 로드밸런싱

Nginx 인그레스 컨트롤러와 Istio 인그레스 게이트웨이는 결국 하나의 파드로 동작한다. 따라서 외부와 통신하기 위해서 Service가 연결된다. 이때 Service를 어떤 타입으로 설정해야 할지 선택해야 한다.
Service가 외부 통신을 하도록 해주는 타입은 크게 **NodePort**와 **LoadBalancer**가 있다. 

NodePort 타입의 경우 특정 포트를 외부에 직접 노출하게 되고 들어오는 트래픽을 로드밸런싱 할 수 없다는 단점이 있다. 
반면 LoadBalancer 타입을 설정하면 NodePort가 생성됨과 동시에 외부의 로드밸런서에 연결된다. 
클라이언트는 로드밸런서의 공개된 포트를 통해서만 접근 가능하기에 내부의 NodePort를 숨길 수 있다. 
또한 로드밸런서가 각각의 노드에 생성된 NodePort들로 트래픽을 로드밸런싱 해준다. 
따라서 일반적으로 인그레스 컨트롤러의 Service 타입은 LoadBalancer로 설정하고 앞단에 별도의 로드밸런서를 설치한다.

만약 AWS나 GCP 같은 클라우드 서비스를 사용한다면 자체 제공해주는 로드밸런서를 쓸 수 있다. 하지만 온프레미스 환경에서 LoadBalancer 타입을 사용하기 위해서는 별도의 베어메탈용 로드밸런서를 구축해야 한다. 여기서 대표적으로 MetalLB라는 로드밸런서를 사용할 수 있다.

### MetalLB 설치

MetalLB는 아래의 명령어를 통해 쿠버네티스에 설치 가능하다.

```shell
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.11.0/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.11.0/manifests/metallb.yaml
```

metalLB를 설치한 후, 아래의 ConfigMap을 통해 LoadBalancer의 External IP가 할당된 대역을 **addresses** 필드에 설정할 수 있다. 참고로 192.168.56.0/24 대역은 필자가 VM에 할당한 네트워크 대역이다.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.56.100-192.168.56.110  # 가상머신에 할당한 IP 대역폭 중 선택
```

ConfigMap 까지 등록한 뒤 인그레스 컨트롤러의 Service를 조회하면 EXTERNAL-IP가 할당된 걸 알 수 있다. 우리는 해당 IP를 사용해서 MetalLB에 접근하면 된다.
하지만 조금 더 실제 환경과 유사하게 구축하기 위해 hosts 파일을 변경해서 jeminapp.com이라는 도메인을 192.168.56.100에 매핑했다.

```shell
[root@k8s-master ~] # kubectl get svc -n ingress-nginx
NAME                                 TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.100.24.105   192.168.56.100   80:31080/TCP,443:31443/TCP   7m22s
ingress-nginx-controller-admission   ClusterIP      10.98.239.249   <none>           443/TCP                      7m22s
```

이제 해당 도메인의 health check 엔드포인트로 요청을 보내보자. 응답이 정상적으로 반환되는걸 확인할 수 있다.

```shell
curl -k https://jeminapp.com/health # Self Signed Certificate로 테스트 했기에 -k 옵션 붙여준다
$ UP
```

## 결론

이번 포스팅에서는 외부 트래픽이 로드밸런서를 통해 인그레스 컨트롤러로 들어오고 내부 Pod로 까지 이어지는 과정에 대해 살펴봤다.
인그레스가 클러스터의 가장 앞단에서 트래픽을 받는 만큼 실무에서 잘 사용하기 위해서는 Nginx나 Istio 자체에 대한 깊은 공부가 필요하다고 본다.
따라서 만약 쿠버네티스 환경에서 실무를 하게 된다면 해당 솔루션에 대한 자세한 학습 과정도
보충해나갈 계획이다.


[nodejs]: https://nodejs.org/
[starter]: https://github.com/cotes2020/chirpy-starter
[pages-workflow-src]: https://docs.github.com/en/pages/getting-started-with-github-pages/configuring-a-publishing-source-for-your-github-pages-site#publishing-with-a-custom-github-actions-workflow
[latest-tag]: https://github.com/cotes2020/jekyll-theme-chirpy/tags
