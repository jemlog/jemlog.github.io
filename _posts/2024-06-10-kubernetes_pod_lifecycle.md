---
title: 쿠버네티스 파드 라이프사이클과 리소스 할당 분석
author: jemlog
date: 2024-06-10 00:20:00
categories: [Kubernetes]
tags: [kubernetes]
pin: false
image:
  path: '/assets/img/k8s-image.png'
---

쿠버네티스를 잘 사용하기 위해서는 방대한 지식을 습득해야 합니다. 그 중 어플리케이션 개발자가 잘 알아야 하는 분야가 있을 것이고, 반대로 인프라 담당자가 잘 이해해야 하는 분야가 있을 것입니다. 저는 개발자로서 쿠버네티스 워크로드의 최소 단위이자, 우리가 만든 어플리케이션이 동작하는 환경인
**파드**에 대한 깊은 이해가 중요하다고 생각합니다. 이번 포스팅에서는 파드와 내부 컨테이너의 라이프사이클 및 리소스 관리에 대해 알아보도록 하겠습니다.

## 컨테이너의 라이프사이클

필자는 처음 쿠버네티스에 대해 학습할때 파드와 컨테이너가 일대일로 매핑되는 개념이라 생각했다. 하지만 파드는 한개 이상의 컨테이너로 구성된 단위라는걸 알게 됐다. 파드의 라이프사이클을
이해하기 위해서는 내부의 컨테이너의 라이프사이클에 대해 학습하는 과정이 선행되야 한다고 생각된다.

파드 생성 요청이 **api-server**에 들어오면 먼저 **kube-scheduler**가 파드를 할당할 노드를 찾는다. 이후 해당 노드의 **kubelet**이 컨테이너 런타임을 사용해서 컨테이너 생성을 시작한다.
컨테이너의 상태는 `Waiting`, `Running` 그리고 `Terminated`가 있는데 하나씩 알아보자.

- **Waiting** : 컨테이너 이미지 저장소로부터 이미지를 가져오거나 Secret 데이터를 적용하는 과정에 해당 상태가 나타난다. 이 상태에서 Pod를 조회하면 왜 컨테이너가 이 상태에 있는지에 대한 이유가 요약되서 표현된다.

- **Running** : 컨테이너가 문제 없이 동작하고 있는 상태이다.

- **Terminated** : 컨테이너가 종료된 상태다. 정상 종료 됐으면 completion 상태가 되고 예외가 발생해서 종료됐다면 failed 상태가 된다.



### 컨테이너 재시작 정책

쿠버네티스를 사용하는 명확한 이유 중 하나는 문제가 생긴 컨테이너를 자동으로 재시작하는 **Self Healing** 기능이라 생각한다. 
개발자는 재시작에 대한 여러가지 정책을 설정할 수 있는데, Pod Spec의 `restartPolicy` 필드를 통해 설정 가능하다.

- Always (default) : 컨테이너의 정상/예외 종료에 상관 없이 무조건 컨테이너 재시작
- OnFailure : 컨테이너가 예외적으로 종료됐을때만 컨테이너를 재시작
- Never : 컨테이너가 한번이라도 종료되면 재시작하지 않음

### 컨테이너 재시작 BackOff

컨테이너가 죽었을때 자동으로 재시작하는 기능은 관리자에게 매우 유용해보인다. 하지만 재시도를 무조건 빠르게 반복하는게 정답일까? 

여러개의 컨테이너가 짧은 주기로 지속적인 재시작을 하면 시스템 전체에 오버헤드가 발생한다. 
사실 쿠버네티스가 아니더라도 재시도 로직이 사용되는 대부분의 영역에서 재시도 주기에 대한 고민이 필요하다. 그리고 보통 재시도 주기를 지수적으로 증가시키는
**Expotential BackOff** 방식이 많이 사용된다.

쿠버네티스 또한 컨테이너 재시작 정책에서 Expotential Backoff를 지원한다. 
파드 재시작은 Kubelet이 담당하는데, 처음 재시작으로부터 `10s, 20s, 40s...` 순서로 BackOff 타이머를 증가시키고 최대 300s까지 수행한다.
이후 컨테이너가 정상 기동된 되서 10분동안 문제 없이 실행되면 컨테이너의 Restart Backoff 타이머는 초기화된다.

