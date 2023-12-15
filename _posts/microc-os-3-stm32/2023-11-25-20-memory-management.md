---
title: "μC/OS-III ch.17 Memory Management"
last_modified_at: 2023-11-25T22:18:12+09:00
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

애플리케이션은 ANSI C 컴파일러의 malloc()과 free()함수를 사용하여 메모리를 동적으로 할당하고 해제할 수 있다. 그러나 임베디드 실시간 시스템에서 malloc()과 free()함수를 사용하는 것은 위험할 수 있다. 왜냐하면 단편화(fragmentation)로 인해 연속된 단일 메모리 영역을 얻는 것이 불가능해질 수 있기 때문이다. 단편화는 많은 수로 나눠진 남은 공간(즉, 전체 남은 공간은 작고 연속되지 않은 조각들로 단편화된다)으로 되는 것을 말한다. malloc()과 free()의 실행시간은 malloc()요청을 만족시킬 만큼 충분히 큰, 메모리의 남은 연속 블록을 찾는데 사용되는 알고리즘을 사용할 떄 일반적으로 비결정적이다.

μC/OS-III는 Fig 17-1과 같이 연속적인 메모리 영역으로 만들어진 파티션에서 애플리케이션이 고정된 크기의 메모리 블록을 얻을 수 있도록 함으로써 malloc()과 free()를 대신한다. 모든 메모리 블록은 동일한 크기이며 파티션은 일정한 수의 블록을 포함한다. 이러한 메모리 블록의 할당과 할당 해제는 일정한 시간동안 수행되며 결정론적이다. 파티션 자체는 일반적으로 정적으로(배열로) 할당되지만, 해제하지 않는다면 malloc()을 사용하여 할당할 수도 있다.

![Figure 17-1 Memory Partition](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/335882ac-384d-46f4-adf3-fc693b6a35e1)

Fig17-2에 표시된 바와 같이, 애플리케이션에는 둘 이상의 메모리 파티션이 존재할 수 있고, 각각의 메모리 블록의 개수가 다르고 크기도 다를 수 있다. 애플리케이션은 다른 크기의 메모리 블록을 얻을 수 있다. 그러나 메모리 블록은 항상 해당 파티션으로 반환되어야 한다. 이러한 유형의 메모리 관리는 메모리 블록의 소진이 가능하지만, 조각화는 일어나지 않는다. 얼마나 많은 파티션을 가질 것인지, 파티션 내의 메모리 블록 크기를 결정하는 것은 애플리케이션이 결정한다.

![Figure 17-2 Multiple Memory Partitions](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/1177beb1-3ce7-4528-90cc-e5ec87d3e8ee)

# Creating A Memory Partition
메모리 파티션은 사용하기 전에 먼저 생성하여야 한다. 이를 통해 μC/OS-III가 해당 메모리 파티션에 대해 알 수 있게 되어 메모리 파티션의 할당과 할당 해제를 관리할 수 있게 된다. 일단 생성된 메모리 파티션은 Fig17-3과 같다. OSMemCreate()를 호출하면 메모리 파티션이 생성된다.

![Figure 17-3 Created Memory Partition](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/4e4b369c-0374-4c2b-8221-046966eb8b7d)

## F17-3(1)
파티션을 만들 때 애플리케이션 코드는 메모리 파티션 제어 블록(memory partition control block, OS_MEM)의 주소를 제공해야 한다. 보통 이 메모리 제어 블록은 정적으로 할당하지만, malloc()을 호출하여 heap영역으로부터 얻을 수도 있다. 그러나 애플리케이션 코드는 절대 할당 해제해서는 안된다.

## F17-3(2)
OSMemCreate()는 제공된 연속 메모리를 single linked list로 구성하고 리스트의 시작 부분에 대한 포인터를 OS_MEM에 저장한다.

## F17-3(3)
각각의 메모리 블록은 포인터를 저장할 수 있을 정도로 커야한다. linked list의 특성을 고려할 때, 블록은 다음 블록을 가리킬 수 있어야 한다.

# Creating A Memory Partition(2)

L17-1은 μC/OS-III에서 메모리 파티션을 어떻게 생성하는지를 보여준다.

```c
OS_MEM MyPartition; (1)
CPU_INT08U MyPartitionStorage[12][100]; (2)

void main (void) (3)
{
    OS_ERR err;
    :
    :
    OSInit(&err);
    :
    OSMemCreate((OS_MEM *)&MyPartition, (4)
                (CPU_CHAR *)"My Partition", (5)
                (void *)&MyPartitionStorage[0][0], (6)
                (OS_MEM_QTY ) 12, (7)
                (OS_MEM_SIZE)100, (8)
                (OS_ERR *)&err); (9)
    /* Check ’err’ */
    :
    :
    OSStart(&err);
}
//L17-1 Creating a memory partition
```

