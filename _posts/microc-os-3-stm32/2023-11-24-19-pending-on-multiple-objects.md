---
title: "μC/OS-III ch.16 Pending On Multiple Objects"
last_modified_at: 2023-11-24T11:18:12+09:00
categories:
    - microc-os-3-stm32
tags:
    - microc-os
    - embedded-system

toc: true
toc_label: "My Table of Contents"
author_profile: true

---
197페이지의 Chapter 10 "Pend Lists(or Wait Lists)"에서 어떻게 여러 task가 세마포어, 상호 배제 세마포어(mutex), 이벤트 플래그 그룹 또는 메시지 큐와 같은 단일 커널 객체에 대기할 수 있는지를 보았다. 이 장에서는 어떻게 task가 여러 객체에 대기할 수 있는지를 볼 것이다. 그러나 μC/OS-III는 여러 세마포어 및/또는 메시지 큐에만 대기를 허용한다. 즉, 여러 이벤트 플래그 그룹 또는 상호 배제 세마포어에 대기할 수 없다.

Fig16-1과 같이 task는 여러 세마포어나 여러 메시지 큐에 동시에 대기할 수 있다. post된 첫 번째 세마포어나 메시지 큐는 task를 실행 준비 상태로 만들고 ready list의 다른 task와 CPU 시간을 경쟁한다. task는 OSPendMulti()를 호출하여 여러 객체에 대기하고, 선택적으로 타임아웃 값을 지정한다. 타임아웃은 모든 객체에 적용된다. 지정된 타임아웃 내에 객체들 중 어느 것도 post되지 않으면 타임아웃되었음을 나타내는 오류 코드와 함께 task가 재개된다.

# Task Pending On Multiple Objects

![Figure 16-1 Task pending on multiple objects](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/027dfb88-d105-4411-a4db-0c04b89209a6)

표16-1은 OSPendMulti()의 함수 프로토타입을 보여주고, Fig16-2는 OS_PEND_DATA의 배열을 보여준다.

```c
OS_OBJ_QTY OSPendMulti (OS_PEND_DATA    *p_pend_data_tbl, (1)
                        OS_OBJ_QTY      tbl_size, (2)
                        OS_TICK         timeout, (3)
                        OS_OPT          opt, (4)
                        OS_ERR          *p_err); (5)

//Table 16-1 OSPendMulti() prototype
```
![Figure 16-2 Array of OS_PEND_DATA](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/c2103043-7aa8-452f-8db2-e8dbf4e15a98)

## L16-0(1)
OSPendMulti()는 OS_PEND_DATA 배열을 전달받는다. 호출자는 OS_PEND_DATA 배열을 만들어야 한다. 배열의 크기는 task가 대기할 커널 객체의 종 개수에 따라 다르다. 예를 들어, task가 3개의 세마포어와 2개의 메시지 큐에 대기할 예정이라면, 배열은 다음과 같이 5개의 OS_PEND_DATA로 구성되어 있다.

```c
OS_PEND_DATA my_pend_multi_tbl[5];
```
호출자는 대기할 각 객체를 가리키기 위해 배열의 각 요소의 .PendObjPtr을 초기화해야한다. 예를 들어 다음과 같다:
```c
OS_SEM MySem1;
OS_SEM MySem2;
OS_SEM MySem3;
OS_Q MyQ1;
OS_Q MyQ2;

void MyTask (void)
{
    OS_ERR err;
    OS_PEND_DATA my_pend_multi_tbl[5];
    :
    while (DEF_ON) {
        :
        my_pend_multi_tbl[0].PendObjPtr = (OS_PEND_OBJ)&MySem1; (6)
        my_pend_multi_tbl[1].PendObjPtr = (OS_PEND_OBJ)&MySem2;
        my_pend_multi_tbl[2].PendObjPtr = (OS_PEND_OBJ)&MySem3;
        my_pend_multi_tbl[3].PendObjPtr = (OS_PEND_OBJ)&MyQ1;
        my_pend_multi_tbl[4].PendObjPtr = (OS_PEND_OBJ)&MyQ2;
        OSPendMulti((OS_PEND_DATA *)&my_pend_multi_tbl[0],
                    (OS_OBJ_QTY )5,
                    (OS_TICK )0,
                    (OS_OPT )OS_OPT_PEND_BLOCKING,
                    (OS_ERR *)&err);
        /* Check ’err” */
        :
    }
}
```
## L16-0(2)
이 argument는 OS_PEND_DATA 테이블의 크기를 지정한다. 위 예제에서 5이다.