### 컨테이너 동작 실패

쿠버네티스는 컨테이너 실패를 restartPolicy를 통해 제어한다. 그렇다면 컨테이너 실패는 어떤 순서로 진행될까.

1. Initial crash : 쿠버네티스는 restartPolicy에 따라 즉시 재시작한다
2. Repeated crash : Initial crash 발생 후에 지수적 백오프 지연을 대기한 후 다시 한번 재시작한다.
3. CrashLoopBackOff : 지속적으로 재시작이 발생하는 상태를 나타낸다.
4. BackOff reset : 만약 컨테이너가 정상적으로 10분동안 동작하면 컨테이너의 backoff timer가 초기화된다.

아마 쿠버네티스를 운영하면서 가장 많이 만나게 될 컨테이너 상태 중 하나는 `CrashLoopBackOff`일 것이다. 이 CrashLoopBackOff는 보통 어떤 상황에서 발생할까?

- 컨테이너를 종료 시키는 어플리케이션 에러 발생
- 설정 파일이 없거나 잘못된 환경 변수가 주입
- 컨테이너가 실행되기 위한 충분한 메모리와 CPU를 가지지 못할때
- 컨테이너가 liveness Probe나 startup probe에서 Failure를 반환받을때

위의 상황이 발생하고 restartPolicy에 의해 재시도가 반복되면 CrashLoopBackOff 상태가 된다.

## 파드의 라이프사이클

지금까지 컨테이너의 동작에 대해 알아봤다. 지금부터는 컨테이너를 포함하는 파드의 라이프사이클에 대해 알아보자. 파드는 총 4가지 상태를 가지게 된다.

- **Pending** : 파드가 스케쥴 되기를 기다리는 시간과 컨테이너 이미지를 다운로드하는 시간을 포함한다.

- **Running** : 파드가 노드에 할당되고 파드 내의 모든 컨테이너가 생성된 상태. 적어도 하나의 컨테이너가 동작하고 있거나 시작하거나 재시작하는 과정에 있다.

- **Succeeded** : 파드 내의 모든 컨테이너가 정상 종료 되서 Restart되지 않는다.

- **Failed** : 파드 내의 모든 컨테이너가 종료 됐지만 적어도 하나의 컨테이너가 실패 상태로 끝난 상황이다.


## 파드의 Probe

쿠버네티스는 지속적으로 파드가 서비스 가능한 상태인지를 체크하고, 문제 발생 시 파드로 들어오는 트래픽을 차단하거나 Restart를 수행한다. 이때 Kubelet이 컨테이너에 대해 API 호출을 통한 헬스 체크를 수행하는데, 이 과정을 **Probe**라고 부른다. 
Probe에는 3가지 종류가 있다.

### Startup Probe

컨테이너 안의 어플리케이션이 시작됐는지를 체크하는 Probe 동작이다. 해당 Probe가 성공하기 전까지 Readiness와 Liveness Probe는 실행되지 않는다. 만약 Startup Probe가 실패하면, Kubelet이 컨테이너를 죽이고 RestartPolicy에 따라 재시도를 수행한다.

### Readiness Probe

컨테이너가 외부 요청에 응답을 정상적으로 할 수 있는지를 판단하는 Probe 동작이다. 만약 해당 Probe가 실패하면 Service에 연결된 파드의 EndPoint를 제거한다.

### Liveness Probe

컨테이너가 정상 동작하고 있는지를 판단하는 Probe 동작이다. 만약 실패하면 Kubelet이 컨테이너를 죽이고 RestartPolicy에 따라 재시도를 수행한다.

Deployment 오브젝트에서 다음과 같이 설정 가능하다.

```yaml
spec:
      containers:
        startupProbe:
          httpGet:
            path: "/health"
            port: 8080
          periodSeconds: 5 # 5초마다 한번씩 진행
          failureThreshold: 10 # 실패 횟수가 10번 넘어가면 최종 실패 처리
        readinessProbe:
          httpGet:
            path: "/health"
            port: 8080
          periodSeconds: 5
          failureThreshold: 5 
        livenessProbe:
          httpGet:
            path: "/health"
            port: 8080
          periodSeconds: 10
          failureThreshold: 10
```

