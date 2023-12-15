---
title: "μC/OS-III ch.8 Context Switching"
last_modified_at: 2023-10-11T21:53:12+09:00
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

μC/OS-III가 다른 task를 실행하기로 결정하면(151페이지의 7장 "Scheduling" 참조), 보통 CPU 레지스터로 구성되는 현재 task의 문맥(context)를 현재 task의 스택에 저장하고 새로운 task의 문맥을 복원하여 해당 task의 실행을 재개한다. 이 프로세스를 문맥 교환(context switch)라고 한다.

문맥교환은 오버헤드를 추가한다. CPU의 레지스터가 많을수록 오버헤드가 증가한다. 문맥교환을 수행하는 데 필요한 시간은 일반적으로 CPU가 저장하고 복원해야 하는 레지스터의 수에 따라 결정된다.

컨텍스트 스위치 코드는 일반적으로 프로세서의 μC/OS-III 포트의 일부이다. 포트는 μC/OS-III을 원하는 프로세서에 적응시키는 데 필요한 코드이다. 이 코드는 특수 C 및 어셈블리 언어 파일, 즉 os_cpu.h, os_cpu_c.c 및 os_cpu_a.asm에 있다. 355페이지의 제 18장 "Porting μC/OS-III" 은 각기 다른 CPU 아키텍처에 μC/OS-III를 포팅하는 데 필요한 것에 대한 더 자세한 사항을 제공한다.

이 장에서 우리는 Figure 8-1과 같은 가상의 CPU를 사용한 문맥교환 과정을 일반적인 용어로 논의할 것이다. 우리의 가상의 CPU는 16개의 정수 레지스터(R0~R15), 별도의 ISR 스택 포인터, 별도의 상태 레지스터(SR)를 포함한다. 모든 레지스터는 32비트 너비이며 16개의 정수 레지스터 각각은 데이터나 주소를 저장할 수 있다. 프로그램 카운터(또는 명령 포인터)는 R15이고 R14와 R14'라는 레이블이 붙은 두 개의 개별 스택 포인터가 있다. R14는 task 스택 포인터(TSP)를 나타내고, R14'는 ISR 스택 포인터(ISP)를 나타낸다. CPU는 예외나 인터럽트를 서비스할 때 자동으로 ISR 스택으로 전환한다. task 스택은 ISR에서 액세스 가능하며(즉, ISR에 있을 때 task 스택에 요소를 푸시 앤 팝할 수 있다), 인터럽트 스택도 작업에서 액세스 가능하다.

![Figure 8-1 Fictitious CPU](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/e5a2b6c8-8e49-4a8e-958c-5050ee3c12ff)

μC/OS-III에서 준비된 task를 위한 스택 프레임은 항상 인터럽트가 방금 발생한 것처럼 설정되어 있고 모든 프로세서 레지스터가 저장되어 있다. task는 생성 시 준비 상태로 진입하므로 비슷한 방식으로 그 task의 스택 프레임이 소프트웨어에 의해 미리 초기화된다. 가상의 CPU를 사용하여 우리는 복원 준비가 완료된 task를 위한 스택 프레임이 Figure 8-2와 같다고 가정할 것이다.

task 스택 포인터는 task의 스택에 저장된 마지막 레지스터를 가리킨다. 프로그램 카운터(PC 또는 R15)와 상태 레지스터(SR)는 스택에 저장된 첫 번째 레지스터들이다. 실제로 이들은 다른 레지스터들이 예외 핸들러 코드에 의해 스택에 푸시되는 동안 예외 또는 인터럽트가 발생(인터럽트가 활성화된 것으로 가정)할 때 CPU에 의해 자동으로 저장된다. 스택 포인터(SP 또는 R14)는 실제로 스택에 저장되지 않고 대신 task의 OS_TCB에 저장된다.

인터럽트 스택 포인터는 다른 메모리 영역인 인터럽트 스택의 현재 상단을 가리킨다. ISR이 실행되면, 프로세서는 함수 호출 및 로컬 argument를 위한 스택 포인터로 R14'를 사용한다.

![Figure 8-2 CPU register stacking order of ready task](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/1764dc50-3f76-4cdd-8a4f-7622d073419b)

문맥교환에는 task로부터 수행되는 것과 ISR로부터 수행되는 것의 두 가지 타입이 있다. task level 문맥교환은 매크로 OS_TASK_SW()에 의해 실제로 호출되는 OSCtxSw()의 코드에 의해 구현된다. 소프트웨어 인터럽트, 트랩 명령어들, 또는 단순히 함수를 호출하는 것과 같은 OSCtxSw()를 호출하는 많은 방법들이 있기 때문에 매크로가 사용된다.

ISR 문맥교환은 OSIntCtxSw()에 의해 구현된다. 두 함수에 대한 코드는 일반적으로 어셈블리 언어로 작성되며 os_cpu_a.asm이라는 파일에서 찾을 수 있다.

# OSCtxSw()
task level 스케줄러(OSSched())가 새로운 높은 우선순위의 task를 실행할 필요가 있다고 판단할 때 OSCtxSw()(os_cpu_a.asm 참조)가 호출된다. Figure 8-3은 OSCtxSw()를 호출하기 직전의 여러 μC/OS-III 변수들과 데이터 구조들의 상태를 보여준다.

![Fig 8-3 Variables and data structures prior to calling OSCtxSw()](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/87410f3e-311b-413b-81fd-9690a129d10e)

