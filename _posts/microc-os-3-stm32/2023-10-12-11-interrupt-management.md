---
title: "μC/OS-III ch.9 Interrupt Management"
last_modified_at: 2023-10-12T00:53:12+09:00
categories:
    - microc-os-3-stm32
tags:
    - microc-os
    - embedded-system

toc: true
toc_label: "My Table of Contents"
author_profile: true

---

이 글은 'uC/OS-III: The Real-Time Kernel For the STM32 ARM Cortex-M3, Jean J. Labrosse, Micrium, 2009'를 번역한 글입니다. 오역이 있을 수 있으며, 발견하시면 github에 issue나 댓글 남겨주시기 바랍니다.

# 서론
인터럽트(interrupt)는 비동기 이벤트가 발생했음을 CPU에 알리기 위해 사용되는 하드웨어 매커니즘이다. 인터럽트가 발견되면, CPU는 문맥(즉, 레지스터)의 일부 또는 전부를 저장하고, 인터럽트 서비스 루틴(ISR)이라고 불리는 특별한 서브루틴으로 이동한다. ISR은 이벤트를 처리하고, ISR이 할 일을 다 끝냈으면, 인터럽트 때문에 일시 중단되었던 task로 복귀하거나, ISR이 (현재 task보다) 더 높은 우선 순위의 task를 ready-to-run 상태로 만든 경우 가장 높은 우선순위의 task로 복귀한다.

인터럽트는 마이크로프로세서가 이벤트가 발생했을 때(즉, 비동기적으로) 이벤트를 처리할 수 있게 하는데, 이는 마이크로프로세서가 이벤트가 발생했는지 확인하기 위해 이벤트를 지속적으로 폴링(살피는 것)하는 것을 방지한다. 이벤트에 대한 task level 응답은 인터럽트 모드를 사용하는 것이 폴링 모드를 사용하는 것보다 더 좋다. 마이크로프로세서는 두 가지 특별한 명령어, 즉 인터럽트를 비활성화하고 활성화하는 것을 통해, 인터럽트를 무시하거나 인지할 수 있게 된다.

real time 환경에서, 인터럽트는 가능한 적게 비활성화 되어야한다. 인터럽트를 비활성화하는 것은 인터럽트 레이턴시에 영향을 미치며, 이는 인터럽트를 놓치게 한다.

프로세서는 일반적으로 인터럽트가 중첩되는 것을 허용하는데, 이는 인터럽트를 서비스하는 동안 프로세서가 다른 (더 중요한) 인터럽트를 인식하고 서비스하는 것을 의미한다.

실시간 커널의 가장 중요한 사양 중 하나는 인터럽트가 비활성화되는 최대 시간이다. 이를 interrupt latency이라고한다. 모든 real-time 시스템은 코드의 임계 구역(critical section)을 조작하기 위해 인터럽트를 비활성화하고, 완료되면 인터럽트를 다시 활성화한다. 인터럽트가 비활성화되는 시간이 증가할수록, 인터럽트 레이턴시도 증가한다.

*Interrupt response*는 인터럽트 수신과 인터럽트를 처리하는 사용자 코드의 시작 사이의 시간으로 정의된다. 이 시간은 인터럽트를 처리하는데 드는 오버헤드를 나타낸다. 일반적으로, 프로세서의 문맥(CPU 레지스터)는 사용자 코드가 실행되기 전에 스택에 저장된다.

*Interrupt recovery*는 ISR이 프로세서가 인터럽트 때문에 일시중지되었던 코드 또는  task를 ready-to-run 상태로 만든 경우, 높은 우선순위의 작업으로 복귀하는데 필요한 시간으로 정의된다.

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

