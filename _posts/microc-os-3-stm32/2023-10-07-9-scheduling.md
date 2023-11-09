---
title: "μC/OS-III ch.7 Scheduling"
last_modified_at: 2023-10-07T19:53:12+09:00
categories:
    - microc-os-3-stm32
tags:
    - microc-os
    - embedded-system

toc: true
toc_label: "My Table of Contents"
author_profile: true

---
dispatcher 라고 불리는 scheduler는 μC/OS-III의 한 부분으로, 어떤 task가 다음에 실행될지 결정하는 역할을 한다. μC/OS-III은 우선순위 기반의 선점형(preemptive)커널이다. 앞에서 살펴본 바와 같이 각 task에는 중요도에 따라 우선순위가 부여된다. 각 작업의 우선순위는 응용 프로그램에 따라 달라지며, μC/OS-III은 동일한 우선순위를 가진 여러 작업을 허용한다.

선점형(preemptive)이라는 말의 의미는 event가 발생했을 때, 그리고 그 event가 더 중요한 task를 ready-to-run 상태로 만든다면 μC/OS-III는 즉시 그 task에 CPU를 부여할 것이라는 것을 의미한다. 따라서 task가 더 높은 우선 순위의 task에 신호를 보내면 현재 task는 중단되고 더 높은 우선순위의 task는 CPU를 제어할 수 있게 된다. 마찬가지로, Interrupt Service Roution(ISR)이 더 높은 우선순위의 task에 신호를 보내거나 매시지가 전송되었을 때, 중단된 task는 중단된 상태로 유지되고 새로운 더 높은 우선순위의 task가 실행된다.

# 7-1 Preemptive scheduling
μC/OS-III는 인터럽트로부터의 이벤트 포스팅을 Direct와 Deferred 포스트의 두가지 다른 방법을 사용하여 처리한다. 이들에 대해서는 175페이지의 9장 '인터럽트 관리'에서 보다 상세하게 논의될 것이다. 스케쥴링 관점에서 보면 ,두 방법의 최종 결과는 동일하다. ready 상태의 task 중 가장 높은 우선순위는 Figure 7-1과 7-2처럼 CPU를 할당받을 것이다.

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
그런 다음 ISR Handler Task는 message queue에서 post call을 제거하고 post 관련 함수를 호출한다. 이번에는 ISR 레벨 대신 task 레벨에서 이를 수행한다. 이러한 추가 단계를 거치는 이유는 인터럽트 비활성화 시간을 가능한 적게 유지하기 위함이다. 해당 주제에 대한 자세한 내용은 175페이지의 9장 "Interrupt Management"를 참고하라. queue가 비면 μC/OS-III는 ready list에서 ISR Handler Task를 제거하고 신호나 메시지를 받은 task를 실행시킨다(즉, 그림에서 (6)부분은 잘못되었는데, ISR handler에서 μC/OS-III로 갔다가 High Priority Task로 넘어가야한다).

# 7-2 Scheduling points
scheduling은 스케줄링 포인트들에서 발생하며, scheduling은 아래의 조건들에 따라 자동적으로 발생하기 때문에 애플리케이션 코드에서 특별한 것을 수행할 필요가 없다.

## task가 다른 task로 메시지나 신호를 보냄
task가 post 서비스 중 하나인 OS???Post()를 호출하여 신호를 보내거나 메시지를 보낼 떄 발생한다. Scheduling은 OS???Post() 끝부분에서 발생한다. OS_OPT_POST_NO_SCHED를 설정하면 스케쥴러를 호출하지않고, Scheduling이 발생되지 않는다.

```c
//if opt==OS_OPT_POST_NO_SCHED then don't call OSSched()
void OSMutexPost (OS_MUTEX *p_mutex,OS_OPT opt,OS_ERR *p_err)
{
    OS_Post((OS_PEND_OBJ *)((void *)p_mutex),
            (OS_TCB *)p_tcb,
            (void *)0),
    if ((opt & OS_OPT_POST_NO_SCHED) == (OS_OPT)0) {
        OSSched(); }
    *p_err = OS_ERR_NONE;
}
```

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
둘 이상의 작업의 우선순위가 같을 때, μC/OS-III는 하나의 task를 미리 정한 시간(Time Quanta)동안 실행한다(그리도 다음 task를 또 time quanta 동안 실행한다). 이 과정을 Round-Robin Scheduling 또는 Time Slicing이라 한다. 만약 task가 주어진 time quanta를 다 사용할 필요가 없다면 자발적으로 CPU를 방출(give up)하여 다음 작업을 실행할 수 있도록 한다. 이를 양보(Yielding)이라 한다. μC/OS-III은 사용자가 런타임에 라운드 로빈 스케줄링을 활성화 또는 비활성화할 수 있도록 한다.

