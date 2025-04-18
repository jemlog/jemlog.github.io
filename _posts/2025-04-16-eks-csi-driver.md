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

### 정적 프로비저닝

### 동적 프로비저닝

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

### AWS EFS와 EFS CSI Driver 사용 예시


[nodejs]: https://nodejs.org/
[starter]: https://github.com/cotes2020/chirpy-starter
[pages-workflow-src]: https://docs.github.com/en/pages/getting-started-with-github-pages/configuring-a-publishing-source-for-your-github-pages-site#publishing-with-a-custom-github-actions-workflow
[latest-tag]: https://github.com/cotes2020/jekyll-theme-chirpy/tags