## L16-0(3)
타임아웃 값을 지정하여, 특정 시간 내에 어떤 객체도 post되지 않을 경우 대기를 중단하고 ready list로 들어갈 수 있다. 0이 아닌 값은 타임아웃할 틱 수를 나타낸다. 0을 지정하면 객체 중 어느 하나가 post될 때까지 영원히 대기할 것임을 나타낸다.

## L16-0(4)
"opt" argument는 객체가 post될 때까지 기다릴 것인지(opt를 OS_OPT_PEND_BLOCKING으로 설정) 아니면 객체가 post되지 않았을 때 기다리지 않고 바로 OSPendMulti()함수가 리턴되게할 것인지(opt를 OS_OPT_PEND_NON_BLOCKING으로 설정)를 지정한다.

## F16-2(1)
대부분의 μC/OS-III 함수 호출과 마찬가지로 함수 호출의 결과에 따라 생성되는 오류 코드를 저장할 변수의 주소를 지정한다. 가능한 오류 코드 목록은 443페이지의 Appendix A "μC/OS-III API Reference"를 참조하라. 항상 그렇듯이 반환된 오류 코드를 검사하는 것이 매우 권장된다.

## F16-2(2)
모든 객체는 OS_PEND_OBJ 자료형으로 형변환된다.

# Task Pending On Multiple Objects(2)
호출 시 OSPendMulti()는 먼저 OS_PEND_DATA 테이블에 지정된 모든 객체가 OS_SEM 또는 OS_Q임을 확인하는 것으로 시작한다. 그렇지 않으면 오류 코드를 반환한다.

다음으로, OSPendMulti()는 OS_PEND_DATA 테이블을 통해 객체 중 이미 post된 객체가 있는지 확인한다. 만약 있다면 OSPendMulti()는 테이블에서 .RdyObjPtr, .RdyMsgPtr, .RdyMsgSize 및 .RdyTS 필드를 채운다.

- .RdyObjPtr: 객체가 post된 경우 객체에 대한 포인터이다. 예를 들어 테이블의 첫 번째 객체가 세마포어이고 세마포어가 post된 경우 my_pend_multi_tbl[0].RdyObjPtr는 my_pend_multi_tbl[0].PendObjPtr로 설정된다.

- .RdyMsgPtr: 이 엔트리에 있는 객체가 메시지 큐이고 메시지 큐에서 메시지를 수신한 경우 메시지에 대한 포인터이다.

- .RdyMsgSize: 이 엔트리에 있는 객체가 메시지 큐이고 메시지 큐에서 메시지를 수신한 경우 메시지의 크기이다.

- .RdyTS: 객체가 post되었을 때 타임스탬프 값이다. 이것은 세마포어나 메시지 큐가 언제 post되었는지 알게 해준다.

post된 객체가 없으면 OSPendMulti()는 현재 task를 대기하는 모든 객체의 wait list에 넣는다. OSPendMulti()의 경우 대기 중인 객체 중 일부의 pend list에 다른 task가 있을 수도 있기 때문에 이 과정은 복잡하고 지루한 과정이다.

Fig16-3은 두 개의 세마포어에 대기 중인 task의 예로, 얼마나 처리하기 까다로워지는지를 보여준다.

![Figure 16-3 Task pending on two semaphores](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/17105034-719a-4c05-8e7c-fb4ae18230f2)

## F16-3(1)
OS_PEND_DATA 테이블의 기본 주소에 대한 포인터는 두 세마포어의 pend list에 배치된 task의 OS_TCB에 있다.

