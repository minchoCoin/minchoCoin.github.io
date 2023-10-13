---
title: "μC/OS-III ch.9 Interrupt Management"
last_modified_at: 2023-10-12T00:53:12+09:00
categories:
    - microc-os-stm32
tags:
    - microc-os
    - embedded-system

toc: true
toc_label: "My Table of Contents"
author_profile: true

---
# 서론
인터럽트(interrupt)는 비동기 이벤트가 발생했음을 CPU에 알리기 위해 사용되는 하드웨어 매커니즘이다. 인터럽트가 발견되면, CPU는 문맥(즉, 레지스터)의 일부 또는 전부를 저장하고, 인터럽트 서비스 루틴(ISR)이라고 불리는 특별한 서브루틴으로 이동한다. ISR은 이벤트를 처리하고, ISR이 할 일을 다 끝냈으면, 인터럽트 때문에 일시 중단되었던 task로 복귀하거나, ISR이 (현재 task보다) 더 높은 우선 순위의 task를 ready-to-run 상태로 만든 경우 가장 높은 우선순위의 task로 복귀한다.

인터럽트는 마이크로프로세서가 이벤트가 발생했을 때(즉, 비동기적으로) 이벤트를 처리할 수 있게 하는데, 이는 마이크로프로세서가 이벤트가 발생했는지 확인하기 위해 이벤트를 지속적으로 폴링(살피는 것)하는 것을 방지한다. 이벤트에 대한 task level 응답은 인터럽트 모드를 사용하는 것이 폴링 모드를 사용하는 것보다 더 좋다. 마이크로프로세서는 두 가지 특별한 명령어, 즉 인터럽트를 비활성화하고 활성화하는 것을 통해, 인터럽트를 무시하거나 인지할 수 있게 된다.

real time 환경에서, 인터럽트는 가능한 적게 비활성화 되어야한다. 인터럽트를 비활성화하는 것은 인터럽트 레이턴시(인터럽트 요청과 ISR 시작사이의 delay)에 영향을 미치며, 이는 인터럽트를 놓치게 한다.

프로세서는 일반적으로 인터럽트가 중첩되는 것을 허용하는데, 이는 인터럽트를 서비스하는 동안 프로세서가 다른 (더 중요한) 인터럽트를 인식하고 서비스하는 것을 의미한다.

실시간 커널의 가장 중요한 사양 중 하나는 인터럽트가 비활성화되는 최대 시간이다. 이를 interrupt disable time이라고한다. 모든 real-time 시스템은 코드의 임계 구역(critical section)을 조작하기 위해 인터럽트를 비활성화하고, 완료되면 인터럽트를 다시 활성화한다. 인터럽트가 비활성화되는 시간이 증가할수록, 인터럽트 레이턴시도 증가한다.

*Interrupt response*는 인터럽트 수신과 인터럽트를 처리하는 사용자 코드의 시작 사이의 시간으로 정의된다. 이 시간은 인터럽트를 처리하는데 드는 오버헤드를 나타낸다. 일반적으로, 프로세서의 문맥(CPU 레지스터)는 사용자 코드가 실행되기 전에 스택에 저장된다.

*Interrupt recovery*는 ISR이 task를 ready-to-run 상태로 만든 경우, 프로세서가 인터럽트 때문에 일시중지되었던 코드 또는 높은 우선순위의 작업으로 복귀하는데 필요한 시간으로 정의된다.

*Task latency*는 인터럽트가 발생한 후 task level 코드가 재개될 때까지 걸리는 시간으로 정의된다.

# 9-1 Handling CPU Interrupts
오늘날 시장에는 많은 CPU 아키텍처가 존재하며, 대부분의 프로세서는 일반적으로 프로세서는 다수의 소스로 부터 오는 인터럽트를 처리한다. 예를 들어, UART는 문자를 수신하고, 이더넷 컨트롤러는 패킷을 수신하고, DMA 컨트롤러는 데이터 전송을 완료하고, 아날로그-디지털 변환기(ADC)는 아날로그 변환을 완료하고, 타이머가 만료되는 등이 있다.

대부분, *interrupt controller*(인터럽트 컨트롤러) 는 Figure 9-1과 같이 프로세서로 제출하는 모든 각기 다른 인터럽트를 캡처한다(CPU 인터럽트 활성화/비활성화는 일반적으로 CPU의 일부이지만, 그림을 위해 별도로 표시됨).

인터럽트 장치는 인터럽트 컨트롤러에 신호를 보내고, 인터럽트 컨트롤러는 인터럽트의 우선순위를 정하고, CPU에 가장 높은 우선순위의 인터럽트를 제공한다.

![Interrupt controller](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/71282e4e-b719-4f55-81ed-1b43144d8658)