# Typical μC/OS-III Interrupt Service Routine (ISR)
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
// Listing 9-1 ISRs under μC/OS-III (assembly language)
```
## L9-1(1)
이전에 서술한 바와 같이, ISR은 보통 어셈블리 언어로 작성된다. My ISR은 인터럽트를 발생시킨 디바이스를 처리할 핸들러의 이름에 대응한다.

## L9-1(2)
먼저, 모든 인터럽트가 비활성화되는 것이 중요하다. 일부 프로세서는 인터럽트 핸들러가 시작될 때마다 인터럽트가 비활성화된다. 그게 아니면 여기서 보여지는 바와 같이 사용자가 인터럽트를 명시적으로 비활성화 시켜야 된다. 프로세서가 상이한 인터럽트 우선순위 레벨들을 지원하는 경우 이 단계는 까다로울 수 있다. 그러나 문제를 해결할 수 있는 방법은 항상 존재한다.

## L9-1(3)
인터럽트 핸들러가 가장 먼저 해야 할 일은 CPU의 문맥(context)를 인터럽트된 task의 스택에 저장하는 것이다. 일부 프로세서에서는 이러한 현상이 자동으로 발생한다. 그러나 대부분의 프로세서에서는 CPU 레지스터를 task의 스택에 저장하는 방법을 아는 것이 중요하다. CPU의 전체 문맥을 저장해야 하는데, 이는 사용되는 CPU에 FPU가 장착되어 있는 경우 FPU(Floating-Point Unit) 레지스터도 포함할 수 있다.

특정 CPU들은 단지 인터럽트들(즉, 인터럽트 스택)을 처리하기 위해 특별한 스택으로 자동 전환하기도 한다. 이는 귀중한 task 스택 공간 사용을 피하기 때문에 일반적으로 유용하다. 그러나 μC/OS-III의 경우, 인터럽트된 task의 문맥은 해당 task의 스택에 저장될 필요가 있다.

프로세서에 ISR을 처리할 전용 스택 포인터가 없다면 소프트웨어적으로 구현하는 것이 가능하다. 구체적으로, ISR에 진입할 때 단순히 현재 task 스택을 저장하고 전용 ISR 스택으로 전환한 후 다시 task 스택으로 전환하면 된다. 물론 이것은 쓸 코드가 추가로 있다는 것을 의미하지만 중첩 인터럽트를 포함한 최악의 경우 인터럽트 스택 사용을 수용하기 위해 task 스택에 여분의 공간을 할당할 필요는 없기 때문에 이점은 엄청나다.

## L9-1(4)
다음으로 OSIntEnter()를 호출하거나, 단순히 어셈블리어로 변수 OSIntNestingCtr를 증가시킨다. 이는 일반적으로 상당히 쉽고 OSIntEnter()를 호출하는 것보다 더 효율적이다. 이름에서 알 수 있듯이 OSIntNestingCtr는 인터럽트 네스팅 레벨을 저장한다.

## L9-1(5)
이것이 첫 번째 중첩 인터럽트라면, 인터럽트된 task의 스택 포인터의 현재 값을 그것의 OS_TCB에 저장할 필요가 있다. 글로벌 포인터 OSTCBCurPtr은 인터럽트된 작업의 OS_TCB를 가리킨다. OS_TCB의 바로 첫 번째 필드가 스택 포인터를 저장할 필요가 있는 곳이다. 즉, OS_TCB에서 OSTCBCurPtr->StkPtr은 오프셋 0에 있게 된다(이는 어셈블리어를 크게 단순화한다).

## L9-1(6)
이때, 당신은 인터럽트 장치가 동일한 다른 인터럽트를 발생시키지 않도록 인터럽트 장치를 클리어할 필요가 있다. 그러나, 대부분의 사람들은 소스의 클리어를 연기하고, "C"에서 ISR 핸들러 내에서 해당 동작을 수행하는 것을 선호한다.

## L9-1(7)
이때 중첩 인터럽트를 허용하고 싶다면, 인터럽트를 다시 활성화해도 괜찮다. 이 단계는 선택 사항이다.

## L9-1(8)
이 시점에서, 추가 처리는 어셈블리 언어로부터 호출된 C 함수로 지연될 수 있다. 이는 ISR 핸들러에서 해야 할 처리의 양이 많은 경우에 특히 유용하다. 그러나, 일반적으로, ISR들은 가능한 한 짧게 유지한다. 사실, 단순히 task에 신호를 보내거나 메시지를 보내고 task이 인터럽트 장치를 서비스하는 세부사항들을 처리하도록 하는 것이 최선이다.

ISR은 OSSemPost(), OSTaskSemPost(), OSFlagPost(), OSQPost() 또는 OSTaskQPost() 중 하나의 함수를 호출해야 한다. 이것은 ISR이 인터럽트 장치를 서비스할 task에게 알려야하기 때문에 필요하다. 이것들은 ISR에서 호출할 수 있는 유일한 함수들이며 task에 신호를 보내거나 메시지를 보낼 때 사용된다. 그러나 만약 ISR이 이러한 함수들 중 하나를 호출할 필요가 없다면, 다음 절에서 설명하는 것처럼 ISR을 "Non Kernel-Aware Interrupt Service Routine"으로 쓰는 것을 고려해 보자.

## L9-1(9)
ISR이 완료되면 OSIntExit()을 호출하여 μC/OS-III에게 ISR이 완료되었음을 알려주어야 한다. OSIntExit()은 OSIntNestingCtr를 단순히 감소시키고 OSIntNestingCtr이 0에 도달하면 ISR이 (이전에 중단된 ISR이 아닌) task level 코드로 돌아가려고 함을 나타낸다. μC/OS-III은 중첩된 ISR 중 하나 때문에 실행할 필요가 있는 더 높은 우선순위의 task가 있는지 판단해야 할 것이다.  즉, ISR은 이 신호나 메시지를 기다리는 더 높은 우선순위의 task에 신호를 보내거나 메시지를 보냈을 수 있다. 이 경우 μC/OS-III은 중단된 작업으로 돌아가는 대신에 문맥이 더 높은 우선순위의 작업으로 전환할 것이다. 후자의 경우 OSIntExit()은 실제로 리턴하지 않고 다른 경로를 취한다.

## L9-1(10)
ISR이 인터럽트된 작업보다 우선순위가 낮은 작업에 신호를 보내거나 메시지를 보낸 경우 OSIntExit()이 리턴하고 종료된다. 이는 인터럽트된 task가 여전히 실행해야 할 우선순위가 가장 높은 task이며 이전에 저장된 레지스터를 복원하는 것이 중요하다는 것을 의미한다.

## L9-1(11)
ISR은 인터럽트로부터 돌아오고, 인터럽트된 task를 재개한다.

참고: (1)부터 (6)까지는 ISR 프롤로그, (9)부터 (11)까지는 ISR 에필로그로 지칭된다.

# Non kernel-Aware Interrupt Service Routine(ISR)
위의 순서는 ISR이 작업에 신호를 보내거나 메시지를 보낸다고 가정한다. 그러나 많은 경우에 ISR은 작업을 통지할 필요가 없을 수도 있고, ISR 내에서 모든 작업을 간단히 수행할 수 있다(빠른 시간에 수행할 수 있다고 가정). 이 경우 ISR은 L9-2와 같이 나타날 것이다.

```c
MyShortISR: (1)
    Save enough registers as needed by the ISR; (2)
    Clear interrupting device; (3)
    DO NOT re-enable interrupts; (4)
    Call user ISR; (5)
    Restore the saved CPU registers; (6)
    Return from interrupt; (7)
