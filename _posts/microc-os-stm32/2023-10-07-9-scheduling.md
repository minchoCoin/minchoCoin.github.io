---
title: "μC/OS-III ch.7 Scheduling"
last_modified_at: 2023-10-07T19:53:12+09:00
categories:
    - microc-os-stm32
tags:
    - microc-os
    - embedded-system

toc: true
toc_label: "My Table of Contents"
author_profile: true

---
dispatcher 라고 불리는 scheduler는 μC/OS-III의 한 부분으로, 어떤 task가 다음에 실행될지 결정하는 역할을 한다. μC/OS-III은 우선순위 기반의 선점적(preemptive)커널이다. 앞에서 살펴본 바와 같이 각 task에는 중요도에 따라 우선순위가 부여된다. 각 작업의 우선순위는 응용 프로그램에 따라 달라지며, μC/OS-III은 동일한 우선순위를 가진 여러 작업을 허용한다.

선점적(preemptive)이라는 말의 의미는 event가 발생했을 때, 그리고 그 event가 더 중요한 task를 ready-to-run 상태로 만든다면 μC/OS-III는 즉시 그 task에 CPU를 부여할 것이라는 것을 의미한다. 따라서 task가 더 높은 우선 순위의 task에 신호를 보내면 현재 task는 중단되고 더 높은 우선순위의 task는 CPU를 제어할 수 있게 된다. 마찬가지로, Interrupt Service Roution(ISR)이 더 높은 우선순위의 task에 신호를 보내거나 매시지가 전송되었을 때, 중단된 task는 중단된 상태로 유지되고 새로운 더 높은 우선순위의 task가 실행된다.

# 7-1 Preemptive scheduling
μC/OS-III는 인터럽트로부터의 이벤트 포스팅을 Direct와 Deferred 포스트의 두가지 다른 방법을 사용하여 처리한다. 이들에 대해서는 175페이지의 9장 '인터럽트 관리'에서 보다 상세하게 논의될 것이다. 스케쥴링 관점에서 보면 ,두 방법의 최종 결과는 동일하다. ready 상태의 task 중 가장 높은 우선순위는 rmfla 7-1과 7-2처럼 CPU를 할당받을 것이다.

![Preemptive scheduling - direct method](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/3cf3b459-fd33-416a-a2c9-ade22470d0b9)

## F7-1(1)
낮은 우선순위의 task가 실행 중이다. 이때 인터럽트가 발생한다.

## F7-1(2)
인터럽트가 활성화되어있으면, CPU는 인터럽트 신호를 발생시킨 장치에 해당하는 ISR로 점프한다.

## F7-1(3)
ISR은 장치에 서비스를 제공하고, ISR은 장치에서 발생한 인터럽트를 처리하기 위해 해당 장치에 서비스를 제공하기 위해 대기 중인 더 높은 우선순위의 task에 신호를 보내거나 메시지를 보낸다. 그렇게 되면, 그 task는 ready-to-run 상태가 된다.

## F7-1(4)
ISR의 작업이 끝나면, μC/OS-III를 호출한다.

## F7-1(5)(6)
더 중요한 ready-to-run 상태의 task가 있기 때문에, μC/OS-III은 중단된 task로 돌아가지 않고 더 중요한 task로 전환한다. 이것이 어떻게 작동하는지에 대한 자세한 내용은 165페이지의 8장 "context switching"을 참조하라.

## F7-1(7)(8)
우선 순위가 높은 task는 인터럽트를 발생시킨 장치에 서비스를 제공하고, 완료되면, μC/OS-III를 호출하여 그 장치에서 발생하는 다른 인터럽트를 기다리도록 요청한다.

## F7-1(9)(10)
ISR이 신호를 보낸 그 '우선순위가 높은 task'는 장치가 다시 서비스를 받을 필요가 있을 때까지 μC/OS-III에 의해 차단된다. 장치가 두번째로 인터럽트를 발생시키지 않았기 때문에 μC/OS-III는 원래 task(중단되었던 task)로 다시 전환한다.

## F7-1(11)
중단되었던 task는 정확히 중단된 지점부터 재개된다.


![Preemptive scheduling - Deferred Post Method](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/866fa347-cb3f-49e2-b369-b1ffda3f5b92)

## F7-2(1)
ISR은 장치에 서비스를 제공하고, 메시지를 전송하거나 신호를 보내는 대신, μC/OS-III는 POST 호출을 통해 post call을 특별 큐에 넣고 ISR Handler Task(가장 높은 우선순위를 가짐)를 ready-to-run 상태로 만든다.

## F7-2(2)
ISR이 작업을 다 끝내면, μC/OS-III를 호출한다.

## F7-2(3)(4)
ISR이 ISR Handler Task를 ready-to-run 상태로 만들었기 때문에 μC/OS-III는 ISR Handler Task를 실행한다.

## F7-2(5)(6)
그런 다음 ISR Handler Task는 message queue에서 post call을 제거하고 post 관련 함수를 호출한다. 이번에는 ISR 레벨 대신 task 레벨에서 이를 수행한다. 이러한 추가 단계를 거치는 이유는 인터럽트 비활성화 시간을 가능한 적게 유지하기 위함이다. 해당 주제에 대한 자세한 내용은 175페이지의 9장 "Interrupt Managemnt"를 참고하라. queue가 비면 μC/OS-III는 ready list에서 ISR Handler Task를 제거하고 중단되었던 task로 전환한다.

