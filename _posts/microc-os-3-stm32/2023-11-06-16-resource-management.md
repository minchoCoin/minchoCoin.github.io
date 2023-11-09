---
title: "μC/OS-III ch.13 Resource Management"
last_modified_at: 2023-11-06T19:18:12+09:00
categories:
    - microc-os-3-stm32
tags:
    - microc-os
    - embedded-system

toc: true
toc_label: "My Table of Contents"
author_profile: true

---
이 장에서는 공유 자원을 관리하기 위해 μC/OS-III에 의해 제공되는 서비스들에 대해 논의할 것이다. 공유 자원은 전형적으로 변수(정적변수 또는 전역변수), 자료 구조, 테이블(RAM 내), 또는 I/O 디바이스 내의 레지스터들이다.

공유 자원을 보호할 때는, 이 장에서 설명될 것과 같이, 상호 배제 세마포어(mutual exclusion semaphore)를 사용하는 것이 바람직하다. 다른 방법들도 또한 제시될 것이다.

task는 모든 task가 하나의 주소 공간에 존재할 때 데이터를 쉽게 공유할 수 있으며 전역 변수, 포인터, 버퍼, 연결 리스트(linked list), 원형 버퍼(ring buffer) 등을 참조할 수 있다. 데이터를 공유하면 task 간 정보 교환이 간단해지지만, 각 task가 경쟁와 데이터 손상을 피하기 위해 데이터에 독점적 접근 권한을 갖도록 하는 것이 중요하다.

예를 들어, 간단한 Time-of-Day 알고리즘을 수행하는 모듈을 소프트웨어에 구현할 때, 모듈은 분명 시, 분, 초를 계속 기록한다. TimeOfDay() 작업은 L13-1과 같이 나타날 수 있다.

인터럽트가 발생하여 이 task가 다른 task에 의해 선점되었다고 생각하고, TimeOfDay() task가 분을 0으로 설정한 이후, 더 중요한 task가 생겼다고 하자. 이제 우선순위가 더 높은 task가 time-of-day 모듈로부터 현재 시간을 알고 싶다면 어떻게 될지 상상해보자. 인터럽트가 발생하기 전에 시간(hours)가 증가하지 않았으므로, 우선 순위가 더 높은 task는 부정확한 시간 값을 읽게 되고, 이 경우 1시간만큼 잘못 읽게 된다.

TimeOfDay() task에서 변수를 업데이트하는 코드는 선점이 있어도 모든 변수를 원자적으로 처리해야한다. Time-of-day 변수는 공유 자원으로 간주되며 해당 변수에 접근하는 모든 코드는 임계 구역이라고 불리는 것을 통해 배타적 접근권을 가져야 한다. μC/OS-III는 공유 자원을 보호하는 서비스를 제공하고 임계 구역을 쉽게 만들 수 있게 해준다.

```c
CPU_INT08U Hours;
CPU_INT08U Minutes;
CPU_INT08U Seconds;

void TimeOfDay (void *p_arg)
{
    OS_ERR err;
    (void)&p_arg;
    while (DEF_ON) {
        OSTimeDlyHMSM(0,
                        0,
                        1,
                        0,
                        OS_OPT_TIME_HMSM_STRICT,
                        &err);
        /* Examine “err” to make sure the call was successful */
        Seconds++;
        if (Seconds > 59) {
            Seconds = 0;
            Minutes++;
            if (Minutes > 59) {
                Minutes = 0;
                Hours++;
                if (Hours > 23) {
                    Hours = 0;
                }
            }
        }
    }
}
//L13-1 Faulty Time-Of-Day clock task
```
공유 자원에 대한 배타적 접근권을 획득하고 임계 구역을 만드는 가장 일반적인 방법은 다음과 같다:

- 인터럽트 비활성화
- 스케줄러 비활성화
- 세마포어 사용
- 상호 배제 세마포어(mutex) 사용