//L9-2 Non-Kernel Aware ISRs with μC/OS-III
```
## L9-2(1)
ISR은 전형적으로 어셈블리 언어로 작성된다. MyShort ISR은 인터럽트 장치를 처리할 핸들러의 이름에 대응한다.

## L9-2(2)
ISR을 처리하는 데 필요한 만큼 충분한 레지스터를 저장한다.

## L9-2(3)
ISR이 반환되면 동일한 인터럽트가 발생하지 않도록 인터럽트 장치를 클리어할 필요가 있을 것이다.

## L9-2(4)
다른 인터럽트가 μC/OS-III 호출을 하여 문맥을 더 높은 우선순위의 task로 강제 전환할 수 있으므로 이 시점에서 인터럽트를 다시 활성화하면 안된다. 이는 위 ISR이 완료되지만 훨씬 나중에 완료됨을 의미한다.

## L9-2(5)
이제 인터럽트 장치를 어셈블리어로 처리하거나 필요한 경우 C 함수를 호출할 수 있습니다.

## L9-2(6)
완료되면 저장된 CPU 레지스터를 복원하기만 하면 된다.

## L9-2(7)
인터럽트로 부터 돌아오고, 인터럽트된 task를 재개한다.

# Processors With Multiple Interrupt Priorities
Fig 9-2와 같이 실제로 다중 인터럽트 레벨을 지원하는 프로세서가 있다.

![Fig 9-2 Kernel Aware and Non-Kernel Aware Interrupts](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/df35a36f-586f-453b-9d1b-2357ad7e445f)

## F9-2(1)
프로세서가 16개의 상이한 인터럽트 우선 순위 레벨들을 지원한다고 가정한다. 
우선 순위 0은 가장 낮은 우선 순위인 반면, 15는 가장 높다. 보여지는 바와 같이, 인터럽트들은 항상 task들(인터럽트가 활성화되었다고 가정함)보다 우선 순위가 더 높다.

## F9-2(2)
여기서는 인터럽트 레벨 0에서 12가 인터럽트를 서비스하도록 할당된 task에 알리기 위해 μC/OS-III post 함수 호출을 하는 것이 허용한다. 임계 구역(critical section)에 들어갈 때 인터럽트를 비활성화하는 것은 인터럽트 마스크를 레벨 12로 상승시키는 것을 의미한다. 즉, 인터럽트 레벨 0~11은 비활성화 되지만, 레벨 12 이상은 허용될 것이다.

## F9-2(3)
인터럽트 레벨 12에서 15는 어떠한 μC/OS-III 함수 호출도 허용되지 않으므로, L9-2와 같이 구현된다. μC/OS-III는 이러한 인터럽트들을 비활성화할 수 없으므로, 이러한 인터럽트의 인터럽트 레이턴시는 매우 짧다.

# Non-Kernel Aware ISRs for Processors with Multiple Priority Levels

```c
MyNonKernelAwareISR: (1)
    Save enough registers as needed by the ISR; (2)
    Clear interrupting device; (3)
    Call user ISR; (4)
    Restore the saved CPU registers; (5)
    Return from interrupt; (6)