## F8-3(1)
OSTCBCurPtr은 현재 실행 중인 task의 OS_TCB를 가리킨다. 현재 실행 중인 task가 OSSched()를 호출한다.

## F8-3(2)
OSSched()는 OSTCBHighRdyPtr이 해당 OS_TCB를 가리키게 함으로써 실행할 새 task를 찾는다.

## F8-3(3)
OSTCBHighRdyPtr->StkPtr이 실행할 새로운 task의 스택의 top을 가리키고 있다.

## F8-3(4)
μC/OS-III는 task를 생성하거나 일시 중단할 때 항상 스택 프레임을 인터럽트가 발생한 것처럼 보이게 하고 모든 레지스터가 스택 프레임에 저장된다. 이것은 task의 예상되는 상태를 나타내며,작업을 다시 시작할 수 있도록한다.

## F8-3(5)
CPU의 스택 포인터는 OSSched()를 호출한 task의 스택 영역(즉, RAM)을 가리킨다. OSCtxSw()가 호출되는 방식에 따라, 스택 포인터는 OSCtxSw()의 반환 주소를 가리키고 있을 수 있다.

# OSCtxSw()-part2
그림 8-4는 OSctxSw()에 의해 구현된 문맥교환을 수행하는 데 관련된 단계를 보여준다.

![Operation performed by OSCtxSw()](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/7be17c66-81f6-47b4-90b2-ac8866d4b7d1)

## F8-4(1)
OSCtxSw()는 현재 task의 상태 레지스터와 프로그램 카운터를 현재 task의 스택에 저장하는 것으로 시작한다. 레지스터의 저장 순서는 인터럽트가 발생할 때 CPU가 스택 프레임에 있는 레지스터들(의 순서를)을 어떻게 생각하느냐에 달려 있다. 이 경우 SR이 먼저 스택에 저장된 것으로 가정한다. 나머지 레지스터들은 이후 스택에 저장된다.

## F8-4(2)
OSCtxSw()는 CPU의 스택 포인터의 내용을 문맥교환되어 나가는 task의 OS_TCB에 저장한다. 즉, OSTCBCurPtr->StkPtr = R14이다.

## F8-4(3)
그리고 OSCtxSw()는 CPU 스택 포인터에 새로운 task의 스택의 top을 저장한다(스택의 top의 위치는 그 task의 OS_TCB에 있다). 즉, R14=OSTCBHighRdyPtr->StkPtr.

## F8-4(4)
마지막으로, OSCtxSw()는 새 스택으로부터 CPU 레지스터를 복원한다. 프로그램 카운터와 상태 레지스터는 일반적으로 인터럽트 명령으로부터 복귀하면서 동시에 복원된다.

# OSIntCtxSw()
OSIntCtxSw()(os_cpu_a.asm 참조)는 ISR level 스케줄러(OSIntExit())가 새로운 높은 우선순위의 task를 실행할 준비가 되었다고 판단할 때 호출된다. Figure 8-5는 OSIntCtxSw()를 호출하기 직전의 여러 μC/OS-III 변수들과 자료구조들의 상태를 보여준다.

![Variables and data structures prior tocalling OSIntCtxSw()](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/0099928f-a391-447b-9349-c194a5397569)

μC/OS-III는 CPU 레지스터가 ISR을 시작할 때 task의 스택에 저장된다고 가정한다(175페이지의 9장 "Interrupt Management" 참조). 따라서 OSTCBCurPtr->StkPtr에는 일시중지 될 예정의 task의 stack의 top(그림에서 왼쪽)이 포함되어 있다. OSIntCtxSw()는 일시중지 된 task의 CPU 레지스터를 저장할 필요가 없는데, 이미 저장되어있기 때문이다.

FIg 8-6은 OSIntCtxSw()가 문맥교환의 후반부를 완료하기 위해 수행하는 동작을 나타낸 것이다. 이는 OSCtxSw()의 후반부와 정확히 동일한 과정이다.

![Fig 8-6 Operations performed by OSIntCtxSw](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/4cce2fb2-b816-4915-b8dd-d6b0a6537bae)

## F8-6(1)
OSINtCtxSw()는 CPu 스택 포인터에 스택의 top을 저장한다(스택의 top은 그 task의 OS_TCB에 있다). R14=OSTCBHighRdyPtr->StkPtr.

## F8-6(2)
그런 다음 OSIntCtxSw()는 새 task의 스택으로 부터 CPU 레지스터를 복원한다. 프로그램 카운터와 상태 레지스터는 일반적으로 인터럽트 명령어로부터 복귀하면서 동시에 복원된다.

# Summary
문맥교환은 task와 관련된 문맥(즉, CPU 레지스터)를 저장하고, 새로운, 높은 우선순위 task의 문맥을 복원하는 것으로 구성된다.

(새로 전환할) task는 task-level 코드에 의해 문맥교환이 시작되면 OSSched(), ISR에 의해 시작되면 OSIntExit()에 의해 전환된다.

OSCtxSw()는 OSSched()에 대해 문맥교환을 수행하고, OSIntCtxSw()는 OSIntExit()에 대해 문맥교환을 수행한다. 단, OSIntCtxSw()는 ISR 진입시 CPU 레지스터를 저장했다고 가정하기 때문에 문맥교환의 후반부만 수행하면 된다.
# Reference
 - uC/OS-III: The Real-Time Kernel For the STM32 ARM Cortex-M3, Jean J. Labrosse, Micrium, 2009

[책 링크](https://micrium.atlassian.net/wiki/spaces/osiiidoc/overview)