## F16-3(2)
OS_PEND_DATA 테이블 내의 엔트리들의 수도 OS_TCB에 저장된다. 이 task는 2개의 세마포어를 대기하고 있고, 따라서 테이블 내에 2개의 엔트리가 존재한다.

## F16-3(3)
첫번째 세마포어는 OS_PEND_DATA의 첫번째 엔트리와 연결되어있다.

## F16-3(4)
OS_PEND_DATA[0]은 해당 엔트리의 .PendObjPtr에 의해 세마포어 객체와 연결되어있고, 해당 세마포어의 포인터는 OSPendMulti()를 호출할 때 지정된다.

## F16-3(5)
세마포어의 pend list에 한 개의 task만 있으므로, .PrevPtr와 .NextPtr은 NULL이다.

## F16-3(6)
두번째 세마포어는 OS_PEND_DATA의 두번째 엔트리를 가리킨다.

## F16-3(7)
OS_PEND_DATA의 두번째 엔트리는 두번째 세마포어를 가리킨다. 두번째 세마포어의 포인터는 OSPendMulti()를 호출할 때 지정된다.

## F16-3(8)
두번째 세마포어는 pend list에 한개의 항목만 있으므로, .PrevPtr와 .NextPtr은 NULL이다.

## F16-3(9)
OSPendMulti()는 각 OS_PEND_DATA 엔트리를 두 세마포어를 기다리는 task와 연결시킨다.

# Task Pending On Multiple Objects(3)
Fig16-4는 하나의 task가 두 개의 세마포어에 대기중이고 다른 task도 두 개의 세마포어 중 하나에 대기하고 있는 좀더 복잡한 예이다. 예시는 세마포어만을 보여주지만, 세마포어와 메시지 큐의 조합도 가능하다.

![Figure 16-4 Tasks pending on semaphores](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/1038b2a3-4b20-45ae-986b-788be0d9e0b9)

ISR이나 task중 하나가 task가 보류 중인 객체 중 하나에 신호를 보내거나 메시지를 보내면 OSPendMulti()가 리턴되어 어떤 객체가 post되었는지를 OS_PEND_DATA에 표시한다. 이것은 post된 객체의 .RdyObjPtr만 해당 객체를 가리키게 하여 수행된다.

OS_PEND_DATA의 엔트리들 중 하나만 NULL이 아닌 .RdyObjPtr을 가지게 되며, 나머지 엔트리들은 모두 .RdyObjPtr이 NULL로 설정된다. task가 5개의 세마포어와 2개의 메시지 큐에서 대기하는 경우로 돌아가보면, task가 모든 객체에 대기 중이 ㄴ상태에서 첫 번째 메시지 큐가 post되면 OS_PEND_DATA는 Fig16-5와 같게 된다.

![Figure 16-5 Message queue 1 posted before timeout expired](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/c1570d00-4cc7-4633-921d-98362b03d823)

# Summary
μC/OS-III는 task가 여러 커널 객체에 대기할 수 있게 한다.

OSPendMulti()는 여러 세마포어 및 메시지 큐에만 대기할 수 있다(이벤트 플래그 및 상호 제외 세마포어는 불가능).

OSPendMulti()가 호출될 때 객체들이 이미 post된 경우, μC/OS-III는 객체들 중 어느 것(둘 이상일 수 있음)이 이미 post되었는지를 지정한다.

객체들 중 아무것도 post되지 않았다면, OSPendMulti()는 대기를 원하는 객체들의 pend list에 해당 task를 배치할 것이다. 객체들 중 하나가 post되는 즉시 OSPendMulti()는 리턴된다. 이 경우, OSPendMulti()는 어떤 객체가 post되었는지를 나타낼 것이다.

OSPendMulti()는 긴 임계 구역을 갖는 복잡한 함수이다.

# Reference
 - uC/OS-III: The Real-Time Kernel For the STM32 ARM Cortex-M3, Jean J. Labrosse, Micrium, 2009

[책 링크](https://micrium.atlassian.net/wiki/spaces/osiiidoc/overview)