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

# Message Queues
메시지 큐(message queue)는 애플리케이션에 의해 할당된 커널 객체이다. 무수히 많은 메시지 큐를 생성할 수 있으며, 사용가능한 램 용량에만 제한된다.

Fig 15-1에 정리된 바와 같이 메시지 큐 상에서 사용자가 수행할 수 있는 여러가지 함수가 있다. 그러나 ISR은 OSQPost()만 호출할 수 있다. 메시지 큐는 메시지를 전송하기 전에 작성되어야 한다.

![Figure 15-1 Operations on message queue](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/09fa94c7-68a4-443a-9bac-05869d0b6984)


메시지 큐는 FIFO(First-in, First-out Pipe)로 구현되어있다. 그러나 μC/OS-III를 사용하면 메시지들을 LIFO(Last in, first-out order)로 메시지를 post할 수 있다. LIFO 메커니즘은 task 또는 ISR이 task에 긴급 메시지를 보내야할 때 유용하다. 이 경우 메시지는 메시지 큐에 이미 있는 다른 모든 메시지를 우회한다. 메시지 큐의 크기는 런타임에 설정 가능하다.

수신 task에 가까운 작은 모래시계(F15-1)는 타임아웃 옵션이 있다는 것을 의미한다. 이 타임아웃은 메시지가 메시지 큐로 전송될 때까지 task가 특정 시간 동안 대기한다는 것을 나타낸다. 메시지가 그 시간 내에 전송되지 않으면, μC/OS-III는 task를 재개하고 타임아웃 때문에 task가 실행 준비가 되었다는 것을 나타내는 오류 코드를 반환한다. 무한 타임아웃을 지정하고, task가 메시지가 도착할 때까지 영원히 기다릴 것이라는 것을 나타낼 수 있다.

메시지 큐에는 메시지가 메시지 큐로 전송되기를 기다리는 task의 목록도 있다. Fig 15-2와 같이 여러 task가 메시지 큐에 대기할 수 있다. 메시지가 메시지 큐로 전송되면, 메시지 큐에서 대기 중인 가장 높은 우선순위의 task가 메시지를 수신한다. 송신자는 메시지 큐에서 대기 중인 모든 task에 메시지를 브로드캐스트할 수 있다. 이 경우 브로드캐스트를 통해 메시지를 수신하는 task 중 어느 하나라도 메시지를 보내는 task보다 높은 우선순위(또는 인터럽트때문에 중단된 task, 즉 ISR에 의해 메시지가 전송된 경우)를 가진다면, μC/OS-III는 대기 중인 가장 높은 우선순위의 task를 실행할 것이다. 모든 task가 타임아웃을 지정해야하는 것은 아니다. 일부 task는 메시지 큐에 영원히 대기하기를 원할 수도 있다.

![Figure 15-2 Multiple tasks waiting on a message queue](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/156843b8-e414-45fc-9039-bdca77d05ef8)

# Task Message Queue
하나의 메시지 큐에 여러 개의 task가 대기하는 것은 상당히 드물다. 이 때문에 각 task에 메시지 큐가 내장되어 사용자는 외부 메시지 큐 개체를 거치지 않고 직접 task에 메시지를 보낼 수 있다. 이 기능은 코드를 단순화할 뿐만 아니라 별도의 메시지 큐 객체를 사용하는 것보다 효율적이다. 각 task에 내장된 메시지 큐는 Fig 15-3과 같다.

![Figure 15-3 Task message queue](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/f00d03ef-a28b-4399-b929-502bdc45835e)


μC/OS-III의 task 메시지 큐 서비스는 OSTaskQ???() 접두사로 시작하며, 애플리케이션 프로그래머가 사용할 수 있는 서비스는 443페이지 Appendix A  "μC/OS-III API Reference"에 설명되어 있다. os_cfg.h에서 OS_CFG_TASK_Q_EN을 set하면 task 메시지 큐 서비스를 사용할 수 있다. task 메시지 큐를 위한 코드는 os_task.c에 있다.

