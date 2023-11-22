---
title: "μC/OS-III ch.15 Synchronization"
last_modified_at: 2023-11-22T11:18:12+09:00
categories:
    - microc-os-3-stm32
tags:
    - microc-os
    - embedded-system

toc: true
toc_label: "My Table of Contents"
author_profile: true

---
task나 ISR이 다른 task에게 정보를 전달하는 것이 때때로 필요하다. 이러한 정보전달을 task간 통신(inter-task communication)이라 불린다. 정보는 전역 데이터를 통해 또는 메시지를 전송하는 두 가지 방법으로 task끼리 주고 받을 수 있다.

231쪽의 Chapter 13 "Resource Management"에서 본 것과 같이, 전역 변수를 사용할 때는 각 task나 ISR이 변수에 대한 배타적 접근을 보장해야한다. ISR이 있다면 공유 변수에 대한 배타적 접근을 보장하는 유일한 방법은 인터럽트를 비활성화하는 것이다. 두 task가 데이터를 동규한다면 인터럽트를 비활성화하거나 스케줄러를 잠그거나 세마포어를 사용하거나, 상호 배제 세마포어(mutex)를 사용하여 변수에 대한 배타적 접근 권한을 얻을 수 있다. task는 전역 변수를 사용해야만 정보를 ISR에 전달할 수 있다는 점에 유의한다. ISR이 task에 신호를 보내거나 task가 변수의 내용을 주기적으로 폴링하여 확인하지 않는 한, task는 ISR이 전역변수를 변경하여도 인식하지 못한다.

μC/OS-III에서 각 task는 메시지 큐를 내장하고 있으므로 메시지를 메시지 큐라는 중간 객체로 보내거나 직접 태스크로 보낼 수 있다. 여러 task가 메시지를 기다리는 경우에는 외부 메시지 큐를 사용할 수 있다. 만약 한 task만이 수신한 데이터를 처리할 것이라면 직접 task로 메시지를 보낼 것이다.

task는 메시지가 도착하기를 기다릴 때 CPU시간을 소모하지 않는다.

# Messages
메시지는 데이터를 가리키는 포인터, 테이터의 크기를 가지는 변수, 그리고 메시지가 언제 전송되었는지를 나타내는 타임스탬프로 구성된다. 포인터는 데이터 영역을 가리킬 수도 있고, 함수를 가리킬 수도 있다. 메시지의 내용과 의미에 대해서는 송신자와 수신자가 합의를 해야한다. 즉, 메시지의 수신자가 수신한 메시지의 의미를 알아야 처리할 수 있을 것이다. 예를 들어 이더넷 컨트롤러는 이 패킷에 대한 포인터를 처리할 줄 아는 task로 보낼 것이다.

실제로 데이터는 값이 아니라 참조에 의해 전달되기 때문에 메시지 내용은 항상 범위 안에 있어야 한다. 즉, 전송되는 데이터는 복사되지 않는다. 343페이지의 Chapter 17 "Memory Management"에서 설명한 대로 동적으로 할당된 메모리를 사용하는 것을 고려해 볼 수 있다. 또는 전역 변수, 전역 자료 구조, 전역 배열, 함수 등에 포인터를 전달할 수 있다.

# Reference
 - uC/OS-III: The Real-Time Kernel For the STM32 ARM Cortex-M3, Jean J. Labrosse, Micrium, 2009

[책 링크](https://micrium.atlassian.net/wiki/spaces/osiiidoc/overview)