```
## L9-3(1)
이전에 서술한 것 같이, ISR은 보통 어셈블리 언어로 작성된다. MyNonKernelAwareISR은 인터럽트를 발생시킨 디바이스를 처리할 핸들러의 이름에 대응한다.

## L9-3(2)
여기서는 ISR을 처리하는 데 필요한 만큼 충분한 레지스터를 저장한다.

## L9-3(3)
동일한 인터럽트가 발생하지 않도록 인터럽트 장치를 클리어할 필요가 있다.

## L9-3(4)
이제 인터럽트를 발생시킨 장치를 처리하거나 필요한 경우 C 함수를 호출할 수 있다.

## L9-3(5)
끝나면, 저장된 CPU 레지스터를 복원한다.

## L9-3(6)
인터럽트로 부터 돌아오고, 인터럽트된 task를 재개한다.

# All Interupts Vector To A Common Location
인터럽트 컨트롤러가 대부분 존재하더라도 일부 CPU는 여전히 공통의 인터럽트 핸들러로 이동한다. ISR은 인터럽트 컨트롤러에 인터럽트를 발생시킨 장치(인터럽트 소스)를 찾기 위해 질의한다. 얼핏 보면 대부분의 인터럽트 컨트롤러가 CPU를 적절한 인터럽트 핸들러로 이동하도록 강제할 수 있기 때문에 이상하게 보일 수 있다. 그러나 μC/OS-III의 경우 각 인터럽트 소스에 대해 고유한 ISR 핸들러로 분기(vectoring)하는 것보다 인터럽트 컨트롤러 벡터를 단일 ISR 핸들러로 분기하는 것이 더 쉽다.

```c
An interrupt occurs; (1)
The CPU vectors to a common location; (2)
The ISR code performs the “ISR prologue” (3)
The C handler performs the following: (4)
    while (there are still interrupts to process) { (5)
    Get vector address from interrupt controller;
    Call interrupt handler;
    }
