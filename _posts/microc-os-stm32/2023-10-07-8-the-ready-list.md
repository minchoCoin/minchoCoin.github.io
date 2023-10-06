---
title: "μC/OS-III ch.6 The Ready List"
last_modified_at: 2023-10-07T12:53:12+09:00
categories:
    - microc-os-stm32
tags:
    - microc-os
    - embedded-system

toc: true
toc_label: "My Table of Contents"
author_profile: true

---

실행 준비가 된 task는 ready list에 배치된다. ready list는 두 부분으로 구성된다. 하나는 실행 준비가 된 task의 우선순위를 가지고 있는 비트맵과 실행 준비가 된 모든 task에 대한 포인터 테이블이다.

# 6-1 Priority levels
그림 6-1에서 6-3은 