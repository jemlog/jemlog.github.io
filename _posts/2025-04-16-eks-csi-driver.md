---
title: AWS CSI Driver를 사용한 EKS 내 스토리지 관리 (작성중)
date: 2025-04-16 00:20:00
categories: [Kubernetes]
tags: [kubernetes, EKS]
pin: false
image:
  path: '/assets/img/eks5.png'
---

EKS를 학습하며 스토리지를 관리하는 방법에 대해 학습한 내용을 정리합니다.

## 쿠버네티스의 기본 스토리지 관리 방법

CSI Driver를 학습하기에 앞서 쿠버네티스의 기본 볼륨 관리 방식에 대해 간단히 정리하고 넘어가자. 쿠버네티스에서 볼륨을 할당하기 위해서는 **PVC**(Persistence Volume Claim), **PV**(Persistence Volume), **StorageClass**가 사용된다.

- PV: 쿠버네티스 관리자 측면에서 실제 볼륨과 매핑되는 쿠버네티스 리소스
- PVC: 쿠버네티스 사용자 측면에서 파드와 볼륨을 매핑하는 쿠버네티스 리소스
- StorageClass: 동적 프로비저닝 방식에 사용되는 리소스

실제 볼륨을 파드에 마운트하는 과정을 프로비저닝이라고 하는데, 정적 프로비저닝과 동적 프로비저닝 두 종류가 있다.

### 정적 프로비저닝

정적 프로비저닝은 사용자가 직접 PV와 스토리지 볼륨 생성을 수행하는 방식이다. 이 경우에는 관리자가 직접 PV 리소스를 생성해야 한다.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: static-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /mnt/data
```
사용자는 PV를 사용하기 위한 PVC 리소스를 생성한다. 이때 PVC에서 직접적으로 PV를 지정하지는 않는데, 쿠버네티스는 resources.requests.storage와 accessModes가 일치하는 PV를 알아서 찾아서 매핑한다.
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: use-static-pv
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```
### 동적 프로비저닝

동적 프로비저닝은 사용자가 PVC와 StorageClass만 정의하면 쿠버네티스가 자동으로 PV와 실제 볼륨 생성을 수행해주는 방식이다. 

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer # 파드가 하나라도 연결되면 그때 볼륨 생성
```
사용자가 PVC를 만들때 StorageClass를 지정하면 실제 볼륨과 PV는 자동으로 생성된다. volumeBindingMode 옵션을 통해 어느 시점에 볼륨이 생성될지를 컨트롤 할 수 있다.
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ebs-sc # StorageClass를 명시적으로 지정
  resources:
    requests:
      storage: 10Gi
```

정적 프로비저닝을 사용하면 관리자가 미리 PV와 볼륨 스토리지를 세팅해둬야 하기에 유연하지 않다는 단점이 있다. 따라서 유동적으로 리소스를 생성/할당/삭제 할 수 있는 동적 프로비저닝이 선호된다.

## AWS CSI Driver를 활용한 EKS 스토리지 할당

그렇다면 AWS 상에서는 어떻게 볼륨을 사용할 수 있을까. PV가 EC2 instance의 내부 디렉토리를 사용할 수도 있지만 외부 스토리지 공급자와 연결될 수도 있다. 
외부 공급자로는 AWS의 관리형 스토리지인 **AWS EBS**와 **AWS EFS**를 사용할 수 있다. 그리고 AWS의 관리형 스토리지들과 AWS EKS 내의 PV를 연결해주는 역할을 **CSI Driver**가 수행한다.

### CSI란 무엇인가

우선 CSI라는게 어떤건지부터 알고 넘어가자. **CSI (Container Storage Interface)**는 컨테이너 오케스트레이터들이 스토리지 시스템과 일관되게 연동할 수 있도록 하는 공식 인터페이스 표준이다.