The “ISR epilogue” is executed; (6)
```
## L9-4(1)
인터럽트가 임의의 장치로부터 발생한다. 인터럽트 컨트롤러는 CPU 상에서 인터럽트 핀을 활성화한다. 만약 첫 번째 인터럽트 이후에 발생하는 다른 인터럽트가 있다면, 인터럽트 컨트롤러는 이후에 발생한 인터럽트를 잠시 보류하고, 우선순위를 적절하게 정할 것이다.

## L9-4(2)
CPU는 단일 인터럽트 핸들러의 주소로 분기한다. 다시 말해서, 모든 인터럽트는 이 하나의 인터럽트 핸들러에 의해서 처리되어야 한다.

## L9-4(3)
ISR은 앞서 설명한 바와 같이 μC/OS-III 에서 필요로 하는 "ISR 프롤로그" 코드를 실행한다. 이를 통해 모든 ISR이 μC/OS-III post 함수 호출을 할 수 있게 된다.

## L9-4(4)
μC/OS-III C 핸들러를 호출하면 ISR 처리를 계속할 것이다. 이것은 코드를 쓰기(그리고 읽기) 쉽게 만든다. 인터럽트는 다시 활성화되지 않는다.

## L9-4(5)
그런 다음 μC/OS-III C 핸들러는 인터럽트 컨트롤러에게 인터럽트를 발생시킨 장치를 묻는다. 인터럽트 컨트롤러는 가장 높은 우선순위의 인터럽트 장치에 대한 인터럽트 핸들러의 주소 또는 숫자 (0 ~ N-1)로 응답할 것이다. 물론, μC/OS-III C 핸들러는 해당 컨트롤러를 위해 특별히 작성되었으므로 특정 인터럽트 컨트롤러를 어떻게 처리해야 하는지 알 것이다. 

인터럽트 컨트롤러가 0과 N-1 사이의 숫자를 제공하면, C 핸들러는 단순히 이 숫자를 인터럽트 디바이스와 연관된 인터럽트 서비스 루틴의 주소를 포함하는 테이블(ROM 또는 RAM 내)에 인덱스로서 사용한다. RAM 테이블은 런타임에 인터럽트 핸들러를 변경하는 데 유용하다. 그러나 많은 임베디드 시스템의 경우 테이블은 또한 ROM에 상주할 수 있다.

인터럽트 컨트롤러가 인터럽트 서비스 루틴의 주소로 응답하면, C 핸들러는 이 함수를 호출하기만 하면 된다.

위의 두 경우 모두 모든 인터럽트 핸들러는 다음과 같이 선언되어야 한다:

void MyISRHandler (void);

각각의 인터럽트 소스(분명히, 각각은 고유한 이름을 가짐)에 대해 각각 핸들러가 하나 있다.
"while" 루프는 서비스할 다른 인터럽트 장치가 없을 때 종료된다.

## L9-4(6)
μC/OS-III "ISR epilogue"는 인터럽트된 task로 돌아갈 필요가 있는지, 아니면 더 중요한 task로 전환할 필요가 있는지 확인하기 위해 실행된다.

## interesting points to note
C 핸들러가 인터럽트 컨트롤러에게 질문을 하기 전에 다른 장치에서 인터럽트가 발생했다면, 인터럽트 핸들러는 그 인터럽트를 잡을 가능성이 높다. 제 2 장치가 더 높은 우선순위의 인터럽트 장치가 된다면, 인터럽트 컨트롤러가 인터럽트들의 우선순위를 정할 것이기 때문에, 그것은 아마도 가장 먼저 서비스될 것이다.

보류 중인 모든 인터럽트가 서비스될 때까지 루프는 종료되지 않을 것이다. 이는 중첩 인터럽트를 허용하는 것과 비슷하지만 ISR 프롤로그와 에필로그를 다시 실행할 필요는 없기 때문에 더 나은 방법이다.

이 방법의 단점은 인터럽트 서비스가 시작되었다면, 완료될 때 까지 높은 우선순위의 인터럽트는 기다려야한다. 따라서, 우선순위에 관계없이, 임의의 인터럽트의 레이턴시는 가장 긴 인터럽트를 처리하는 데 걸리는 시간만큼 길 수 있다.

# Every Interrupt Vectors To A Unique Location
인터럽트 컨트롤러가 해당 인터럽트 핸들러로 벡터링하는 경우, 각 ISR은 'Typical μC/OS-III Interrupt Service Routine (ISR)'에 설명된 대로 작성되어야 한다. 이는 설계를 약간 복잡하게 만든다. 그러나 한 핸들러에서 다른 핸들러로 대부분의 코드를 복사해서 붙여넣을 수 있고, 실제 장치에 특정한 것만 변경할 수 있다.

인터럽트 컨트롤러가 사용자가 인터럽트 소스에 대해 질의하는 것을 허용한다면 모든 벡터가 동일한 위치를 가리키도록 설정하는 것만으로 모든 인터럽트가 동일한 위치로 벡터링하는 것을 시뮬레이션할 수 있다. 그러나 고유한 위치로 벡터를 변환하는 대부분의 인터럽트 컨트롤러는 모든 인터럽트 장치에 대해 고유한 벡터가 필요하지 않기 때문에 사용자가 인터럽트의 출처에 대해 질의하는 것을 허용하지 않는다.

# Direct And Deferred Post Methods
μC/OS-III는 인터럽트로부터의 이벤트 포스팅을 Direct와 Deferred Post의 두 가지 다른 방식을 사용하여 처리한다. 애플리케이션에서, OS_cfg.h에 있는 OS_CFG_ISR_POST_DEFERED_EN의 값을 변경하여 두 방식 중 하나를 선택한다. 0으로 설정하면 μC/OS-III은 Direct Post 방식을 사용하고 1로 설정하면 μC/OS-III은 Deferred Post 방식을 사용한다.

두 가지 방법 사이를 전환하기 위해 configuration 값 OS_CFG_ISR_POST_DEFERED_EN 이외의 것을 변경할 필요는 없다. 물론, configuration 상수를 변경하려면 μC/OS-III을 다시 컴파일해야 할 것이다.

## Direct Post Method
Direct Post Method는 μC/OS-II에서 사용되며, μC/OS-III에 복제되었다. Figure 9-3은 Direct Post에서 일어나는 일에 대한 task diagram을 보여준다.

![9-3 Direct Post Method](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/bd75a58f-8787-463d-87b4-5cbe61f16c13)


### F9-3(1)
장치가 인터럽트를 발생시킨다.

### F9-3(2)
해당 디바이스를 처리하는 인터럽트 서비스 루틴(ISR)이 실행된다(인터럽트가 활성화되어있다고 가정함). 장치 인터럽트는 일반적으로, 그 task가 대기하고 있는 이벤트이다. 이 인터럽트가 발생되기를 기다리는 task는 인터럽트때문에 중단된 task보다 높은 우선순위를 갖거나, 더 낮거나, 혹은 동등하다.

### F9-3(3)
만약 ISR이 같거나 더 낮은 우선순위의 task를 ready-to-run상태로 만들었다면, ISR이 완료되었을 때 μC/OS-III는 인터럽트때문에 중단된 task의 정확히 중단된 시점으로 되돌아간다.

### F9-3(4)
ISR이 더 높은 우선순위의 작업을 ready-to-run 상태로 만든 경우, μC/OS-III는 새로운, 더 높은 우선순위의 작업으로 문맥교환할 것인데, 이는 더 중요한 작업이 이 장치 인터럽트를 기다리고 있었기 때문이다.

### F9-3(5)
Direct Post 방식에서, μC/OS-III는 ISR이 임계 구역 중 일부를 접근할 수 있기 때문에 인터럽트를 비활성화함으로써 임계 구역들을 보호해야 한다.

## Direct Post Method (2)
위의 논의에서, 인터럽트가 활성화되어있고, ISR이 인터럽트 장치에 빠르게 응답할 수 있다고 가정하였다. 그러나 어플리케이션 코드가 μC/OS-III 서비스 호출을 하면(어느 시점에서 그럴 것이다) 인터럽트가 비활성화될 가능성이 있다. OS_CFG_ISR_POST_DEFERED_EN이 0으로 설정되면 μC/OS-III은 임계 구역에 접근하는 동안 인터럽트를 비활성화한다. 따라서 μC/OS-III이 인터럽트를 다시 활성화시킬 때까지 인터럽트에 대해 반응하지 않을 것이다.

Direct Post 방식을 사용할지 여부를 결정하는 핵심 요소는 일반적으로 μC/OS-III 인터럽트 비활성화 시간이다. 이는 사용되는 프로세서용 μC/OS-III 포트와 함께 제공되는 μC/CPU 파일이 최대 인터럽트 디스에이블 시간을 측정하기 위한 코드를 포함하기 때문에 인터럽트 비활성화 시간을 확인하기 상당히 용이하다. 이 코드는 테스트 목적으로 활성화될 수 있으며 제품을 배포할 준비가 되었을 때 제거될 수 있다.

다음과 같이 관련된 코드의 실행 시간을 더하여 interrupt latency, interrupt response, interrupt recovery 및 task latency를 계산할 수 있다.

```
Interrupt Latency = Maximum interrupt disable time;