사용되는 상호 배제 방법은 표 13-1과 같이 코드가 공유 자원에 얼마나 빨리 접근할 것인가에 달려있다.

|자원 공유 방법   |언제 써야 되나?   |
|---|---|
|인터럽트 활성화/비활성화   |공유 자원에 대한 접근이 매우 빠르고(몇 개의 변수를 읽기 또는 쓰기), 접근이 μC/OS-III의 인터럽트 비활성 시간보다 빠른 경우. interrupt latency 시간에 영향을 미치므로 이 방법을 사용하지 않는 것이 좋습니다. |
|스케줄러 잠그기/풀기   |공유 자원에 대한 접근 시간이 μC/OS-III의 인터럽트 비활성화 시간보다 길지만 μC/OS-III의 스케줄러 잠금 시간보다 짧은 경우. 스케줄러를 잠그는 것은 스케줄러를 잠그는 작업을 우선 순위가 가장 높은 작업으로 만드는 것과 동일한 효과가 있다. 이 방법은 μC/OS-III를 사용하는 목적을 달성하지 못하므로 사용하지 않는 것이 좋다. 그러나 interrupt latency에 영향을 주지 않으므로 인터럽트를 비활성화하는 것보다 더 나은 방법이다. |
|세마포어   |공유 자원에 접근할 필요가 있는 모든 task가 기한(deadline)을 갖지 않을 때. 이것은 세마포어가 우선순위 역전(unbounded priority inversion, 후술)을 발생시킬 수 있기 때문이다. 그러나 세마포어는 상호 배제 세마포어보다 실행시간 측면에서 약간 더 빠르다. |
|상호 배제 세마포어   |공유 자원에 접근해야하는 task에 기한(deadline)이 있는 경우 선호되는 방법이다.  μC/OS-III의 상호 배제 세마포어에는 우선 순위 역전(unbounded priority inversion)을 방지하는 우선 순위 상속 메커니즘이 내장되어 있다.  그러나 우선순위를 변경(CPU가 처리 필요)해야할 수도 있기 때문에 상호 배제 세마포어는 세마포어에 비해 약간 느리다(실행 시간 측면에서).|

# Disable/Enable Interrupts (1)
공유 자원에 대한 독점적 접근을 얻는 가장 쉽고 빠른 방법은 L13-2의 의사 코드와 같이 인터럽트를 비활성화하고 활성화하는 것이다.

```c
Disable Interrupts;
Access the resource;
Enable Interrupts;
//L13-2 Disabling and Enabling Interrupts
```

μC/OS-III는 특정 내부 변수와 자료 구조에 접근하기 위해 (모두는 아닐지 몰라도) 이 기법을 사용하여 일부 변수와 자료 구조가 원자적으로 조작됨을 보장한다. 그러나 인터럽트를 비활성화하고 활성화하는 것은 실제로는 OS 관련 함수와 CPU-specific 파일의 함수가 아니라 CPU 관련 함수이다(사용 중인 프로세서의 cpu.h 파일 참조). CPU 모듈에서 제공되는 서비스를 μC/CPU라고 한다. 각각의 다른 대상 CPU 아키텍처는 자체적인 μC/CPU 관련 파일 집합을 가지고 있다.

```c
void YourFunction (void)
{
    CPU_SR_ALLOC(); (1)

    CPU_CRITICAL_ENTER(); (2)
    Access the resource; (3)
    CPU_CRITICAL_EXIT(); (4)
}
//L13-3 Using CPU macros to disable and enable interrupts
```

## L13-3(1)
CPU_SR_ALLOC() 매크로는 인터럽트를 활성화/비활성화하는 다른 두 매크로가 사용될 때 필요하다. 이 매크로는 단순히 지역 변수를 저장하는 저장소를 할당하여 CPU의 현재 인터럽트 디스에이블 상태의 값을 유지한다.  인터럽트가 이미 비활성화되어 있다면 임계 구역을 나갈 때 인터럽트를 활성화시키지 않아야될 필요성이 있다.