만약 코드가 어떤 task로 메시지를 보내야 할지 알고 있다면 이 기능을 사용한다. 예를 들어 이더넷 컨트롤러로부터 인터럽트를 받았다면 수신한 패킷의 주소를 수신패킷처리를 담당하는 task로 보낼 수 있다.

# Bilateral Rendez-Vous
![Figure 15-4 Bilateral Rendez-vous](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/886b9624-12ab-429e-9f15-8320b5815f90)

bilateral rendez-vous에서 각 메시지 큐는 최대 1개의 메시지를 가질 수 있다. 처음에는 양쪽 메시지 큐가 모두 비어있다. 왼쪽의 task가 rendez-vous point에 도달하면 상단 메시지 큐에 메시지를 보내고, 하단 메시지 큐에 메시지가 도착하기를 기다린다. 마찬가지로 오른쪽의 task가 rendez-vous point에 도달하면 하단의 메시지 큐에 메시지를 보내고 상단 메시지 큐에 메시지가 도착하기를 기다린다.

F15-5는 task 메시지 큐를 사용하여 bilateral rendez-vous를 수행하는 방법을 보여준다.

![Figure 15-5 Figure Bilateral Rendez-vous with task message queues](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/e4aef51b-dab4-43bf-aadc-c44eb157af3b)

# Flow Control
task간 통신은 종종 한 task에서 다른 task로의 데이터 전송이다. 한 task는 데이터를 생산(produce)하고 다른 task는 데이터를 소비(consume)한다. 그러나 데이터 처리는 시간이 걸리고 소비자는 데이터를 생산한 만큼 빠르게 소비하지 못할 수 있다. 즉, 우선순위가 높은 task가 소비자를 선점할 경우, 생산자가 메시지 큐를 오버플로우시킬 수 있다. 이러한 문제를 해결하기 위한 한 가지 방법은 Fig15-6과 같이 flow control을 추가하는 것이다.

![Figure 15-6 Producer and consumer tasks with flow control](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/83474a56-c470-4b97-bf02-3206fed052e3)


여기서 카운팅 세마포어는 저장할 수 있는 메시지의 개수로 초기화되어 사용된다. 메시지 큐가 10개 이상의 메시지를 저장할 수 없는 경우, 카운팅 세마포어는 10의 카운트를 포함한다.

L15-1의 의사 코드에서 보는 바와 같이, 생산 task는 세마포어에서 대기한 후, 메시지를 보낼 수 있다. 소비 task는 메시지를 기다리고, 메시지를 처리하면 세마포어에 신호를 보낸다.

```c
Producer Task:
    Pend on Semaphore;
    Send message to message queue;

Consumer Task:
    Wait for message from message queue;
    Signal the semaphore;
//L15-1 Producer and consumer flow control
```
task 메시지 큐와 task 세마포어(273페이지의 14장 "Synchronization" 참조)를 결합하면 Fig15-7과 같이 flow control을 구현할 수 있다. 그러나 이 경우 task 세마포어의 값을 task 메시지 큐의 최대 허용 메시지 수와 동일한 값으로 설정하려면 task 생성 직후 OSTaskSemSet()를 호출해야 한다.

![Figure 15-7 Flow control with task semaphore and task message queue](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/5cef38dd-1147-4479-8d29-44e2c4790843)

# Keeping The Data In Scope
일반적으로 전송되는 메시지는 자료구조, 변수, 배열, 테이블 등을 가리킨다. 그러나 중요한 것은 데이터의 수신자가 데이터 처리를 완료할 때까지 데이터는 변하지 않아야 한다. 일단 전송된 데이터는 송신자가 건드리지 않아야한다. 이는 당연해 보이지만 잊어버리기 쉽다.