# 7-2 Scheduling points
scheduling은 스케줄링 포인트들에서 발생하며, scheduling은 아래의 조건들에 따라 자동적으로 발생하기 때문에 애플리케이션 코드에서 특별한 것을 수행할 필요가 없다.

## task가 다른 task로 메시지나 신호를 보냄
task가 post 서비스 중 하나인 OS???Post()를 호출하여 신호를 보내거나 메시지를 보낼 떄 발생한다. Scheduling은 OS???Post() 끝부분에서 발생한다. OS_OPT_POST_NO_SCHED를 설정하면 스케쥴러를 호출하지않고, Scheduling이 발생되지 않는다.

## task가 OSTimeDly()나 OSTimeDlyHMSM()를 호출했을 때
지연시간이 0이아니면 task가 시간이 만료될때까지 list에서 기다리기 때문에 항상 scheduling이 발생한다. task가 wait list에 들어가자마자 scheduling이 발생하며, delay 함수를 호출한 task와 우선순위가 같거나 낮은 task로 context switch 된다.

## task가 이벤트가 발생되기를 기다릴 때(그리고 그 이벤트가 발생되지 않았을 때)
OS???Pend()함수 중 하나가 호출될 때이다. task는 event를 기다리는 wait list에 들어가고, 0이 아닌 timeout이 설정되면, timeout을 기다리는 task list에 들어간다. 그런 다음 scheduler가 실행 가능한 가장 중요한 task를 선택하기 위해 호출된다.

## task가 pend를 중단했을 때
task는 다른 task를 기다리는 것을 중단할 수 있다. 이는 OS???PendAbort()를 호출했을 때 일어난다. scheduling은 특정 kernel object wait list에서 task가 제거되었을 때 일어난다.

## task가 만들어졌을 때
어떤 task가 자기자신보다 더 높은 우선순위로 task를 만들었다면 스케쥴러가 호출된다.

## task가 삭제될때
현재 task가 종료되고 삭제되면, 스케쥴러가 호출된다.

## kernel object가 삭제될때
event flag group, semaphore, message queue, mutual exclusion semaphore가 삭제되면, 해당 커널 객체를 기다리고 있는 task는 ready-to-run 상태가 된다. 그리고 스케쥴러는 kernel object를 삭제한 task보다 더 높은 우선순위를 가진 task가 있는지 확인하기 위해 호출된다. task들은 kernel object가 삭제되었다는 것을 알게될 것이다.

## task가 자기자신이나 다른 task의 우선순위를 바꾸었을 때
task는 자기자신이나 다른 task의 우선순위를 변경하고, 해당 task의 새로운 우선순위가 우선순위를 바꾸는 작업을 한 task의 우선순위보다 높을때 스케쥴러가 호출된다.

## task가 OSTaskSuspend()를 호출하여 스스로 중지했을 때
OSTaskSuspend()를 호출한 task는 더이상 실행될 수 없기 때문에 스케쥴러가 호출된다. 중단된 task는 다른 task에 의해 다시 시작되어야 한다.

## task가 OSTaskSuspend()를 통해 중단된 task를 재개했을 때
OSTaskResume()을 호출하는 task보다 다시 시작된 task의 우선순위가 더 높으면 스케쥴러가 호출된다.

## 모든 중첩 ISR이 끝났을 때
모든 중첩된 ISR의 끝에서 스케쥴러가 호출되어 ISR에 의해 더 중요한 task가 ready-to-run 상태로 만들어졌는지 확인한다. 스케쥴링은 OSSched() 대신에 OSIntExit()에 의해 수행된다.

## OSSchcedUnlock()에 의해 스케쥴러가 풀렸을 때
스케쥴러가 잠긴 후에 풀릴 수 있다. OSSchedLock()을 통해 스케쥴러를 잠글 수 있다. 스케쥴러를 잠그는 것은 중첩될 수 있고, 스케쥴러가 잠긴 횟수만큼 풀려야한다.

## OSSchedroundRobinYield()함수를 호출하여 task가 자신에게 할당된 시간을 포기했을 때
해당 task가 동일한 우선순위를 가진 task 사이에서 실행되고, 해당 task가 자신에게 할당된 시간을 포기하고 다른 task에게 넘길때 발생한다.

## OSSched()를 호출하였을 때
응용 프로그램 코드가 OSSched()를 호출하여 스케쥴러를 동작시킬 수 있다.
OS???Post() 함수를 호출하고 OS_OPT_POST_NO_SCHED를 지정하여 스케쥴러를 매번 실행시키지 않고 여러 post를 한꺼번에 수행할 수 있다(물론 위의 상황에서 마지막 post가 OS_OPT_POST_NO_SCHED 옵션이 없는 post일 수 있다).

# Round-Robin Scheduling

# Reference
 - uC/OS-III: The Real-Time Kernel For the STM32 ARM Cortex-M3, Jean J. Labrosse, Micrium, 2009

[책 링크](https://micrium.atlassian.net/wiki/spaces/osiiidoc/overview)
