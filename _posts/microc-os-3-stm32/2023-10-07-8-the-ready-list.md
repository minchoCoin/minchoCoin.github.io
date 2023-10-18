---
title: "μC/OS-III ch.6 The Ready List"
last_modified_at: 2023-10-07T12:53:12+09:00
categories:
    - microc-os-3-stm32
tags:
    - microc-os
    - embedded-system

toc: true
toc_label: "My Table of Contents"
author_profile: true

---

실행 준비가 된 task는 ready list에 배치된다. ready list는 두 부분으로 구성된다. 준비가 된 우선순위를 가지고 있는 비트맵과 실행 준비가 된 모든 task에 대한 포인터 테이블이다.

# 6-1 Priority levels
그림 6-1에서 6-3은 준비가 된 우선순위의 비트맵을 보여준다. 테이블의 폭은 cpu.h에 있는 CPU_DATA 타입에 따라 다르며, 8,16,32bit가 될 수 있다. 즉 폭은 프로세서가 사용하는 것에 따라 달라진다.

μC/OS-III는 최대 OS_CFG_PRIO_MAX 개의 다른 우선순위 수준을 허용한다(os_cfg.h 참조). μC/OS-III에서 낮은 우선 순위 번호는 높은 우선 순위 수준에 해당한다. 따라서 우선 순위 0은 우선순위가 최고 이고, 우선순위 OS_CFG_PRIO_MAX-1 은 우선순위가 최저이다. μC/OS-III는 idle task에 최저 우선순위를 유일하게 할당하므로, 다른 task가 이 우선순위를 가지는 것을 허용하지 않는다. 어떤 우선순위에 해당하는 task가 ready-to-run 상태이면, 해당하는 bit가 set된다(즉 1). 그림 6-1~6-3에서 우선순위 번호가 왼쪽에서 오른쪽으로 번호가 매겨져 있으며, 테이블 인덱스가 증가하면서 우선순위 번호도 커진다(즉 우선순위가 낮아짐). 이 순서는 가장 높은 우선순위를 파악하는 과정을 가속화하는 Count Leading Zeros(CLZ)라는 특별한 명령어를 사용하기 위해 사용된다. CLZ는 많은 현대 프로세서에서 사용한다.

![CPU_DATA declared as a CPU_INT08U](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/ef90d68b-b5d9-42da-a737-f4a347bc1da2)

![CPU_DATA declared as a CPU_INT16U](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/10e32129-9cc2-4a3a-ab72-47caec50ba07)

![CPU_DATA declared as a CPU_INT32U](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/cf2862f1-434f-488b-a04a-4d284b56e2f2)

os_prio.c는 비트맵 테이블을 설정, 초기화, 검색하는 코드를 포함한다. 이 함수들은 μC/OS-III 내부 함수 이며  필요한 경우 os_prio.c를 어셈블리 언어에 해당하는 OS_PRIO.ASM으로 대체하여 어셈블리 언어로 최적화할 수 있도록 하기 위해 os_prio.c에 위치한다

| Function            | Description                                            |
|---------------------|--------------------------------------------------------|
| OS_PrioGetHighest() | 가장 높은 우선순위를 찾는다                            |
| OS_PrioInsert()     | 비트맵 테이블에서 해당하는 우선순위의 비트를 set한다   |
| OS_PrioRemove()     | 비트맵 테이블에서 해당하는 우선순위의 비트를 reset한다 |

ready-to-run task 중 가장 높은 우선순위인 것을 확인하려면, 비트맵 테이블은 OS_PrioGetHighest()를 사용하여 테이블에서 첫번째로 bit가 set인 것을 찾을 때 까지 스캔한다.

이 함수의 코드가 L6-1에 있다.

```c
OS_PRIO OS_PrioGetHighest (void)
{
    CPU_DATA *p_tbl;
    OS_PRIO prio;
    prio = (OS_PRIO)0;
    p_tbl = &OSPrioTbl[0];
    while (*p_tbl == (CPU_DATA)0) { (1)
        prio += DEF_INT_CPU_NBR_BITS; (2)
        p_tbl++;
    }
    prio += (OS_PRIO)CPU_CntLeadZeros(*p_tbl); (3)
    return (prio);
}
```
## L6-1(1)
OS_PrioGetHighest()는 0이 아닌 값이 발견될 때까지 OSPrioTbl[0]부터 1씩 증가하면서 테이블을 스캔한다. idle_task으로 인해 테이블에 0이 아닌 값이 항상 존재하기 때문에 루프는 항상 종료된다.

(예를 들어 OSPrioTbl[0]=0b00000000(=0), OSPrioTbl[1]=0b00010000(!=0)일때 p_tbl은 &OSPrioTbl[1]에서 멈춘다)