Fig 7-3은 동일한 우선순위로 실행되는 task들을 갖는 타이밍 다이어그램을 보여준다. 우선순위 'X'에서 ready-to-run 상태의 task는 3개가 있다. 예시를 위해 time quanta는 4 clock tick이라 하자. 이는 어두운 tick mark로 표시된다.

![Round Robin Scheduling](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/69699f82-7ffa-461b-adba-cbdffbd30858)

## F7-3(1)
Task 3이 실행 중이다. 이 시간 동안, tick interrupt가 발생하지만 time quanta가 아직 만료되지는 않았다.

## F7-3(2)
4번째 tick interrupt에서, time quanta가 만료된다.

## F7-3(3)
task 1이 ready-to-run 상태의 우선순위 'X'의 task list의 다음 task이기 때문에 μC/OS-III는 task 1을 재개한다.

## F7-3(4)
task 1이 time quanta가 만료(즉, 4틱 후)될 때 까지 실행된다.

## F7-3(5)(6)(7)
task 3이 실행되지만 OSSchedRoundRobinYield()를 호출하여 time quanta를 포기한다. 이로 인해, 우선순위 'X'의 ready-to-run 상태의 task list에서 다음 task를 실행된다. μC/OS-III가 Task #1을 실행하려고 할때, time quanta를 3틱으로 리셋하여, 다음 time quanta가 이 시점부터 3틱 후에 만료되도록 한다.

## F7-3(8)
task 1이 time quanta 동안 실행된다.

# Round-Robin Scheduling(2)
μC/OS-III는 OSSchedRoundRobinCfg() 함수를 통해 런타임에 기본 time quanta를 변경할 수 있다(443페이지 부록 A, "μC/OS-III API Reference" 참조). 또한 round robin scheduling 을 활성화/비활성화하고 기본 time qunata를 변경할 수 있다.

μC/OS-III는 또한 사용자가 task마다 time quanta를 지정하는 것을 가능하게 한다. 어떤 task는 1 틱, 다른 task는 12, 또 다른 task는 3, 그리고 또 다른 task는 7 등의 time quanta를 가질 수 있다. task의 time quanta는 task가 생성될 때 지정된다. task의 time quanta는 또한 함수 OSTaskTimeQuantaSet()을 통해 런타임에 변경될 수 있다.

# Scheduling internals
스케줄링은 OSSched()와 OSIIntExit()의 두 가지 함수에 의해 수행된다. OSSched()는 task 코드에 의해 호출되고 OSIIntExit()은 ISR에 의해 호출된다. 두 함수 모두 os_core.c에 있다.

Fig 7-1은 스케줄러가 사용하는 두 가지 자료구조를 보여준다. priority ready bitmap과 ready list(Chapter 6, "The Ready List"에 설명됨)이다.

![Priority ready bitmap and Ready list](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/a526bfca-e2df-4242-81a6-7af3ac7f62bc)

## 7-4-1 OSSched()

task level에서 호출되는 스케쥴러인 OSSched(os_core.c 참조)의 의사코드가 L7-1에 있다.
```c
void OSSched (void)
{
    Disable interrupts;
    if (OSIntNestingCtr > 0) { (1)
        return;
    }
    if (OSSchedLockNestingCtr > 0) { (2)
        return;
    }
    Get highest priority ready; (3)
    Get pointer to OS_TCB of next highest priority task; (4)
    if (OSTCBNHighRdyPtr != OSTCBCurPtr) { (5)
        Perform task level context switch;
    }
    Enable interrupts;
}
//L7-1 OSSched() pseudocode
```
### L7-1(1)
OSSched()는 task level 스케줄러이므로, ISR에서 호출되지 않는지 확인하는 것으로 시작한다. ISR은 OSSched() 대신에 OSIntExit()을 호출해야한다. 만약 OSSched()가 ISR에서 호출되면 OSSched()는 단순히 return 한다.