## L17-1(1)
애플리케이션은 메모리 파티션 제어 블록(즉, OS_MEM)을 할당할 필요가 있다. 여기에 나타난 바와 같이 정적 할당일 수도 있고, malloc()을 사용할 수도 있다. 다만, 애플리케이션 코드는 메모리 제어 블록을 할당 해제해서는 안된다.

## L17-1(2)
애플리케이션은 메모리 블록들로 분할될 메모리에 대한 저장공간을 할당할 필요가 있다. 정적 할당일 수도 있고, malloc()을 사용할 수도 있다. 다른 task가 이 저장공간에 의존할 수 있으므로, 할당 해제하지 않아야 한다.

## L17-1(3)
메모리 파티션은 파티션에서 블록을 할당하고 할당을 해제하기 전에 생성해야 한다. 메모리 파티션을 생성하기 위한 가장 좋은 장소 중 하나는 멀티태스킹 프로세스를 시작하기 전, 즉, main()에 있다. 실제로 코드를 main()에 배치할 수도 있고, main()에서 함수를 호출하여 이를 수행할 수도 있다.

## L17-1(4)
메모리 파티션 제어 블록의 주소를 OSMemCreate()에 전달한다. OS_MEM 자료구조의 내부 멤버 중 어떤 것도 참조해서는 절대 안 된다. 대신 항상 μC/OS-III의 API를 사용해야 한다.

## L17-1(5)
메모리 파티션에 이름을 지정할 수 있다. μC/OS-III는 실제 문자가 아닌 ASCII 문자열의 포인터를 파티션 제어 블록에 저장하므로 ASCII 문자열의 길이에는 제한이 없다.

## L17-1(6)
그런 다음 메모리 블록을 위해 할당한 저장공간의 주소를 전달해야한다.

## L17-1(7)
여기서 메모리 파티션에서 사용할 수 있는 메모리 블록 수를 지정해야 한다. hard-coded 숫자보다는, define constant를 사용해야 한다.

## L17-1(8)
각 메모리 블록의 크기를 지정한다. hard-coded 숫자값으로 지정하는 것은 권장되지 않는다.

## L17-1(9)
대부분의 μC/OS-III 서비스와 마찬가지로 OSMemCreate()는 호출 결과를 나타내는 오류 코드를 반환한다. err가 OS_ERR_NONE이면 성공한 것이다.

# Creating A Memory Partition(3)
L17-2는 μC/OS-III로 메모리 파티션을 만드는 방법을 보여주지만, 이번에는 malloc()을 사용하여 저장공간을 할당한다. 메모리 제어 블록이나 저장공간을 할당 해제하지 않아야 한다.

```c
OS_MEM *MyPartitionPtr; (1)

void main (void)
{
    OS_ERR err;
    void *p_storage;
    :
    OSInit(&err);
    :
    MyPartitionPtr = (OS_MEM *)malloc(sizeof(OS_MEM)); (2)
    if (MyPartitionPtr != (OS_MEM *)0) {
        p_storage = malloc(12 * 100); (3)
        if (p_storage != (void *)0) {
            OSMemCreate((OS_MEM *)MyPartitionPtr, (4)
                        (CPU_CHAR *)"My Partition",
                        (void *)p_storage, (5)
                        (OS_MEM_QTY ) 12, (6)
                        (OS_MEM_SIZE)100, (6)
                        (OS_ERR *)&err);
                        /* Check ’err” */
        }
    }
    :
    OSStart(&err);
}
//L17-2 Creating a memory partition
```

## L17-2(1)
메모리 파티션 제어 블록을 정적으로 할당하는 대신 malloc()을 사용하여 동적으로 할당할 수 있다.

## L17-2(2)
애플리케이션이 메모리 제어 블록을 위한 공간을 할당한다.

## L17-2(3)
메모리 파티션을 위한 공간을 할당한다.

## L17-2(4)
할당된 메모리 제어 블록의 포인터를 OSMemCreate()로 전달한다.

## L17-2(5)
파티션으로 쓰일 저장공간의 주소를 OSMemCreate()로 전달한다.

## L17-2(6)
마지막으로 블록의 개수와 각 블록의 크기를 전달하여 μC/OS-III가 각각 100바이트씩 12개의 블록으로 구성된 linked list를 구현하게 한다. 여기서 hard-coded 숫자를 사용하지만, define을 사용하는 것이 좋다.

# Getting A Memory Block From A Partition
애플리케이션 코드는 L17-3과 같이 OSMemGet()을 호출하여 파티션으로부터 메모리 블록을 요청할 수 있다. 파티션은 이미 만들어진 것으로 가정하자.