### Readiness Probe와 LivenessProbe의 차이

Startup Probe는 컨테이너가 시작될때 정상적으로 초기화가 됐는지 확인한다는 목적이 있다. 
근데 Readiness Probe와 Liveness Probe의 차이는 조금 모호하다. 
둘 다 파드가 Running 하는 동안 주기적으로 헬스 체크를 수행한다는 공통점이 있다. 
얼핏 보면 컨테이너에 문제가 생겼는지는 둘 중 하나로만 체크해도 충분해보인다. 
그렇다면 굳이 두 가지를 나눠놓은 이유가 뭘까?

필자는 시스템 장애에 크게 두가지 종류가 있다고 본다. 하나는 **회복성을 가진 장애**이고, 나머지는 **자동으로 복구될 수 없는 치명적인 장애**다.
첫번째 장애의 경우에는 시스템에 순간적으로 트래픽이 몰려 부하가 발생하는 상황에서 가능할 것이다. 이때는 잠시 트래픽을 차단해주는 것 만으로도 장애 상황에서 벗어날 수 있다. 
어플리케이션에서 회복탄력성을 위해 적용하는 서킷 브레이커 패턴과 비슷한 원리라고 생각하면 된다.
이 경우에는 Readiness Probe를 사용해서 파드로부터 Service를 분리하는 방식으로 장애 상황에 대처가 가능하다.

하지만 트래픽을 차단한 후에도 스스로 회복하지 못하는 치명적인 장애라면 어떻게 해야 할까? 이때는 컨테이너를 재시작하는게 최선일 것이다.
이 경우는 Liveness Probe를 사용해서 장애 상황에 대처하면 된다.

그렇다면 Readiness와 Liveness를 어떻게 설정하는게 이상적일까? 
Liveness Probe가 실패했을때 컨테이너를 재시작하는 작업은 불필요한 오버헤드를 발생시킬 수 있다.
따라서 우선은 트래픽을 차단해서 어플리케이션이 회복할 시간을 벌어주는게 좋은 방법이라고 본다.
**Readiness Probe의 주기와 임계치를 Liveness Probe보다 짧게 설정하자.** 
만약 서비스를 분리한 상태에서도 어플리케이션이 회복되지 않는다면 그때 Liveness Probe를 통해 컨테이너 재시작을 수행하면 된다.

## 파드의 종료 과정

Spring 어플리케이션을 운영할때 Graceful한 종료 과정은 상당히 중요하다. 그래야 어플리케이션 종료 시 점유하고 있던 리소스들을 안전하게 해제해서 리소스 누수를 막을 수 있기 때문이다. 
파드는 우리의 어플리케이션이 돌아가는 환경이다. 따라서 파드 자체가 종료되는 과정도 Graceful 해야만 한다. 우리가 파드 종료 명령을 내린 후, 어떤 과정을 거쳐서 종료가 실행되는지 살펴보자.

<img width="967" alt="스크린샷 2024-07-15 오전 10 04 47" src="https://github.com/user-attachments/assets/d2688b8b-214e-4620-8551-263db802d2d9">

1. 클라이언트가 api-server에 파드 삭제 요청을 보낸다.
2. api-server가 kube-proxy와 kubelet에게 병렬적으로 요청을 보낸다. 이때 kube-proxy는 iptables에서 파드 IP와 Service의 연결을 끊는다.
3. kube-proxy는 요청을 받고 어플리케이션에 SIGTERM 신호를 보낸다. SIGTERM 신호를 받으면 어플리케이션은 내부적으로 Graceful한 종료 작업을 수행한다. 성공적으로 종료되면 컨테이너 상태는 Terminated로 바뀌고 파드 상태도 Succeeded로 변경된다.
4. 만약 종료 작업이 `terminationGracePeriodSeconds` (default 30초) 안에 끝나지 않으면 kubelet은 SIGKILL을 전달한다. 어플리케이션은 강제 종료 되고 파드는 Failed 상태가 된다.

terminationGracePeriodSeconds는 파드가 Graceful하게 종료되도록 기다려주는 시간이다. 해당 값은 어플리케이션 상황에 맞춰서 합리적으로 설정해야 한다. 