표준이 존재하기 전까지는 쿠버네티스가 자체적으로 EBS, NFS 등의 저장소에 대한 스토리지 드라이버를 코드 베이스가 가지고 있었다. 이렇게 되면 새로운 스토리지들을 지원해야 할때마다 직접 쿠버테니스 코드를 변경해야 했다.
이 문제를 해결하기 위해 CSI 라는 일관된 인터페이스가 등장했고, 벤더들은 이 CSI에 맞춰서 플러그인을 쿠버네티스에 제공만 하면 된다.

개인적으로 쿠버네티스는 추상화를 통한 유연함의 이점을 최대로 가져가는 아키텍처라고 생각한다. CRI (Container Runtime Interface), CNI (Container Network Interface) 등의 인터페이스를 이용해 쿠버네티스가 특정 기술에 종속되는걸 방지했고, 이는 여러 플러그인 생태계가 활성화 될 수 있는 계기가 됐다고 본다.
 

### AWS EBS와 EBS CSI Driver 사용 예시

EBS는 EC2 인스턴스에 직접 연결되는 Volume storage이다. 특징으로는 사용자와 1:1로 연결되고 하나의 AZ에만 위치할 수 있다. 즉, PV의 AccessMode를 **ReadWriteOnce**만 사용 가능하다. 또한 PV의 NodeAffinity를 EBS가 위치한 AZ로 지정해서 파드가 같은 AZ에 할당되도록 해야 한다. 만약 다른 AZ로 배정되면 volume mount가 불가능하다.

**Amazon EBS CSI Driver**는 PV의 프로비저닝이나 마운트 등의 작업을 수행한다. 또한 EBS 볼륨을 동적으로 생성하고 삭제하는 라이프사이클을 관리한다.

<img src="/assets/img/ebs-csi2.png" alt="EBS CSI Driver 이미지">

EBS CSI Driver는 크게 두개의 컴포넌트로 구성된다. **ebs-csi-controller**는 Deployment로 설치되고 **ebs-csi-node**는 DaemonSet으로 설치가 된다. 각각의 파드에는 세부적인 기능을 수행하는 컨테이너들이 할당된다. 각각의 기능을 간단히 정리해보자.

- **controller-plugin**: 컨트롤러 사이드에서 EBS 볼륨 생성, 삭제, Attach, Detach등의 제어를 담당한다.
- **resizer**: PVC 크기 변경하려고 할때 이를 감지하고 볼륨 확장 요청을 보낸다.
- **snapshotter**: VolumeSnapshot 리소스를 생성하면 CSI 드라이버에게 Snapshot 생성,삭제,복원 작업을 수행한다.
- **provisioner**: PVC를 감지하고 CSI Driver에게 실제 PV 생성을 요청한다.
- **attacher**: 스토리지 볼륨을 Pod가 실행되는 노드에 Attach, Detach 한다.
- **node-plugin**: 볼륨을 마운트하고 파일 시스템을 준비하는 작업을 수행
- **node-register**: node-plugin을 kubelet에 등록해주는 사이드카
- **liveness-probe**: CSI Driver가 정상 동작하는지 지속적으로 체크한다.

위의 컨테이너들은 서로 상호작용하며 AWS SDK를 사용해 EBS API를 통해 외부 스토리지와 EKS 노드를 연결해준다.

그렇다면 어떤 과정을 거쳐서 사용하는지 살펴보자. 먼저 EBS CSI Driver를 위한 IRSA를 만들어줘야 한다. 여기서 **IRSA**는 IAM Roles for Service Accounts의 줄임말로 AWS EKS에서 Kubernetes ServiceAccount에 IAM Role을 연결할 수 있도록 해준다.

```bash
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster my-cluster \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \ # IAM Role에 연결할 Policy
  --approve \ # 생성 시 자동 승인
  --role-only \ # IAM Role만 생성
  --role-name AmazonEKS_EBS_CSI_DriverRole # Role Name 지정
```

[nodejs]: https://nodejs.org/
[starter]: https://github.com/cotes2020/chirpy-starter
[pages-workflow-src]: https://docs.github.com/en/pages/getting-started-with-github-pages/configuring-a-publishing-source-for-your-github-pages-site#publishing-with-a-custom-github-actions-workflow
[latest-tag]: https://github.com/cotes2020/jekyll-theme-chirpy/tags