### L7-1(2)
다음 단계는 스케줄러가 잠겨 있지 않은지 확인하는 것입니다. 코드가 OSSchedLock()를 호출하면 스케줄러를 잠그고, OSSched()는 단순히 return 한다.

### L7-1(3)
OSSched()는 141페이지의 6장 'The Ready List'에서 설명된 바와 같이 bitmap OSPrioTbl을 스캔함으로서, ready 상태의 가장 높은 우선순위의 task를 확인한다.

### L7-1(4)
어떤 우선순위의 task가 준비되었는지 알게 되면, 우선순위를 OSRdyList[]의 인덱스로 사용하고, OSRdyList[우선순위].HeadPtr에 있는, 즉 list의 맨 앞에 있는 OS_TCB를 추출한다. 이 시점에서 어느 OS_TCB로 전환할지, 그리고 어떤 task를 'OSSched()를 호출한 task로' 저장할지 알게된다. 구체적으로, OSTCBCurPtr은 현재 task의 OS_TCB를 가리키고, OSTCBHighRdyPtr은 전환할 새로운 OS_TCB를 가리킨다.

### L7-1(5)
현재 실행중인 task와 전환될 task가 같지 않으면, OSSched()는 context-switch를 수행할 코드를 호출한다. (165페이지의 8장 'Context-Switching'참조) 그러나 코드가 나타내듯이 task level 스케줄러는 문맥교환(context-switch)를 수행하기 위해 task-level의 함수를 호출한다.

스케줄러와 문맥교환은 인터럽트를 비활성화한 상태에서 실행된다. 이 프로세스는 원자적이어야 하기 때문이다.

## 7-4-2 OSIntExit()
ISR level 스케줄러인 OSIntExit(os_core.c 참조)에 대한 의사 코드는 L7-2에 있다. 인터럽트는 OSIntExit()이 호출될 때 비활성화된 것으로 가정한다.

```c
void OSIntExit (void)
{
    if (OSIntNestingCtr == 0) { (1)
        return;
    }
    OSIntNestingCtr--;
    if (OSIntNestingCtr > 0) { (2)
        return;
    }
    if (OSSchedLockNestingCtr > 0) { (3)
        return;
    }
    Get highest priority ready; (4)
    Get pointer to OS_TCB of next highest priority task; (5)
    if (OSTCBHighRdyPtr != OSTCBCurPtr) { (6)
    Perform ISR level context switch;
    }
}
```
### L7-2(1)
OSIntExit() 호출로 인해 OSIntNestingCtr이 wrap around되지 않는 것을 확인한다. 잘 일어나지는 않지만, 확인하는 것이 좋다(wrap around란, unsigned 변수에서, 0에서 1을 뺐을 때 최대값이 되는 것을 의미한다. OSIntNestingCtr이 0이라는 의미는, ISR이 하나도 실행 중이지 않다는 의미로, OSIntExit()이 ISR 레벨에서 실행한 것이 아닐때 발생한다).

### L7-2(2)
OSIntExit()은 ISR 마지막에 호출되므로, 중첩(nesting) 카운터를 감소시킨다. 만약 모든 중첩 ISR이 끝나지 않았으면, 코드는 단순히 return한다. 끝내야할 인터럽트가 여전히 남아있으므로, 스케줄러를 실행할 필요가 없다.

### L7-2(3)
OSIntExit()은 스케줄러가 잠겨있지 않은지 확인한다. 만약 잠겨있다면 스케줄러를 실행하지 않고 단순히 스케줄러를 잠근, 중단된 task로 돌아간다.

### L7-2(4)
여기까지왔으면 이것이 마지막 ISR이며, 스케줄러가 잠겨있지 않다. 따라서 실행해야할 가장 높은 우선 순위의 task를 찾아야한다.

### L7-2(5)
OSRdyList[]에서 가장 우선순위가 높은 OS_TCB를 추출한다.

### L7-2(6)
현재 task와 (실행 준비된)가장 높은 우선순위의 task가 다를 경우, μC/OS-III는 ISR 레벨 문맥교환(context-switch)을 수행한다. ISR레벨 context switch는 인터럽트된 task의 context가 초기에 저장되었다고 가정하고, 새로운 task의 context만 복원하면 된다는 점에서 다르다. 이는 165페이지의 8장 'context-switch'에서 설명한다.