예를 들면, μC/OS-III와 함께 제공되는 고정 크기의 메모리 파티션 관리자( 343페이지 Chapter 17, "Memory Management")를 사용하여 데이터를 전달하는 데 사용되는 메모리 블록을 동적으로 할당하고 확보하는 것이다. Fig15-8은 예시를 보여준다. 예시를 위해 어떤 장치가 어떤 프로토콜을 사용하여 데이터 바이트를 패킷으로 UART를 통해 전송하고 있다고 가정하자. 이 경우 패킷의 첫 번째 바이트는 고유 하고 패킷의 끝 바이트도 고유하다.

```c
void UART_ISR (void)
{
    OS_ERR err;

    RxData = Read byte from UART;
    if (RxData == Start of Packet) { /* See if we need a new buffer */
        RxDataPtr = OSMemGet(&UART_MemPool, /* Yes */
                            &err);
        *RxDataPtr++ = RxData;
        RxDataCtr = 1;
    } if (RxData == End of Packet byte) { /* See if we got a full packet */
        *RxDataPtr++ = RxData;
        RxDataCtr++;
        OSQPost((OS_Q *)&UART_Q, /* Yes, post to task for processing */
                (void *)RxDataPtr,
                (OS_MSG_SIZE)RxDataCtr,
                (OS_OPT )OS_OPT_POST_FIFO,
                (OS_ERR *)&err);
        RxDataPtr = NULL; /* Don’t point to sent buffer */
        RxDataCtr = 0;
    } else; {
        *RxDataPtr++ = RxData; /* Save the byte received */
        RxDataCtr++;
    }
}
//L15-2 UART ISR Pseudo-code
```

![Figure 15-8 Using memory partitions for message contents](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/86eca083-6255-4001-9856-3a52bc7789af)


## F15-8(1)
문자를 수신하면, UART는 인터럽트를 발생시킨다.

## F15-8(2)
L15-2의 의사 코드는 UART ISR 코드가 어떤 모습일지 보여준다. 여기에는 간략화를 위해 생략된 부분이 많다. ISR은 UART로 부터 받은 바이트를 읽어 그것이 패킷의 시작에 해당하는지 본다. 그렇다면 메모리 파티션으로부터 버퍼를 얻게 된다.

## F15-8(3)
수신된 바이트는 버퍼에 저장된다.

## F15-8(4)
만약 수신된 데이터가 패킷 종료 바이트라면, 단순히 버퍼의 주소를 메시지 큐에 게시하여 task가 수신된 패킷을 처리할 수 있을 것이다.

## F15-8(5)
만약 전송된 메시지가 UART task를 가장 우선순위가 높은 task로 만든다면, μC/OS-III는 (인터럽트 때문에) 중단된 task로 돌아가는 대신 해당 task(UART task)로 전환할 것이다. task는 메시지 큐에서 패킷을 검색한다. OSQPend()는 패킷의 바이트 수와 메시지가 전송된 시간을 나타내는 타임스탬프를 반환한다.

패캣을 처리하는 task가 끝나면 OSMemPut()을 호출하여 버퍼는 메모리 파티션으로 반환된다.

# Using Message Queues
표 15-1은 μC/OS-III에서 이용 가능한 메시지 큐 서비스의 요약을 보여준다. 전체 설명은 443페이지의 Appendix A, "μC/OS-III API Reference" 를 참조한다.

| Function Name  | Operation |
|----------------|-----------|
| OSQCreate()    |메시지 큐를 생성한다.|
| OSQDel()       |메시지 큐를 삭제한다.|
| OSQFlush()     |메시지 큐를 비운다.|
| OSQPend()      |메시지 큐에 대기한다.|
| OSQPendAbort() |메시지 큐에 대기하는 것을 중단한다.|
| OSQPost()      |메시지 큐를 통해서 메시지를 보낸다.|

(표 15-1 Message queue API summary)

