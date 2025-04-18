---
title: AWS CSI Driver를 사용한 EKS 내 스토리지 관리
date: 2025-04-16 00:20:00
categories: [Kubernetes]
tags: [kubernetes]
pin: false
image:
  path: '/assets/img/eks5.png'
---

EKS를 학습하며 스토리지를 관리하는 방법에 대해 학습한 내용을 정리합니다.

## 쿠버네티스의 기본 스토리지 관리 방법

### 정적 프로비저닝

### 동적 프로비저닝

## AWS CSI Driver를 활용한 EKS 스토리지 할당

Amazon EBS CSI Drvier는 EKS 클러스터에 구성되는 PV를 위해 Amazon EBS 볼륨의 생명 주기를 관리한다. 즉, Amazon EBS와 Amazon EKS의 중간 연결고리가 되는 것이다.

### CSI Driver의 구조

### EBS CSI Driver 사용 예시

### EFS CSI Driver 사용 예시


[nodejs]: https://nodejs.org/
[starter]: https://github.com/cotes2020/chirpy-starter
[pages-workflow-src]: https://docs.github.com/en/pages/getting-started-with-github-pages/configuring-a-publishing-source-for-your-github-pages-site#publishing-with-a-custom-github-actions-workflow
[latest-tag]: https://github.com/cotes2020/jekyll-theme-chirpy/tags