```c
OS_MEM MyPartition; (1)
CPU_INT08U *MyDataBlkPtr;

void MyTask (void *p_arg)
{
    OS_ERR err;
    :
    while (DEF_ON) {
        :
        MyDataBlkPtr = (CPU_INT08U *)OSMemGet((OS_MEM *)&MyPartition, (2)
                                                (OS_ERR *)&err);
        if (err == OS_ERR_NONE) { (3)
            /* You have a memory block from the partition */
        }
        :
        :
    }
}
//L17-3 Obtaining a memory block from a partition
```

## L17-3(1)
메모리 파티션 제어 블록은 파티션을 사용할 모든 task 또는 ISR에서 엑세스할 수 있어야 한다.

## L17-3(2)
원하는 파티션에서 메모리 블록을 얻기 위해 OSMemGet()을 호출하기만 하면 된다. 할당된 메모리 블록에 대한 포인터가 반환된다. 이것은 malloc()과 비슷하지만, 메모리 블록이 조각나지 않도록 보장된 풀에서 나온다는 점이 다르다. 응용 프로그램은 각 블록의 크기를 알고 있으므로, 블록을 overflow시키지 않는다고 가정한다.

## L17-3(3)
반환된 오류 코드를 검사하여 남은 메모리 블록이 있는지, 애플리케이션이 그 메모리 블록에 값을 넣을 수 있는지 확인하는 것이 중요하다.

# Returning A Memory Block To A Partition
애플리케이션 코드는 할당된 메모리 블록 사용이 끝나면 다시 해당 파티션으로 되돌려 놓아야 한다. 이것은 L17-4에 나와있는 것처럼 OSMemPut()을 호출하여 수행한다. 파티션은 이미 생성되었다고 가정한다.

```c
OS_MEM MyPartition; (1)
CPU_INT08U *MyDataBlkPtr;

void MyTask (void *p_arg)
{
    OS_ERR err;
    :
    while (DEF_ON) {
        :
        OSMemPut((OS_MEM *)&MyPartition, (2)
                    (void *)MyDataBlkPtr, (3)
                    (OS_ERR *)&err);
        if (err == OS_ERR_NONE) { (4)
            /* You properly returned the memory block to the partition */
        }
        :
        :
    }
}
```

## L17-4(1)
메모리 파티션 제어 블록은 파티션을 사용할 모든 task 또는 ISR에서 접근가능해야한다.

## L17-4(2)
OSMemPut()을 호출하여 메모리 블록을 해당 파티션으로 반환할 수 있다. 적절한 메모리 블록이(여러 개의 다른 파티션이 있다고 가정할 때) 적절한 파티션으로 반환되는지 확인하지 않는다는 점에 유의한다. 따라서 임베디드 시스템을 설계할 때 주의해야 한다.

## L17-4(3)
할당된 데이터 블록의 포인터를 전달하여 풀로 반환할 수 있다. void 포인터로 반환한다는 점에 유의한다.

## L17-4(4)
호출이 성공했는지 반환된 오류 코드를 확인할 수 있다.

# Using Memory Partition
메모리 관리 서비스는 os_cfg.h에 OS_CFG_MEM_EN을 1로 설정하여 활성화할 수 있다.

표17-1에 요약된 바와 같이 메모리 파티션과 관련된 여러가지 함수들이 있다.

| Function Name | Operation |
|---------------|-----------|
| OSMemCreate() |메모리 파티션을 생성한다.|
| OSMemGet()    |메모리 파티션으로 부터 메모리 블록을 얻는다.|
| OSMemPut()    |메모리 블록을 메모리 파티션으로 반환한다.|

(표17-1 Memory Partition API summary)

OSMemCreate()는 task에서만 호출할 수 있지만, OSMemGet()과 OSMemPut()은 ISR에서도 호출할 수 있다.

L17-4는 μC/OS-III의 동적 메모리 할당 기능과 메시지 전달 기능(309 페이지 Chapter 15, "Message Passing" 참조)을 사용하는 예를 보여준다. 이 예제에서 왼쪽의 task가 아날로그 입력(압력, 온도, 전압)의 값을 읽고 체크하며, 아날로그 입력 중 어느 하나가 임계값을 초과하면 두 번쨰 task로 메시지를 보낸다. 전송되는 메시지는 어느 채널에 오류가 있었는지에 대한 정보, 오류 코드, 오류의 심각성을 나타내는 정보 및 기타 정보가 포함된다.

이 예제에서 오류 처리는 중앙 집중식이다. 다른 task들 및 ISR도 오류 처리 task에 오류 메시지를 post할 수 있다. 오류 처리 task는 모니터(디스플레이)상에 에러 메시지를 표시하거나, 오류를 디스크에 기록하거나, 오류를 해결하는 task를 깨우는 것을 담당할 수 있다.

![Figure 17-4 Using a Memory Partition – non blocking](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/36429ddb-def0-4230-b675-f25ef06507f6)