너무 짧으면 아직 리소스가 완전히 회수 되기 전에 강제 종료를 해서 리소스 누수가 발생할 수 있다. 반면 너무 길게 설정하면 종료 과정에서 문제가 발생한 
파드를 빠르게 회수하지 못해서 시스템 전체적으로 리소스가 낭비될 수 있다. 
따라서 어플리케이션의 여러 타임아웃 설정 값들을 반영해서 적절한 값을 선택해야 한다.

## 파드의 리소스 요청과 제한

지금까지는 파드와 컨테이너의 라이프사이클에 대해서 알아봤다. 여기서 애플리케이션 개발자가 파드와 관련해서 알아야하는게 하나 더 있다고 생각한다. 바로 리소스와 관련된 부분이다. 파드를 생성할때 `requests`와 `limits`을 통해 각 파드의 리소스를 제한할 수 있고,
내부 어플리케이션이 임계치를 넘어 리소스를 사용할 경우에는 무한 Restart가 실행될 수 있기 때문에 애플리케이션의 리소스 사용과 requests, limits의 관계에 대해서 정확히 파악해야 한다.

```yaml
resources:
  requests:
    memory: "100Mi"
    cpu: "100m"
  limits:
    memory: "200Mi"
    cpu: "200m"
```

requests는 **처음 예약되는 리소스**를 의미하고 limits는 **리소스 사용 임계치**를 말한다. 

CPU와 메모리를 나눠서 살펴보자. 우선 limits에 설정한 값보다 메모리 사용량이 늘어나면 어떻게 될까. 메모리 사용량을 초과한 컨테이너는 종료 대상 후보가 된다. 이후에 메모리 사용량이 줄어들지 않으면 OOMKILLED와 함께 컨테이너를 종료시킨다.
만약 RestartPolicy가 설정되어 있다면 Kubelet이 컨테이너를 재시작한다. 

그렇다면 CPU의 경우에는 어떻게 될까. CPU는 limits 까지 사용량이 증가하면 그 임계치에서 쓰로틀링이 걸린다. 메모리처럼 컨테이너가 다운되거나 하는 일은 없다.

이번에는 limits와 requests 중 하나만 설정했을때 어떻게 동작하는지 살펴보자. 먼저 limits는 설정했지만 requests는 설정하지 않았다면 어떻게 동작할까. CPU와 메모리 모두 requests가 limits에 맞춰서 설정된다.

반대로 requests는 설정했지만, limits를 설정하지 않았다면 어떻게 될까. CPU의 경우에는 노드가 제공하는 만큼 사용하다가 쓰로틀링이 걸릴 것이고, 메모리의 경우에는 노드 제공 가능 메모리 한계치까지 사용하다가 가용 메모리를 거의 다 사용하면 노드의 OOM Killer에 의해 프로세스 종료된다.

```yaml
spec:
  containers:
    - name: order-service
      image: sjmin/order-service:v1.0.0
      env:
        - name: JAVA_OPTIONS
          value: |
            -XX:+UseContainerSupport
            -XX:InitialRAMPercentage=60
            -XX:MinRAMPercentage=60
            -XX:MaxRAMPercentage=60
```

컨테이너 환경에서 자바 프로세스의 메모리를 사용할때는 Deployment 오브젝트에 위의 옵션을 줌으로써 노드 전체가 아닌 컨테이너가 할당받은 리소스에 한정해서 특정 비율만큼 RAM 크기를 설정할 수 있다.

## Reference

- [https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
- [https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [https://kubernetes.io/docs/tasks/configure-pod-container/assign-memory-resource/](https://kubernetes.io/docs/tasks/configure-pod-container/assign-memory-resource/)
- [https://kubernetes.io/docs/tasks/configure-pod-container/assign-cpu-resource/](https://kubernetes.io/docs/tasks/configure-pod-container/assign-cpu-resource/)

[nodejs]: https://nodejs.org/
[starter]: https://github.com/cotes2020/chirpy-starter
[pages-workflow-src]: https://docs.github.com/en/pages/getting-started-with-github-pages/configuring-a-publishing-source-for-your-github-pages-site#publishing-with-a-custom-github-actions-workflow
[latest-tag]: https://github.com/cotes2020/jekyll-theme-chirpy/tags