Interrupt Response = Interrupt latency 
                    + Vectoring to the interrupt handler
                    + ISR prologue;

Interrupt Recovery = Handling of the interrupting device
                    + Posting a signal or a message to a task
                    + OSIntExit()
                    + OSIntCtxSw();

Task Latency = Interrupt response
                + Interrupt recovery
                + Time scheduler is locked;
```
μC/OS-III ISR 프롤로그, ISR 에필로그, OSIntExit(), 그리고 OSIntCtxSw()의 실행시간은 독립적으로 측정될 수 있으며, 일정해야한다.

post call의 실행 시간을 OS_TS_GET()을 이용하여 쉽게 측정할 수 있다.

Direct Post 방식에서는 타이머를 다룰 때만 스케줄러가 잠기므로, 짧은 콜백이 동시에 만료되는 타이머가 적으면 task latency가 빨라야한다(213페이지의 12장 "Timer Management"를 참조) μC/OS-III는 스케줄러가 잠겨있는 시간을 측정할 수 있어서, 이를 이용하여 task latency를 확인할 수 있다.

- 콜백 : 호출될 함수를 알려 주어 다른 프로그램 또는 다른 모듈에서 함수를 호출하게 하는 방법. 일반적으로 운영 체계(OS)가 호출할 애플리케이션의 함수를 지정해 특정한 사건 또는 메시지가 발생했을 때 호출되도록 지정할 수 있다. 이런 함수를 콜백 함수라고 한다.

## Deferred Post Method
Deferred Post 방식(OS_CFG_ISR_POST_DEFERRED_EN이 1로 설정되었을 때)에서, 임계 구역에 접근하기 위해 인터럽트를 비활성화하는 대신 μC/OS-III는 스케줄러를 잠근다. 이는 인터럽트가 발생 된 것을 알고 서비스되도록하는 동시에 다른 작업이 임계 구역에 접근하도록 하는 것을 방지한다. Deferred Post 방식에서 인터럽트는 거의 비활성화 되지 않는다. Deferred Post 방식은 Figure 9-4와 같이 좀 더 복잡하다.

![Figure 9-4 Deferred Post Method block diagram](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/0385b8c7-1314-499a-912f-972d340def05)


### F9-4(1)
장치가 인터럽트를 발생시킨다.

### F9-4(2)
해당 디바이스를 처리하는 인터럽트 서비스 루틴(ISR)이 실행된다(인터럽트가 활성화되어있다고 가정함). 장치 인터럽트는 일반적으로, 그 task가 대기하고 있는 이벤트이다. 이 인터럽트가 발생되기를 기다리는 task는 인터럽트때문에 중단된 task보다 높은 우선순위를 갖거나, 더 낮거나, 혹은 동등하다.

### F9-4(3)
ISR은 post 서비스 중 하나를 호출하여 신호를 보내거나 task에 메시지를 보낸다. 그러나 post 동작을 수행하는 대신, ISR은 인터럽트 큐(Interrupt Queue)라고 불리는 특별한 큐에 argument와 함께 포스트 콜을 넣는다. 그러면 ISR은 인터럽트 큐 핸들러 task(Interrupt Queue Handler Task)를 ready-to-run 상태로 만든다. 인터럽트 큐 핸들러 task는 μC/OS-III 내부에 있으며 항상 가장 높은 우선순위의 task(즉, 우선순위 0)이다.

### F9-4(4)
ISR이 끝나면, μC/OS-III는 항상 인터럽트 큐 핸들러 task로 문맥 교환하고, 이후 큐에서 post 명령을 추출한다. 이때 인터럽트를 비활성화하여 큐가 비워지는 동안 다른 인터럽트가 인터럽트 큐에 엑세스하는 것을 방지한다. 그런 다음 task는 인터럽트를 다시 활성화하고, 스케줄러를 잠그고, task-level에서 post call을 수행한다. 이는 task-level에서 임계 구역을 효과적으로 다루게한다.

### F9-4(5)
인터럽트 큐 핸들러 작업이 인터럽트 큐를 다 비우면, 스스로 ready-to-run 상태가 아닌 것으로 만들고, 스케줄러를 호출하여 어떤 task가 다음에 실행되어야 하는지 결정한다. 원래 인터럽트에 의해 중단된 task가 가장 높은 우선순위인 경우, μC/OS-III는 그 task를 재개할 것이다.

### F9-4(6)
만약 post 때문에 더 중요한 task가 ready-to-run 상태가 되었다면 μC/OS-III는 그 task로 문맥 교환할 것이다.

## Deferred Post Method(2)
모든 추가 과정은 코드의 임계구역에서 인터럽트가 비활성화되는 것을 피하기 위해 수행된다. 추가 과정에 소요되는 시간은 post call과 argument 들을 큐에 복사하고, 큐에서 꺼내고, 문맥교환하는 시간으로 구성된다.

Direct Post 방법과 유사하게, interrupt latency, interrupt response, interrupt recovery, task latency를 아래와 같이 계산할 수 있다.

```c
Interrupt Latency = Maximum interrupt disable time;