## L6-1(2)
각 항목에서 0이 발견될 때마다 다음 테이블 항목으로 이동하고 "prio"를 각 항목의 너비(비트 수) 만큼 증가시킨다. 각 항목이 32bit인 경우 "prio"는 32만큼 증가된다.

## L6-1(3)
첫 번째로 0이아닌 항목이 발견되면 해당 항목(값)의 leading zeros 수를 더하고 계산된 우선순위를 호출자에게 반환한다. 1이 나타날때까지 왼쪽부터 0의 수를 세는 것(counting leading zeros)은 CPU에 따라 있을 수도 있고 없을 수도있다. 만약 없다면, C로 구현해야한다.

(예를 들어 OSPrioTbl[0] = 0b00000010 이면 우선순위는 8*0 + 6 = 6, 여기서 계산에 사용된 6은 count of leading zeros이다.) 

CPU_CntLeadZeros() 함수는 CPU_DATA 항목에서 왼쪽(즉, 최상위 비트)부터 얼마나 많은 0이 있는지를 단순히 계산한다. 예를 들어, 32비트를 가정하면 0xF0001234는 선행하는 0이 없고, 0x00F01234는 선행하는 0이 8개다.

테이블을 통한 선형적으로 검색하는 것이 비효율적으로 보일 수 있다. 그러나 우선순위의 개수가 작게 유지되면 검색은 빠르다. 검색을 간소화하기위한 여러 최적화가 있는데, 예를 들어 32bit 프로세서를 사용하고 우선순위의 수를 64로 제한한다면 위의 코드는 L6-2로 최적화될 수 있다. 일부 프로세서는 내장된 Count Leading Zeros 명령어가 있으므로, 코드를 C 대신 어셈블리어로 작성할 수 있다. μC/OS-III에서, 64개의 우선순위가 task가 64개로 제한된다는 뜻은 아니다. 같은 우선순위를 가진 task는 얼마든지 있을 수 있다(0과 OS_CFG_PRIO_MAX-1 제외).

```c
OS_PRIO OS_PrioGetHighest (void)
{
    OS_PRIO prio;
    if (OSPrioTbl[0] != (OS_PRIO_BITMAP)0) {
        prio = OS_CntLeadZeros(OSPrioTbl[0]);
    } else {
        prio = OS_CntLeadZeros(OSPrioTbl[1]) + 32;
    }
    return (prio);
}
```

# 6-2 The Ready List
ready-to-run task는 ready list에 배치된다. Fig 6-4처럼, ready list는 OSRdyList[]라는 배열이고 배열의 크기는 OS_CFG_PRIO_MAX이며, 배열의 데이터 타입은 OS_RDY_LIST(os.h 참조)이다. OS_RDY_LIST 타입은 .Entries, .TailPtr, .HeadPtr 3개의 필드로 구성된다.

.Entries는 ready list의 인덱스에 해당하는 우선순위의 ready-to-run task의 수를 가지고 있다. 해당하는 task가 없는 경우, entries는 0으로 설정된다.

.TailPtr과 .HeadPtr은 특정 우선 순위로 준비된 모든 task의 doubly linked list를 만드는데 사용됩니다. .HeadPtr은 list의 head(처음)를 가리키고, .TailPtr은 list의 tail(끝)을 가리킨다.

array의 index는 task의 우선순위이다. 예를 들어, task가 우선순위 5에서 생성되고, 그 task가 ready-to-run이면 OSRdyList[5]에 삽입될 것이다.

table 6-2는 μC/OS-III가 ready list를 조작하기 위한 기능을 나타낸다. 이 함수들은 os_core.c에 있으며, μC/OS-III의 내부 함수이므로 응용 프로그램 코드가 이 함수들을 호출하지 않아야 한다.

| Function                   | Description                                     |
|----------------------------|-------------------------------------------------|
| OS_RdyListInit()           | ready list를 빈 상태로 초기화한다(Fig 6-4 참조) |
| OS_RdyListInsert()         | ready list에 TCB를 넣는다                       |
| OS_RdyListInsertHead()     | ready list의 head부분에 TCB를 넣는다            |
| OS_RdyListInsertTail()     | ready list의 tail부분에 TCB를 넣는다            |
| OS_RdyListMoveHeadToTail() | TCB를 ready list의 head부분에서 tail로 옮긴다   |
| OS_RdyListRemove()         | TCB를 ready list에서 제거한다                   |

(table 6-2 Ready List access functions)

![Empty Ready List](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/250f6a98-14a8-463f-8e01-2032ddfed837)

