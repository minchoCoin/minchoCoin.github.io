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
애플리케이션은 ANSI C 컴파일러의 malloc()과 free()함수를 사용하여 메모리를 동적으로 할당하고 해제할 수 있다. 그러나 임베디드 실시간 시스템에서 malloc()과 free()함수를 사용하는 것은 위험할 수 있다. 왜냐하면 단편화(fragmentation)로 인해 연속된 단일 메모리 영역을 얻는 것이 불가능해질 수 있기 때문이다. 단편화는 많은 수로 나눠진 남은 공간(즉, 전체 남은 공간은 작고 연속되지 않은 조각들로 단편화된다)으로 되는 것을 말한다. malloc()과 free()의 실행시간은 malloc()요청을 만족시킬 만큼 충분히 큰, 메모리의 남은 연속 블록을 찾는데 사용되는 알고리즘을 사용할 떄 일반적으로 비결정적이다.

μC/OS-III는 Fig 17-1과 같이 연속적인 메모리 영역으로 만들어진 파티션에서 애플리케이션이 고정된 크기의 메모리 블록을 얻을 수 있도록 함으로써 malloc()과 free()를 대신한다. 모든 메모리 블록은 동일한 크기이며 파티션은 일정한 수의 블록을 포함한다. 이러한 메모리 블록의 할당과 할당 해제는 일정한 시간동안 수행되며 결정론적이다. 파티션 자체는 일반적으로 정적으로(배열로) 할당되지만, 해제하지 않는다면 malloc()을 사용하여 할당할 수도 있다.

![Figure 17-1 Memory Partition](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/335882ac-384d-46f4-adf3-fc693b6a35e1)

Fig17-2에 표시된 바와 같이, 애플리케시녀에는 둘 이상의 메모리 파티션이 존재할 수 있고, 각각의 메모리 블록의 개수가 다르고 크기도 다를 수 있다. 애플리케이션은 다른 크기의 메모리 블록을 얻을 수 있다. 그러나 메모리 블록은 항상 해당 파티션으로 반환되어야 한다. 이러한 유형의 메모리 관리는 메모리 블록의 소진이 가능하지만, 조각화는 일어나지 않는다. 얼마나 많은 파티션을 가질 것인지, 파티션 내의 메모리 블록 크기를 결정하는 것은 애플리케이션이 결정한다.

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
# Reference
 - uC/OS-III: The Real-Time Kernel For the STM32 ARM Cortex-M3, Jean J. Labrosse, Micrium, 2009

[책 링크](https://micrium.atlassian.net/wiki/spaces/osiiidoc/overview)