Interrupt Response = Interrupt latency
                    + Vectoring to the interrupt handler
                    + ISR prologue;

Interrupt Recovery = Handling of the interrupting device
                    + Posting a signal or a message to the Interrupt Queue
                    + OSIntExit()
                    + OSIntCtxSw() to Interrupt Queue Handler Task;

Task Latency = Interrupt response
                + Interrupt recovery
                + Re-issue the post to the object or task
                + Context switch to task
                + Time scheduler is locked;
```
μC/OS-III ISR 프롤로그, ISR 에필로그, OSIntExit(), OSIntCtxSw()의 실행시간은 독립적으로 측정될 수 있고, 일정해야한다.

post call의 실행시간을 OS_TS_GET()을 사용하여 쉽게 측정할 수 있다. post call에 소요되는 시간은 Deferred Post 방식에서 짧은데, post call과 argument들을 인터럽트 큐에 넣는 것이 전부이기 때문이다.

차이점은 Deferred Post 방식은 인터럽트가 매우 짧은 시간동안만 비활성화되므로, 처음 3개 측정값(interrupt latency, interrupt response interrupt recovery)은 굉장히 작아야한다. 그러나 task latency는 μC/OS-III가 임계 구역에 접근하기 위해 스케줄러를 잠글수록 값이 크다.

# Direct VS. Deferred Post Method
Direct Post 방식에서, μC/OS-III는 임계 구역에 접근하기 위해 인터럽트를 비활성화한다. 이와 비교하여, Deferred Post 방식에서는 μC/OS-III는 임계 구역에 접근하기 위해 스케줄러를 잠근다.

Deferred Post 방식에서, μC/OS-III는 인터럽트 큐에 접근하기 위해 인터럽트를 비활성화해야한다. 그러나 인터럽트 비활성화 시간은 매우 짧고 일정하다.

![Figure 9-5 Direct vs. Deferred Post Methods](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/51aad6f1-53fa-4801-8ec5-45f554b18a6d)

인터럽트 소스가 매우 빠르고, Direct Post 방식을 사용했을 때 발생하는 인터럽트 비활성화 시간을 수용할 수 없어 인터럽트 비활성화 시간이 중요한 응용 프로그램의 경우 Deferred Post 방식을 사용한다.

만약 표 9-1에 있는 기능을 쓸 계획인 경우, Deferred Post 방식을 고려해보라.

| Feature | Reason |
|---------|--------|
|동일한 우선순위를 가진 여러개의 task         | 이는 μC/OS-III의 중요한 기능임에도 불구하고, 동일한 우선순위에 있는 여러 task는 더 긴 임계구역을 생성한다. 그러나 동일한 우선순위의 task가 몇 개 정도만 있다면 interrupt latency는 상대적으로 적을 것이다. 사용자가 동일한 우선순위의 여러 task를 생성하지 않는 경우에는 Direct Post 방식을 추천한다.       |
| Event Flags(273쪽의 14장 "Synchronization" 참조)        |여러 개의 작업이 서로 다른 이벤트를 대기하고 있는 경우, 이벤트 대기 중인 모든 task를 수행하려면 상당한 처리 시간이 필요하므로, 임계 구역이 길어짐을 의미한다. Event Flag 그룹에서 몇 가지 작업(약 1~5개)만 대기 중인 경우, 임계 구역은 Direct Post 방식을 사용해도 충분히 짧다.        |
|여러 object에 대기(333쪽의 16장 "Pending On Multiple Objects 참조)         |여러 object에 대기하는 것은 μC/OS-III에서 제공하는 가장 복잡한 기능이고, Direct Post 방식을 사용하였을 때 인터럽트 비활성화 시간이 많이 길어진다. 만약 여러 object에 대기하고 있으면, Deferred Post 방식이 추천된다. 만약 이 기능을 사용하지 않을 경우, Direct Post 방식을 사용할 수도 있다.        |
|Broadcast on Post calls(OSSemPost() and OSQPost() 설명 참조)         |μC/OS-III는 broadcast에서 여러 task에 대한 post를 처리할 때 인터럽트를 비활성화한다. broadcast 옵션을 쓰지 않는 경우, Direct Post 방식을 사용한다. broadcast는 세마포어 및 메시지 큐에서만 적용된다        |

(표 9-1 Direct Post 방식을 쓸 때 피해야할 μC/OS-III 기능)

# The Clock Tick (Or System Tick)
μC/OS-III 기반 시스템은 일반적으로 clock tick 또는 system tick이라 불리는 주기적인 시간 소스를 요구한다.

10에서 1000Hz 사이의 빈도로 인터럽트를 생성하도록 구성된 하드웨어 타이머는 clock tick을 제공한다. 또한 tick 소스는 AC전력(통상적으로 50~60Hz)로 부터 인터럽트를 생성함으로써 얻을 수 있다. 실제로 전력의 제로 크로싱을 검출함으로써 100또는 120Hz를 쉽게 도출할 수 있다. 즉, 만일 양쪽의 주파수를 사용하는 영역에서 사용되어야 한다면, 사용자가 어떤 주파수를 사용할지를 지정하게 하거나, 또는 어느 주파수를 사용하는지를 자동으로 검출하도록 할 필요가 있다.

clock tick 인터럽트는 시스템의 심장박동으로 볼 수 있다. clock tick의 빈도는 애플리케이션마다 다르고, 시간 소스의 resolution(Hz)에 따라 다르다. 그러나 tick rate가 빠를 수록, 시스템에 부과되는 오버헤드가 커진다.

clock tick 인터럽트를 통해 μC/OS-III는 clock tick 단위로 task를 지연시킬 수 있고, task가 이벤트를 기다리고 있을 때, 타임아웃 여부를 확인할 수 있다.

clcok tick 인터럽트는 OSTimeTick()을 호출해야한다. OSTimeTick()의 의사코드는 L9-5와 같다.

```c
void OSTimeTick (void)
{
    OSTimeTickHook(); (1)
#if OS_CFG_ISR_POST_DEFERRED_EN > 0u
    Get timestamp; (2)
    Post "time tick" to the Interrupt Queue;
#else
    Signal the Tick Task; (3)
    Run the round-robin scheduling algorithm; (4)
    Signal the timer task; (5)
#endif
}
//L9-5
```

## L9-5(1)
time tick ISR은 hook 함수인 OSTimeTickHook()을 호출함으로써 시작된다. hook 함수는 틱 인터럽트가 발생할 때 추가적인 처리를 수행할 수 있게 한다. tick hook은 OS_AppTimeTickHookPtr이 NULL이 아닌 경우 사용자 정의 tick hook을 호출할 수 있다. hook이 먼저 호출되는 이유는 애플리케이션이 이 주기적인 시간 소스에 즉시 접근할 수 있도록 하기 위함이다. 이는 일정한 간격으로 센서를 읽거나, PWM(Pulse Width Modulation) 레지스터 업데이트 등에 유용할 수 있다.

## L9-5(2)
만약 μC/OS-III가 Deferred Post 방식으로 설정되어있다면, μC/OS-III는 현재 타임스탬프를 확인하고 인터럽트 큐에 tick task를 넣고, tick task call을 연기한다. 따라서 tick task는 인터럽트 큐 핸들러 task에 의해 신호를 받을 것이다.

## L9-5(3)
만약 μC/OS-III가 Direct Post 방식으로 설정되어있다면, μC/OS-III는 tick task에 신호를 보내고, 따라서 delay와 타임아웃을 처리할 수 있다.

## L9-5(4)
μC/OS-III는 라운드 로빈 스케줄링을 실행하여 현재 task의 시간 할당이 끝났는지 확인한다.

## L9-5(5)
tick task는 timer의 시간 소스로 쓰인다(231쪽의 13장 "Resource Management" 참조)

# The Clock Tick (Or System Tick)(2)
일반적으로 많이 오해하는 것이 μC/OS-III는 system tick이 항상 필요하다는 것이다. 사실, 많은 저전력 애플리케이션들은 tick list를 유지하는데 필용한 전력 때문에, system tick을 구현하지 않을 수 있다. μC/OS-III preemptive 커널이기 때문에, 틱 인터럽트 이외의 이벤트는 키패드에서 키 누름 등으로 저전력 모드에 놓인 시스템을 꺠울 수 있다. system tick이 없다는 것은 사용자가 delay나 타임아웃을 사용하는 것을 허용하지 않는다는 것을 의미한다. 이는 저전력 제품의 설계자에 의해 결정된다.

# Summary
μC/OS-III는 인터럽트를 관리하는 서비스를 제공한다. ISR은 짧아야 하며, 해당 인터럽트 장치 서비스를 담당하는 task에 신호를 보내거나 메시지를 보내야 한다.

task에 신호를 보내거나 메시지를 보낼 필요가 없는 ISR은 그렇게 할 필요가 없다. 즉 μC/OS-III는 커널이 인식하지 못하는 ISR을 허용한다.

μC/OS-III는 모든 인터럽트 장치에 대해 단일 ISR로, 또는 각 장치에 대해 고유한 ISR로 벡터링하는 프로세서를 지원한다.

μC/OS-III는 두 가지 방식을 지원한다: Direc Post 방식은 μC/OS-III 임계 구역들이 인터럽트를 비활성화함으로써 보호된다. Deferred Post 방식은 μC/OS-III가 코드의 임계 구역들에 접근할 때 스케줄러를 잠근다. 사용되는 방식은 task response뿐만 아니라 interrupt response에 크게 의존한다.

μC/OS-III에서 delay와 타임아웃이 필요한 애플리케이션를 쓰기 위해서는 주기적인 시간소스가 있어야한다.

# Reference
 - uC/OS-III: The Real-Time Kernel For the STM32 ARM Cortex-M3, Jean J. Labrosse, Micrium, 2009

 - https://en.wikipedia.org/wiki/Interrupt_latency
 - https://terms.tta.or.kr/dictionary/dictionaryView.do?word_seq=037861-1

[책 링크](https://micrium.atlassian.net/wiki/spaces/osiiidoc/overview)