내부 μC/OS-III의 작업이 모두 활성화되었다고 가정하면 Fig 6-5는 OSInit()를 호출한 후 ready list의 상태(즉, μC/OS-III가 초기화된 모습)를 보여준다. 각 μC/OS-III 작업은 고유한 우선순위를 가지고 있었다고 가정한다(실제로는 그렇지 않다).

## F6-4(1)
OSRdyList[OS_CFG_PRIO_MAX-1]에는 하나의 항목만 있다: idle task

## F6-4(2)
list는 OS_TCB를 가리킨다. 여기서는 TCB의 관련된 필드들만 보여준다. .PrevPtr과 .NextPtr은 같은 우선순위에 있는 작업들의 OS_TCB들의 doubly linked list를 형셩하는데 사용된다. idle task의 경우, 이 필드들은 항상 NULL을 가리킨다.

## F6-4(3)
우선순위 0은 os_cfg.h에서 OS_CFG_ISR_DEFERRED_EN이 1로 설정되면, ISR handler task만 쓸 수 있다. 즉 이 경우 ISR handler task는 우선순위 0에서 실행될 수 있는 유일한 작업이다.


![Ready List after calling OSInit()](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/27994553-861a-4566-bae0-382d88d17b00)

## F6-5(1)
tick task 및 다른 세 개의 선택적 task는 고유의 우선순위를 갖는다. 일반적으로, tick task의 우선순위를 timer task보다 높게 설정하고, timer task를 statistic task보다 높게 설정할 것이다.

## F6-5(2)
특정 우선순위 레벨에 TCB가 하나밖에 없다면 tail 과 head pointer는 같은 TCB를 가리키고 있을 것이다.

# 6-3 Adding Tasks To The Ready List
task는 다수의 μC/OS-III 서비스에 의해 ready list에 추가된다. 예를 들어 OSTaskCreate()가 있는데, ready-to-run 상태의 task를 생성하여 ready list에 추가한다. Fig 6-6과 같이 task를 생성하고, 원하는 우선순위의 ready list에 task가 이미 존재할때(이 예에서는 2개)를 보자. OSTaskCreate()는 해당 우선 순위 list의 마지막에 새로운 task를 삽입할 것이다.
![Inserting a newly created task in the ready list](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/9cae12b4-86a3-4714-abc1-d0917e6a773b)

## F6-6(1)
이 예제에서, OSTaskCreate()를 호출하기 전에, 우선순위 'prio'로 ready list 에있는 작업 2개가 있다.

## F6-6(2)
새로운 TCB는 OSTaskCreate()로 전달되며, μC/OS-III는 해당 TCB의 내용을 초기화한다.

## F6-6(3)
OSTaskCreate()는 OS_RdyListInsertTail()을 호출하며, 이는 4개의 포인터를 설정하고 OSRdyList[prio]의 .Entries 필드도 증가시켜 새로운 TCB를 ready list에 연결한다. Fig 6-6에 표시되지 않은 것은, OSTaskCreate()가 OS_PrioInsert()도 호출하여 비트맵 테이블의 비트를 설정하는 것이다. 물론, OSRdyList[Prio]에 Prio 우선순위에 ready-to-run 상태인 task의 리스트가 존재하기 때문에 그 작업은 필요하지 않지만, OS_PrioInsert()는 매우 빠르므로, 성능에 영향을 주지 않는다.

새로운 TCB가 list의 끝에 추가되는 이유는 list의 head에 있는 task가 task를 동일한 우선순위로 만들었을 수 있기 때문이다. 따라서 새로운 task를 바로 다음에 실행할 task로 할 이유가 없다. 실제로, 현재 task와 동일한 우선순위의 task가 ready-to-run 상태로 만들어지면, list의 끝에 삽입된다. 그러나 현재 task와 다른 우선순위의 task가 ready-to-run 상태로 만들어지면 list의 맨 앞에 삽입된다.

# 6-4 Summary
μC/OS-III는 임의의 수의 우선순위 레벨을 지원한다. 그러나 256개의 우선순위 레벨은 가장 복잡한 응용 프로그램에서도 충분하며, 대부분의 시스템은 64레벨 이상 필요하지 않다.

ready list는 준비된 우선순위 레벨을 추적하는 bitmap table과 각 우선순위에서 ready 상태인 모든 task의 목록을 포함하는 테이블 두 가지 자료구조로 구성된다.

count leading zeros 명령어를 가진 프로세서는 가장 높은 우선순위의 task를 결정하는데 사용되는 bitmap table 탐색 프로세스를 가속화할 수 있다.
# Reference
 - uC/OS-III: The Real-Time Kernel For the STM32 ARM Cortex-M3, Jean J. Labrosse, Micrium, 2009


[책 링크](https://micrium.atlassian.net/wiki/spaces/osiiidoc/overview)