표 15-2은 μC/OS-III에서 이용 가능한 task 메시지 큐 서비스의 요약을 보여준다. 전체 설명은 443페이지의 Appendix A, "μC/OS-III API Reference" 를 참조한다.

| Function Name      | Operation |
|--------------------|-----------|
| OSTaskQPend()      |메시지를 대기한다.|
| OSTaskQPendAbort() |메시지를 대기하는 것을 중단한다.|
| OSTaskQPost()      |task에 메시지를 보낸다.|
| OSTaskQFlush()     |메시지 큐를 비운다.|

Fig15-9는 회전하는 바퀴의 속도를 판단할 때 메시지 큐를 사용하는 예를 보여준다.

![Figure 15-9 Measuring RPM](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/360a0b69-cf81-4248-84d6-d5cdd3ebcbc1)


## F15-9(1)
회전하는 바퀴의 RPM을 측정하는 것이 목표이다.

## F15-9(2)
센서는 바퀴에 있는 구멍이 지나가는 것을 감지하는데 사용된다. 정확도를 위해, 바퀴에는 동일한 간격으로 있는 다수의 구멍이 있을 수 있다.

## F15-9(3)
구멍이 감지될 때마다 free-running counter 값을 캡처하기 위해 32비트 input capture 레지스터가 사용된다.

## F15-9(4)
구멍이 감지되면, 인터럽트가 발생한다. ISR은 input capture 레지스터에 있는 현재 카운트를 보고 이전 캡처의 값을 빼서 1회전(구멍이 한개만 있다고 가정)에 걸린 시간을 계산한다.

```c
Delta Counts = Current Counts – Previous Counts;
Previous Counts = Current Counts;
```

## F15-9(5)(6)
Delta Counts는 메시지 큐로 보내진다. 메시지는 실제로 포인터이므로, 사용 중인 프로세서에서 포인터가 32비트라면 간단히 32비트 delta count값을 포인터로 형변환하여 메시지 큐를 통해 전송할 수 있다. 더 안전하고 이식성이 높은 접근법은 μC/OS-III의 메모리 관리 서비스(343페이지의 Chapter 17, "Memory Management" 참조 )에서 나오는 메모리 블록을 사용하여 delta counts를 저장할 공간을 동적으로 할당하고 할당된 메모리 블록의 주소를 보내는 것이다. 읽은 count 값은 다음 인터럽트 때 사용할 Previous Counts에 저장된다.

## F15-9(7)
메시지가 전송되면, RPM 측정 task가 깨어나고, 다음과 같이 RPM을 계산한다:

```c
RPM = 60 * Reference Frequency / Delta Counts;
```
pend 함수 호출 시 타임아웃을 지정할 수 있고, 타임아웃 내에 메시지가 전송되지 않으면 task는 깨어날 것이다. 이는 바퀴가 회전하지 않는다는 것이고, 따라서 RPM은 0이다.

## F15-9(8)
task는 RPM 계산과 함께, 평균 RPM, 최대 RPM, 속도가 임계값 이상인지 미만인지 등도 계산할 수 있다.

# Using Message Queues(2)
위의 예에서 몇 가지 흥미로운 것이 있다. 첫째, ISR은 매주 짧다. input capture를 읽고 delta count를 task에 post하여 시간이 걸리는 계산은 task로 넘길 수 있다. 둘째, pend에 timeout을 지정하여, 바퀴가 멈춘 것을 쉽게 감지할 수 있다. 마지막으로, task는 추가 계산을 할 수 있고, 바퀴가 너무 빨리, 또는 너무 느리게 회전하는 것과 같은 오류를 더 감지할 수 있다. task는 이러한 오류를 다른 task에 알릴 수 있다.

L15-3은 μC/OS-III의 메시지 큐 서비스를 이용한 RPM 측정 예제를 구현하는 방법을 보여준다. 코드의 일부는 의사 코드인 반면, μC/OS-III 서비스 호출은 적절한 argument를 가진 실제 호출이다.