## L13-3(2)
CPU_CRITICAL_ENTER()는 CPU_SR_ALLOC()가 할당한 지역 변수에 CPU interrupt disable 플래그의 현재 상태를 저장하고 마스크 가능한 모든 인터럽트를 사용 안 함으로 설정한다.

## L13-3(3)
코드의 임계 구역은 인터럽트가 비활성화되어 있기 때문에 ISR이나 다른 task에 의해 변경될 걱정 없이 접근할 수 있다. 다시 말해서, 이 연산은 원자적이다.

## L13-3(4)
CPU_CRITICAL_EXIT()은 이전에 저장된 CPU interrupt disable 상태를 지역 변수에서 복원한다.

# Disable/Enable Interrupts (2)
CPU_CRITICAL_ENTER() 와 CPU_CRITICAL_EXIT() 는 항상 쌍으로 사용된다. 인터럽트를 비활성화하면 인터럽트에 대한 시스템의 반응에 영향을 미치므로 인터럽트는 가능한 짧은 시간 동안 비활성화되어야 한다. 이를 interrupt latency라고 한다. 인터럽트 활성화/비활성화 방식은 몇 개의 변수를 변경하거나 복사할 때만 사용된다.

이것이 task가 ISR과 변수 또는 자료 구조를 공유할 수 있는 유일한 방법임에 유의한다.

μC/CPU는 interrupt latency를 실제로 측정하는 방법을 제공한다.

μC/OS-III을 사용할 때 interrupt latency에 영향을 주지 않으면서 μC/OS-III가 비활성화하는 시간 만큼 인터럽트가 비활성화될 수도 있다. 분명한 것은 μC/OS-III가 인터럽트를 비활성화하는 기간을 아는 것이 중요한데, 이는 사용되는 CPU에 의존한다.

이 방법은 작동하지만 실시간 이벤트에 대한 시스템의 응답성에 영향을 미치므로 인터럽트를 비활성화하지 않아야 한다.

# Lock/Unlock
task가 ISR과 변수 또는 자료구조를 공유하지 않는 경우 L13-4와 같이 자원에 접근하는 동안 μC/OS-III의 스케줄러를 비활성화 및 활성화할 수 있다.

```c
void YourFunction (void)
{
    OS_ERR err(); (1)

    OSSchedLock(&err); (2)
    Access the resource; (3)
    OSSchedUnlock(&err); (4)
}
//L13-4 Accessing a resource with the scheduler locked
```
이 방법을 사용하면 둘 이상의 task가 경쟁 상태 가능성 없이 데이터를 공유한다. 주으이할점은 스케줄러가 잠겨 있는 동안 인터럽트는 활성화되어있고, 임계 구역에 있는 동안 인터럽트가 발생하면 ISR이 실행된다. ISR이 끝나면 더 높은 우선순위의 task가 ISR에 의해 ready-to-run 상태로 만들어졌다고 하더라도 커널은 항상 인터럽트 때문에 중단된 task로 복귀한다. ISR이 인터럽트 때문에 중단되었던 task로 복귀하므로 커널의 동작은 (스케줄러가 잠겨있는 동안) 비선점 커널의 동작과 유사하다.

OSSchedLock()과 OSSchedUnlock()은 최대 250단계 깊이까지 중첩될 수 있다. 스케줄러는 애플리케이션에서 OSSchedUnlock() 호출 횟수와 애플리케이션에서 OSSchedLock() 호출 횟수가 동일할 때만 호출된다.

스케줄러의 잠금이 풀린 후, 우선순위가 높은 작업이 ready-to-run 상태가 된 경우 μC/OS-III는 문맥 교환을 수행한다.

μC/OS-III는 스케줄러가 잠겨 있을 때 사용자가 blocking call을 하지 못하게 한다. 만일 애플리케이션이 blocking call을 할 수 있었다면, 애플리케이션은 거의 실패했을 것이다.

