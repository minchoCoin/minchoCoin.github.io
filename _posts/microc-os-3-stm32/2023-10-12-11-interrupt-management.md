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
인터럽트 컨트롤러가 해당 인터럽트 핸들러로 벡터링하는 경우, 각 ISR은 'Typical μC/OS-III Interrupt Service Routine (ISR)'에 설명된 대로 작성되어야 한다. 이는 설계를 약간 복잡헤가 만든다. 그러나 한 핸들러에서 다른 핸들러로 대부분의 코드를 복사해서 붙여넣을 수 있고, 실제 장치에 특정한 것만 변경할 수 있다.

인터럽트 컨트롤러가 사용자가 인터럽트 소스에 대해 질의하는 것을 허용한다면 모든 벡터가 동일한 위치를 가리키도록 설정하는 것만으로 모든 인터럽트가 동일한 위치로 벡터링하는 것을 시뮬레이션할 수 있다. 그러나 고유한 위치로 벡터를 변환하는 대부분의 인터럽트 컨트롤러는 모든 인터럽트 장치에 대해 고유한 벡터가 필요하지 않기 때문에 사용자가 인터럽트의 출처에 대해 질의하는 것을 허용하지 않는다.
# Reference
 - uC/OS-III: The Real-Time Kernel For the STM32 ARM Cortex-M3, Jean J. Labrosse, Micrium, 2009

 - https://en.wikipedia.org/wiki/Interrupt_latency

[책 링크](https://micrium.atlassian.net/wiki/spaces/osiiidoc/overview)