```c
OS_Q RPM_Q; (1)
CPU_INT32U DeltaCounts;
CPU_INT32U CurrentCounts;
CPU_INT32U PreviousCounts;

void main (void)
{
    OS_ERR err ;
    :
    OSInit(&err) ; (2)
    :
    OSQCreate((OS_Q *)&RPM_Q,
                (CPU_CHAR *)"My Queue",
                (OS_MSG_QTY)10,
                (OS_ERR *)&err);
    :
    OSStart(&err);
}
void RPM_ISR (void) (3)
{
OS_ERR err;

    Clear the interrupt from the sensor;
    CurrentCounts = Read the input capture;
    DeltaCounts = CurrentCounts – PreviousCounts;
    PreviousCounts = CurrentCounts;
    OSQPost((OS_Q *)&RPM_Q, (4)
            (void *)DeltaCounts,
            (OS_MSG_SIZE)sizeof(void *),
            (OS_OPT )OS_OPT_POST_FIFO,
            (OS_ERR *)&err);
}

void RPM_Task (void *p_arg)
{
    CPU_INT32U delta;
    OS_ERR err;
    OS_MSG_SIZE size;
    CPU_TS ts;

    DeltaCounts = 0;
    PreviousCounts = 0;
    CurrentCounts = 0;
    while (DEF_ON) {
        delta = (CPU_INT32U)OSQPend((OS_Q *)&RPM_Q, (5)
                                    (OS_TICK )OS_CFG_TICK_RATE_HZ * 10,
                                    (OS_OPT )OS_OPT_PEND_BLOCKING,
                                    (OS_MSG_SIZE *)&size,
                                    (CPU_TS *)&ts,
                                    (OS_ERR *)&err);
        if (err == OS_ERR_TIMEOUT) { (6)
            RPM = 0;
        } else {
            if (delta > 0u) {
                RPM = 60 * Reference Frequency / delta; (7)
            }
        }
        Compute average RPM; (8)
        Detect maximum RPM;
        Check for overspeed;
        Check for underspeed;
        :
        :
    }
}
```
## L15-3(1)
변수가 선언된다. 메시지 큐를 위한 저장소 할당이 필요하다.

## L15-3(2)
OSInit()를 호출하고 메시지 큐를 만든 후 사용해야한다. 이 작업을 수행하기 위한 최적의 장소는 시작 코드이다.

## L15-3(3)
RPM ISR은 센서 인터럽트를 지우고 32비트 input capture의 값을 읽는다. 16비트 input capture만 있는 경우에도 RPM을 읽을 수 있다. 다만 RPM이 낮을 경우 오버플로우가 발생할 수 있다.

RPM ISR은 delta count를 직접 계산한다. 현재 count를 post하고 task가 delta를 계산하도록 하는 것도 가능하다. 그러나 빼기는 빠른 연산이고, ISR처리 시간을 크게 증가시키지 않는다.

## L15-3(4)
그런 다음 코드는 delta count를 RPM 계산을 담당하는 RPM task로 전송한다. 값을 post하려고 할 때 큐가 가득 차있으면 메시지가 손실된다는 점에 유의한다. 데이터가 처리되는 것보다더 빨리 생성되면 이러한 일이 발생한다. 불행히도 ISR이 있기 때문에 flow control을 구현할 수 없다.

## L15-3(5)
RPM task는 메시지 큐를 대기함으로써 RPM ISR로부터 메시지가 오기를 기다린다. 세 번째 argument는 타임아웃 값이다. 이 경우 10초이다. 그러나 타임아웃 값은 애플리케이션 요구사항에 따라 달라진다.

또한 ts 변수에는 post가 완료된 시점의 타임스탬프 값이 포함되어 있다. OS_TS_GET()을 호출하고 ts 값을 빼서 메시지를 수신한 이후 task가 실행되기까지 걸린 시간을 측정할 수 있다:

```c
response_time = OS_TS_GET() – ts;
```

## L15-3(6)
타임아웃이 발생하면 바퀴가 더 이상 회전하지 않는다고 생각할 수 있다.

## L15-3(7)
RPM은 수신한 delta count와 free running counter의 주파수로부터 계산된다.

## L15-3(8)
추가적인 계산은 필요에 따라 수행된다. 사실 오류 조건들이 검출된 경우 메시지는 다른 task로 전송될 수 있고, 이 경우 메시지는 다른 task에 의해 처리될 것이다. 예를 들어 바퀴가 너무 빨리 회전하는 경우 다른 task는 바퀴 속도를 제어하는 장치를 끌 수 있다.

# Using Message Queues(3)
L15-4에서 OSQPost()와 OSQPend()가 OSTaskQPost()와 OSTaskQPend()로 대체된다. 코드는 조금 더 간단하고 별도의 메시지 큐를 만들 필요가 없다. 그러나 RPM task를 생성할 때, task에서 사용하는 메시지 큐의 크기를 지정하고 OS_CFG_TASK_Q_EN을 1로 설정하여 컴파일하는 것이 중요하다. 메시지 큐를 사용하는 것과 task 메시지 큐를 사용하는 것의 차이점에 대해 설명한다.

```c
OS_TCB RPM_TCB; (1)
OS_STK RPM_Stk[1000];
CPU_INT32U DeltaCounts ;
CPU_INT32U CurrentCounts ;
CPU_INT32U PreviousCounts ;

void main (void)
{
    OS_ERR err ;
    :
    OSInit(&err) ;
    :
    void OSTaskCreate ((OS_TCB *)&RPM_TCB, (2)
                        (CPU_CHAR *)"RPM Task",
                        (OS_TASK_PTR )RPM_Task,
                        (void *)0,
                        (OS_PRIO )10,
                        (CPU_STK *)&RPM_Stk[0],
                        (CPU_STK_SIZE )100,
                        (CPU_STK_SIZE )1000,
                        (OS_MSG_QTY )10,
                        (OS_TICK )0,
                        (void *)0,
                        (OS_OPT )(OS_OPT_TASK_STK_CHK + OS_OPT_TASK_STK_CLR),
                        (OS_ERR *)&err);
                        :
                        OSStart(&err);
}

void RPM_ISR (void)
{
    OS_ERR err;
    Clear the interrupting from the sensor;
    CurrentCounts = Read the input capture;
    DeltaCounts = CurrentCounts – PreviousCounts;
    PreviousCounts = CurrentCounts;
    OSTaskQPost((OS_TCB *)&RPM_TCB, (3)
                (void *)DeltaCounts,
                (OS_MSG_SIZE)sizeof(DeltaCounts),
                (OS_OPT )OS_OPT_POST_FIFO,
                (OS_ERR *)&err);
}

void RPM_Task (void *p_arg)
{
    CPU_INT32U delta;
    OS_ERR err;
    OS_MSG_SIZE size;
    CPU_TS ts;

    DeltaCounts = 0;
    PreviousCounts = 0;
    CurrentCounts = 0;
    while (DEF_ON) {
        delta = (CPU_INT32U)OSTaskQPend((OS_TICK )OS_CFG_TICK_RATE * 10, (4)
                                        (OS_OPT )OS_OPT_PEND_BLOCKING,
                                        (OS_MSG_SIZE *)&size,
                                        (CPU_TS *)&ts,
                                        (OS_ERR *)&err);
        if (err == OS_ERR_TIMEOUT) {
            RPM = 0;
        } else {
            if (delta > 0u) {
                RPM = 60 * ReferenceFrequency / delta;
            }
        }
        Compute average RPM;
        Detect maximum RPM;
        Check for overspeed;
        Check for underspeed;
        :
        :
    }
}
//L15-4 Pseudo-code of RPM measurement
```