이 방법은 잘 작동하지만, 선점적 커널을 갖는 목적과 맞지 않기 때문에 스케줄러를 비활성화하는 것을 하지 않을 수도 있다. 스케줄러를 잠그면 현재 task가 가장 우선순위가 높은 task가 된다.

# Semaphores
세마포어는 원래 원래 기계적 신호 전달 메커니즘(mechanical signaling mechanism)이었다. 철도 산업은 이 창치를 사용하여 둘 이상의 열차가 공유하는 철도 선로에 대해 상호 배제를 제공하였다. 세마포어는 현재 사용 중인 선로로부터 열차를 차단하기 위해 차단기를 내려 열차에 신호를 전달하였다. 선로가 사용가능해지면, 차단기를 올려 대기하고 있던 열차가 그 선로를 이용할 수 있게 된다.

소프트웨어에서 세마포어를 상호 배제의 수단으로 사용한다는 생각은 1959년 네덜란드의 컴퓨터 과학자 Edgser Dijkstra가 만들었다. 컴퓨터 소프트웨어에서 세마포어는 대부분 멀티태스킹 커널이 제공하는 프로토콜 매커니즘이다. 세마포어는 원래 공유 자원에 대한 접근을 제어하기 위해 사용되었으나, 현재는 14장 273페이지의 "Synchronization"에서 설명한 것처럼 동기화를 위해 사용된다. 그러나 세마포어가 자원을 공유하기 위해 어떻게 사용될 수 있는지를 설명하는 것은 유용하다. 세마포어의 단점은 나중에 논의할 것이다.

세마포어는 원래 lock mechanism이였고 코드는 계속 실행하기 위해 lock에 대한 key를 얻었다. key를 획득한다는 것은 획득하지 못하면 잠겨있는 구간에 들어갈 수 있는 권한을 획득한다는 것을 의미한다. 잠겨 있는 구간에 들어가려고 시도하면, task가 키를 획득할 수 있을 때까지 대기하게 된다.

일반적으로, 이진 세마포어와 카운팅 세마포어 두 가지 유형이 존재한다. 이름에서 알 수 있듯, 이진 세마포어는 단지 0 또는 1 두 가지 값만을 가질 수 있다. 카운팅 세마포어는 8비트, 16비트, 32비트 중 어느 것으로 구현되었는지에 따라 0부터 255 사이의 값, 0부터 65535 사이의 값, 0부터 4294967295를 사용할 수 있다. μC/OS-III의 경우, 세마포어의 최댓값은 OS_SEM_CTR(os_type.h 참조)에 의해 결정되며, 이는 필요에 따라 변경될 수 있다. μC/OS-III은 세마포어의 값과 함께 세마포어를 대기하는 task를 추적하기도 한다.

자원을 공유하기 위해 세마포어를 사용하는 경우 task만 세마포어를 사용할 수 있으며, ISR은 사용할 수 없다.

세마포어는 OS_SEM 타입으로 정의되는 커널 객체로서, os_sem 구조(os.h 참조)로 정의된다. 애플리케이션은 임의의 개수의 세마포어를 가질 수 있다(사용 가능한 RAM의 양에만 제한됨).

표 13-2에 정리된 것과 같이 세마포어에 대해 애플리케이션이 수행할 수 있는 연산이 많이 있다. 이 장에서는 가장 많이 사용되는 함수인 OSSemCreate(), OSSemPend(), OSSemPost() 세 가지에 대해서만 논의한다. 다른 함수들은 443페이지 부록 A의 "μC/OS-III API Reference"에 설명되어있다. 세마포어가 자원 공유를 위해 사용될 때 모든 세마포어 함수는 task에서 호출되어야하며 ISR에서 호출되지 않아야한다. 이후 13장에서 설명되는 것과 같이 시그널링을 위해 세마포어를 사용할 때는 동일한 제한이 적용되지 않는다.