현대의 인터럽트 컨트롤러는 사용자가 인터럽트 우선순위를 정하고, 어떤 인터럽트가 아직 대기 중인지 기억할 수 있다. 그리고 많은 경우, 인터럽트 컨트롤러가 ISR의 주소(vector 주소라고도 함)를 CPU에 직접 제공하도록 해준다.

전역(global) 인터럽트(즉, Figure 9-1의 스위치)가 비활성화되면, CPU는 인터럽트 컨트롤러의 요청을 무시하고, 그 요청은 CPU가 인터럽트를 다시 활성화할 때 까지 인터럽트 컨트롤러에 의해 보류된다.

CPU는 아래 2가지 방법중 하나를 이용하여 인터럽트를 처리한다.

1. 모든 인터럽트가 하나의 인터럽트 핸들러를 가리킨다(vector).
2. 각 인터럽트는 각각의 인터럽트 핸들러를 직접 가리킨다.

이 두 가지 방법을 논의하기 전에, μC/OS-III가 CPU 인터럽트를 어떻게 처리하는지 이해하는 것이 중요하다.

# Typical μC/OS-III INTERRUPT SERVICE ROUTINE (ISR)
μC/OS-III에서는 인터럽트 서비스 루틴이 어셈블리어로 작성되어야 한다. 그러나 C 컴파일러가 인라인 어셈블리어를 지원한다면 ISR 코드는 C 소스 파일에 직접 넣을 수 있다. μC/OS-III를 사용할 때의 전형적인 ISR에 대한 의사 코드는 Listing 9-1과 같다.

```c
MyISR: (1)
    Disable all interrupts; (2)
    Save the CPU registers; (3)
    OSIntNestingCtr++; (4)
    if (OSIntNestingCtr == 1) { (5)
        OSTCBCurPtr->StkPtr = Current task’s CPU stack pointer register value;
    }
    Clear interrupting device; (6)
    Re-enable interrupts (optional); (7)
    Call user ISR; (8)
    OSIntExit(); (9)
    Restore the CPU registers; (10)
    Return from interrupt; (11)
// Listing 9-1
```
## L9-1(1)
이전에 서술한 바와 같이, ISR은 보통 어셈블리 언어로 작성된다. My ISR은 인터럽트를 발생시킨 디바이스를 처리할 핸들러의 이름에 대응한다.

## L9-1(2)
먼저, 모든 인터럽트가 비활성화되는 것이 중요하다. 일부 프로세서는 인터럽트 핸들러가 시작될 때마다 인터럽트가 비활성화된다. 그게 아니면 여기서 보여지는 바와 같이 사용자가 인터럽트를 명시적으로 비활성화 시켜야 된다. 프로세서가 상이한 인터럽트 우선순위 레벨들을 지원하는 경우 이 단계는 까다로울 수 있다. 그러나 문제를 해결할 수 있는 방법은 항상 존재한다.

## L9-1(3)
인터럽트 핸들러가 가장 먼저 해야 할 일은 CPU의 문맥(context)를 인터럽트된 task의 스택에 저장하는 것이다. 일부 프로세서에서는 이러한 현상이 자동으로 발생한다. 그러나 대부분의 프로세서에서는 CPU 레지스터를 task의 스택에 저장하는 방법을 아는 것이 중요하다. CPU의 전체 문맥을 저장해야 하는데, 이는 사용되는 CPU에 FPU가 장착되어 있는 경우 FPU(Floating-Point Unit) 레지스터도 포함할 수 있다.

특정 CPU들은 단지 인터럽트들(즉, 인터럽트 스택)을 처리하기 위해 특별한 스택으로 자동 전환하기도 한다. 이는 귀중한 task 스택 공간 사용을 피하기 때문에 일반적으로 유용하다. 그러나 μC/OS-III의 경우, 인터럽트된 task의 문맥은 해당 task의 스택에 저장될 필요가 있다.

프로세서에 ISR을 처리할 전용 스택 포인터가 없다면 소프트웨어적으로 구현하는 것이 가능하다. 구체적으로, ISR을 입력할 때 단순히 현재 task 스택을 저장하고 전용 ISR 스택으로 전환한 후 다시 task 스택으로 전환하면 된다. 물론 이것은 쓸 코드가 추가로 있다는 것을 의미하지만 중첩 인터럽트를 포함한 최악의 경우 인터럽트 스택 사용을 수용하기 위해 task 스택에 여분의 공간을 할당할 필요는 없기 때문에 이점은 엄청나다.
# Reference
 - uC/OS-III: The Real-Time Kernel For the STM32 ARM Cortex-M3, Jean J. Labrosse, Micrium, 2009

 - https://en.wikipedia.org/wiki/Interrupt_latency

[책 링크](https://micrium.atlassian.net/wiki/spaces/osiiidoc/overview)