## L15-4(1)
메시지 큐를 선언하는 대신, 메시지를 수신할 task의 OS_TCB를 아는 것이 중요하다.

## L15-4(2)
RPM task가 생성되고, 큐 크기는 10으로 지정된다. 물론 실제 애플리케이션에서는 hard-coded 값이 아니라 define을 사용하여야한다, 여기서는 설명을 위해 고정된 숫자를 사용한다.

## L15-4(3)
ISR은 메시지 큐에 post하는 대신 task의 OS_TCB 주소를 이용해 메시지를 task에 직접 post한다.

## L15-4(4)
RPM task는 OSTaskQPend()를 호출하여 RPM ISR로부터의 메시지를 기다리는 것으로 시작한다. 현재 task가 pend할 것임을 가정하므로, OS_TCB 주소를 지정할 필요가 없다. 두 번째 argument는 타임아웃이다. 여기서 타임아웃은 10초이며, 이는 6RPM을 의미한다.

# Clients And Servers
메시지 큐를 사용하는 또 다른 예는 Fig15-10에 있다. 여기서 task(서버)는 다른 task나 ISR(클라이언트)이 보내는 오류를 확인하는데 사용된다. 예를 들어, 클라이언트는 회전하는 바퀴의 RPM이 초과되었는지를 감지하고, 다른 클라이언트는 과열이 있는지 감지하고, 다른 클라이언트는 사용자가 종료 버튼을 누른 것을 감지한다. 클라이언트가 오류를 감지하면 메시지 큐를 통해 메시지를 보낸다. 전송된 메시지는 감지된 오류, 어떤 임계값이 초과되었는지, 오류 코드를 나타내거나, 오류를 처리할 함수의 주소를 제시하는 등을 나타낸다.

![Figure 15-10 Clients and Servers](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/bf3fe6e6-83b2-4fa7-91b3-64b5fe725d3f)

# Message Queues Internals
앞서 설명한 바와 같이 메시지는 실제 데이터를 가리키는 포인터와, 실제 데이터의 크기를 나타내는 변수, 그리고 메시지가 실제로 전송된 시점을 나타내는 타임스탬프로 구성된다. 전송될 때 메시지는 Fig15-11과 같이 OS_MSG 타입의 자료형에 있게 된다.

![Figure 15-11 OS_MSG structure](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/64eb8073-a6d6-4767-b7ef-c7d9a1c49227)

μC/OS-III는 사용 가능한 OS_MSG 풀을 관리한다. 풀에서 사용 가능한 메시지의 총 수는 os_cfg_app.h에 있는 OS_CFG_MSG_POOL_SIZE 값에 의해 결정된다. μC/OS-III가 초기화되면 OS_MSG들은 Fig15-12와 같이 single linked list로 연결된다. OS_MSG_POOL 타입의 자료구조(4개의 필드를 가지고 있음)에 의해 free list가 유지된다. 4개의 필드는 .NextPtr(free list를 가리킴), .NbrFree(풀에서 사용가능한 OS_MSG 수), .NbrUsed(애플리케이션에 할당된 OS_MSG 수), .NbrUsedMax(애플리케이션에 할당된 최대 메시지 개수)이다.

![Figure 15-12 Pool of free OS_MSGs](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/bb18da9f-9dce-4e13-a874-e5ba66fbea6f)

메시지는 Fig15-13과 같이 OS_MSG_Q 타입의 자료구조를 이용하여 큐잉된다.

![Figure 15-13 OS_MSG_Q structure](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/fc8238e4-7a52-426f-8c4f-6273be2ed07a)



# Reference
 - uC/OS-III: The Real-Time Kernel For the STM32 ARM Cortex-M3, Jean J. Labrosse, Micrium, 2009

[책 링크](https://micrium.atlassian.net/wiki/spaces/osiiidoc/overview)