## 7-4-3 OS_SchedRoundRobin()
한 task의 time quanta가 만료되고, 이 task와 동일한 우선순위의 task가 여러개 있을 때,  μC/OS-III는 현재 우선 순위에서 실행 준비가 된 다음 task를 선택하여 실행한다. OS_SchedRoundRobin()은 이 작업을 수행하는 데 사용되는 코드이다. OS_SchedRoundRobin()은 OStimeTick() 또는 OS_IntQTask() 중 하나에서 호출된다. OS_SchedRoundRobin()은 OS_core에 있다.

OS_SchedroundRobin()은 Direct Method를 선택한 경우 OSTimeTick()에 의해 호출된다(175페이지의 9장 'Interrupt Management' 참조). 8장에서 설명될 Deferred Post Method를 선택한 경우 OS_IntQTask()에 의해 호출된다.

round-robin 스케줄러의 의사코드가 L7-3에 있다.

```c
void OS_SchedRoundRobin (void)
{
    if (OSSchedRoundRobinEn != TRUE) { (1)
        return;
    }
    if (Time quanta counter > 0) { (2)
        Decrement time quanta counter;
    }
    if (Time quanta counter > 0) {
        return;
    }
    if (Number of OS_TCB at current priority level < 2) { (3)
        return;
    }
    if (OSSchedLockNestingCtr > 0) { (4)
        return;
    }
    Move OS_TCB from head of list to tail of list; (5)
    Reload time quanta for current task; (6)
}
//L7-3 OS_SchedRoundRobin() pseudocode
```
### L7-3(1)
먼저 round robin scheduling이 활성화되어있는지 확인하는 것으로 시작한다. round robin scheduling을 활성화하려면, OSSchedRoundRobinCfg()를 호출해야한다.

### L7-3(2)
실행 중인 task의 OS_TCB 내부에 있는 time quanta counter가 감소한다. 만약 감소되었는데도 0이 아니면 OS_SchedRoundRobin()는 단순히 return한다(즉 아직 time quanta가 남았다).

### L7-3(3)
time quanta counter가 0이 되면, OS_SchedRoundRobin()은 현재 우선순위에 다른 ready-to-run 상태의 작업이 있는지 확인한다. 만약 다른 task가 없으면, 단순히 return한다. round robin 스케줄링은 동일한 우선순위에 여러 task가 있고, task가 time quanta 내에 작업을 완료하지 못할 때만 적용된다.

### L7-3(4)
스케줄러가 잠겼으면 OS_SchedRoundRobin()은 단순히 return한다.

### L7-3(5)
OS_SchedRoundRobin()은 현재 task의 OS_TCB를 ready list의 처음 부분에서 끝 부분으로 옮긴다.

### L7-3(6)
list 맨 앞에 있는 task에 대한 time quanta를 로드한다. 각 task는 task가 생성될 때 또는 OSTaskTimeQuantaSet()을 통해 time quanta를 지정할 수 있다. time quanta를 0으로 설정하면, μC/OS-III는 기본 time quanta로 설정되었다고 가정한다. 기본 time quanta는 OSSchedRoundRobinDfltTimeQuanta에 있는 값이다.

# 7-5 Summary
μC/OS-III는 선점(preemptive) 스케줄러이므로 실행 가능한 가장 높은 우선순위의 작업을 항상 실행한다.

μC/OS-III는 동일한 우선순위에 여러 task를 허용한다. ready-to-run 상태의 task가 여러 개 있는 경우 μC/OS-III은 이 task들 사이에 round robin을 할 것이다.

스케줄링은 응용 프로그램이 특정 함수를 호출할 때 발생한다.

μC/OS-III는 두 개의 스케쥴러를 가지고 있는데, OSSched()는 task-level 코드로 호출되며, OSIntExit()은 각 ISR의 끝에 호출된다.
# Reference
 - uC/OS-III: The Real-Time Kernel For the STM32 ARM Cortex-M3, Jean J. Labrosse, Micrium, 2009

[책 링크](https://micrium.atlassian.net/wiki/spaces/osiiidoc/overview)