## F17-4(1)
아날로그 입력은 task가 읽는다. task는 입력값들 중 하나가 범위 밖에 있다는 것을 알게 되고 오류 메시지를 오류 핸들러에게 보낼 필요가 있다고 결정한다.

## F17-4(2)
그런 다음 task는 메모리 파티션으로부터 메모리 블록을 획득하여 오류 관련 정보를 저장할 수 있다.

## F17-4(3)
task는 관련 정보를 메모리 블록에 기록한다. task는 결함이 있는 아날로그 채널, 오류 코드, 심각도 표시, 해결책 등을 기록한다. 타임스탬프는 저장할 필요가 없는데, 타임스탬프는 μC/OS-III의 내장된 기능이므로, 메시지를 수신한 task는 메시지가 언제 post되었는지 알 수 있다.

## F17-4(4)
메시지를 기록하면, 그러한 오류 메시지를 처리할 task에 post한다. 물론 수신 task는 정보가 메시지 내에 어떻게 표시되어있는지 알아야한다. 일단 메시지가 전송되면 아날로그 입력 task는 메모리 블록이 전송되었기 때문에 더 이상 접근할 수 없다(실제로 접근할 수 없는 것은 아니지만, 접근하지 않는 것이 좋다).

## F17-4(5)
오류 핸들러 task(오른쪽)은 보통 메시지 큐를 대기한다. 이 task는 메시지를 전송받을 때까지 실행되지 않는다.

## F17-4(6)
메시지가 수신되면, 오류 핸들러 task는 메시지의 내용을 보고 필요한 동작을 수행한다. 표시된 바와 같이, 일단 송신되면, 송신자는 메시지와 관련해 어떠한 동작도 하지 않을 것이다.

## F17-4(7)
일단 오류 핸들러 task가 메시지를 처리하는 것을 완료하면, 메모리 블록을 메모리 파티션으로 반환한다. 따라서 송신자와 수신자는 메모리 파티션에 대해 알아야 하고, 송신자는 메모리 파티션의 주소를 메시지의 일부로써 전달할 수 있어야 오류 핸들러 task는 메모리 블록을 반환할 위치를 알 수 있을 것이다.

# Using Memory Partition(2)
가끔은 파티션의 블록이 부족할 경우를 대비하여 task가 메모리 블록을 기다리게 하는 것이 좋다. μC/OS-III는 파티션에 대기하는 것을 지원하지 않지만, 메모리 파티션을 보호하기 위해, 카운팅 세마포어(231페이지 Chapter 13, "Resource Management" 참조)를 추가함으로써 이를 지원할 수 있다. 카운팅 세마포어의 초기값은 파티션의 블록 개수로 설정될 것이다. 이를 Fig17-5에 나타냈다.

![Figure 17-5 Using a Memory Partition - blocking](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/0cd0ba0f-7d43-412d-b38a-59511dec8abf)

## F17-5(1)
메모리 블록을 얻기 위해, OSSemPend()를 호출하여 세마포어를 얻은 후 OSMemGet()을 호출하여 메모리 블록을 받는다.

## F17-5(2)
블록을 반환하려면 OSMemPut()을 호출하여 메모리 블록을 반환한 다음, OSSemPost()를 호출하여 세마포어를 시그널링한다.

위의 작업은 순서대로 수행되어야 한다.

ISR이 OSMemGet()과 OSMemPut()을 호출할 수 있지만(OSMemGet()과 OSMemPut()은 blocking 되지 않으며 빠르게 수행된다), ISR에서 blocking call을 할 수 없다(즉, Fig17-5와 같이 카운팅 세마포어를 쓸 수 없다).

# Summary
단편화가 일어날 수 있으므로, 임베디드 시스템에서 malloc()과 free()를 사용하면 안된다. 또는 malloc()을 사용하되, 할당을 해제하면 안된다.

애플리케이션 프로그래머는 무제한의 메모리 파티션을 만들 수 있다(사용가능한 RAM 양에만 제한됨).

μC/OS-III의 메모리 파티션 서비스는 OSMem??() 접두사로 시작하며, 애플리케이션 프로그래머가 사용할 수 있는 서비스는 443페이지의 Appendix A, "μC/OS-III API Reference" 에 설명되어 있다.

메모리 관리 서비스는 os_cfg.h에서 OS_CFG_MEM_EN을 1로 설정하여 활성화할 수 있다.

OSMemGet()과 OSMemPut()은 ISR이 호출할 수 있다.

# Reference
 - uC/OS-III: The Real-Time Kernel For the STM32 ARM Cortex-M3, Jean J. Labrosse, Micrium, 2009

[책 링크](https://micrium.atlassian.net/wiki/spaces/osiiidoc/overview)
