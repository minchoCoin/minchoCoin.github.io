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

세마포어는 OS_SEM 타입으로 정의되는 커널 객체로서, OS_SEM은 os_sem 구조체(os.h 참조)로 정의된다. 애플리케이션은 임의의 개수의 세마포어를 가질 수 있다(사용 가능한 RAM의 양에만 제한됨).

표 13-2에 정리된 것과 같이 세마포어에 대해 애플리케이션이 수행할 수 있는 연산이 많이 있다. 이 장에서는 가장 많이 사용되는 함수인 OSSemCreate(), OSSemPend(), OSSemPost() 세 가지에 대해서만 논의한다. 다른 함수들은 443페이지 부록 A의 "μC/OS-III API Reference"에 설명되어있다. 세마포어가 자원 공유를 위해 사용될 때 모든 세마포어 함수는 task에서 호출되어야하며 ISR에서 호출되지 않아야한다. 이후 13장에서 설명되는 것과 같이 시그널링을 위해 세마포어를 사용할 때는 동일한 제한이 적용되지 않는다.

| Function name | Operation |
|---------------|-----------|
|OSSemCreate()               |세마포어를 만든다.           |
|OSSemDel()               |세마포어를 삭제한다.          |
|OSSemPend()               |세마포어를 기다린다.           |
|OSSemPendAbort()               |세마포어를 기다리는 것을 중지한다.           |
|OSSemPost()               |세마포어를 방출하거나 시그널링한다.         |
|OSSemSet()               |세마포어 카운트를 원하는 값으로 설정한다.           |

(표 13-2 Semaphore API summary)

## Binary Semaphores
자원을 획득하고자 하는 task는 Wait(또는 Pend)를 실행해야한다. 만약 세마포어가 사용가능하다면(세마포어 값이 0보다 큰 경우) 세마포어 값은 0으로 설정되고, task는 자원을 소유하게 된다. 만약 세마포어의 값이 0이면, 해당 세마포어에 대해 대기하고, 해당 task는 대기 목록에 놓이게 된다. μC/OS-III는 타임아웃이 지정되도록 허용한다. 만약 일정 시간 내에 세마포어를 사용할 수 없다면, 세마포어를 요청한 task는 ready-to-run 상태가 되어 호출자에게 에러 코드(타임아웃이 발생했음을 알림)를 반환한다.

task는 Signal(또는 Post)연산을 수행하여 세마포어를 해제한다. 만약 세마포어를 기다리는 task가 없다면 , 세마포어 값은 단순히 1로 설정된다. 세마포어를 기다리는 task가 적어도 하나 있다면, 세마포어에서 대기 중인 가장 높은 우선순위의 task는 ready-to-run 상태가 되고, 세마포어의 값은 증가되지 않는다. 만약 실행준비된 task가 현재 task(세마포어를 방출한 task)보다 높은 우선순위를 가지면, 문맥 교환이 발생하고, 더 높은 우선순위의 task가 실행된다. 현재 task는 가장 높은 우선순위의 ready-to-run task가 될 때까지 멈춘다.

위에서 서술한 동작은 L13-5에 나타난 의사 코드를 이용하여 요약된다.

```c
OS_SEM MySem; (1)
void main (void)
{
    OS_ERR err;
    :
    :
    OSInit(&err);
    :
    OSSemCreate(&MySem, (2)
                “My Semaphore”, (3)
                1, (4)
                &err); (5)
    /* Check “err” */
    :
    /* Create task(s) */
    :
    OSStart(&err);
    (void)err;
}
```
### L13-5(1)
애플리케이션은 OS_SEM 타입의 변수로 세마포어를 선언해야한다. 이 변수는 다른 세마포어 서비스들이 참조한다.

### L13-5(2)
OSSemCreate()를 호출하여 세마포어를 생성하고 주소를 (1)에서 할당된 세마포어에 전달한다. 세마포어는 다른 task에서 사용하기 전에 먼저 생성되어야 한다. 여기서 세마포어는 시작 코드(즉, main())에서 초기화되지만, 다른 task에서 초기화될 수도 있다(그러나 사용하기 전에 반드기 초기화해야한다).

### L13-5(3)
세마포어에 ASCII 이름을 할당할 수 있는데, 이는 디버거나 μC/Probe가 세마포어를 쉽게 식별할 수 있도록 하는데 사용할 수 있다. ASCII 문자는 RAM보다 ROM에 저장되어있다. 런타임에 세마포어의 이름을 변경할 필요가 있다면 RAM에 있는 배열 형태로 문자를 저장하고, OSSemCreate()로 전달할 수 있다. 물론 배열은 NUL문자로 끝나야한다.

### L13-5(4)
세마포어의 초기값을 지정한다. 세마포어가 하나의 공유 자원에 접근하기 위해 사용될 때는 세마포어를 1로 초기화해야한다.

### L13-5(5)
OSSemCreate()는 결과에 따라 에러 코드를 반환한다. 모든 argument가 유요하면, err에는 OS_ERR_NONE이 포함된다. 다른 에러 코드와 그 의미에 대해서는 OSSemCreate() 설명(443페이지 Appendix A "μC/OS-III API Reference")을 참조한다.

## Binary Semaphores (2)

```c
void Task1 (void *p_arg)
{
    OS_ERR err;
    CPU_TS ts;

    while (DEF_ON) {
        :
        OSSemPend(&MySem, (1)
                    0, (2)
                    OS_OPT_PEND_BLOCKING, (3)
                    &ts, (4)
                    &err); (5)
        switch (err) {
            case OS_ERR_NONE:
                Access Shared Resource; (6)
                OSSemPost(&MySem, (7)
                            OS_OPT_POST_1, (8)
                            &err); (9)
                /* Check “err” */
                break;
            case OS_ERR_PEND_ABORT:
                /* The pend was aborted by another task */
                break;
            case OS_ERR_OBJ_DEL:
                /* The semaphore was deleted */
                break;
            default:
                /* Other errors */
        }
        :
    }
}
//L13-6 Using a semaphore to access a shared resource
```

### L13-6(1)
task는 OSSemPend()를 호출하여 해당 세마포어를 대기한다. 애플리케이션은 대기할 세마포어를 지정해야하며, 세마포어는 이전에 생성되어있어야한다.

### L13-6(2)
다음 argument는 클럭 틱 수로 지정된 타임아웃이다. 실제 타임아웃은 tick rate에 따라 달라진다. tick rate(os_cfg_app.h)를 1000으로 설정하면, 10틱의 타임아웃은 10밀리초를 나타낸다. 타임아웃을 0으로 지정하는 것은 세마포어를 영원히 기다리는 것을 의미한다.

### L13-6(3)
세 번째 argument는 기다리는 방법을 지정한다. OS_OPT_PEND_BLOCKING과 OS_OPT_PEND_NON_BLOCKING의 두 가지 옵션이 있다. 블로킹 옵션은 세마포어를 사용할 수 없는 경우 OSSemPend()를 호출한 task는 세마포어가 방출될 때까지 또는 타임아웃이 만료될 때까지 대기한다는 것을 의미한다. 논블로킹 옵션은 세마포어를 사용할 수 없는 경우 OSSemPend()가 즉시 리턴되며, 기다리지 않는다는 것을 나타낸다. 후자는 공유 리소스를 사용하기 위해 세마포어를 사용할 때 거의 사용되지 않는다.

### L13-6(4)
세마포어가 방출되면, μC/OS-III는 타임스탬프를 읽고 OSSemPend()가 반환될 때 이 타임스탬프를 반환한다. 이 기능을 애플리케이션은 post가 언제 일어났고 세마포어가 언제 방출되었는지 알 수 있다. 이때 OS_TS_GET()을 읽어서 현재 타임스탬프를 가져오고, 이를 이용해 대기시간을 계산할 수 있다.

### L13-6(5)
OSSemPend()는 결과에 따라 오류 코드를 반환한다. 호출이 성공하면 err는 OS_ERR_NONE이다. 성공하지 못하면 오류 코드는 오류의 이유를 나타낼 것이다. OSSemPend()의 가능한 오류 코드 목록은 443페이지 Appendix A "μC/OS-III API Reference"를 참조한다. 다른 task가 세마포어를 삭제하거나 대기를 중단시킬 수 있으므로, 오류 코드를 확인하는 것이 중요하다. 그러나 실행 시간에 커널 객체를 삭제하는 것은 권장되지 않는데, 심각한 문제를 일으킬 수 있기 때문이다.

### L13-6(6)
OSSemPend()가 리턴될 때 오류가 없으면 공유자원에 접근할 수 있다.

### L13-6(7)
공유자원에 접근이 끝나면 OSSemPost()를 호출하여 방출할 세마포어를 지정한다.

### L13-6(8)
OS_OPT_POST_1은 세마포어에 많은 task가 대기하고 있다면 세마포어가 하나의 task만 시그널링한다는 것을 나타낸다. 사실, 세마포어가 공유 자원에 접근하기 위해 사용될 때, 항상 이 옵션을 지정해야한다.

### L13-6(9)
대부분의 μC/OS-III 함수와 마찬가지로 호출 시 오류 메시지를 수신할 변수의 주소를 지정한다.

## Binary Semaphores (3)

```c
void Task2 (void *p_arg)
{
    OS_ERR err;
    CPU_TS ts;
    while (DEF_ON) {
        :
        OSSemPend(&MySem, (1)
                    0,
                    OS_OPT_PEND_BLOCKING,
                    &ts,
                    &err);
        switch (err) {
            case OS_ERR_NONE:
                Access Shared Resource;
                OSSemPost(&MySem,
                            OS_OPT_POST_1,
                            &err);
                /* Check “err” */
                break;
            case OS_ERR_PEND_ABORT:
                /* The pend was aborted by another task */
                break;
            case OS_ERR_OBJ_DEL:
                /* The semaphore was deleted */
                break;
            default:
                /* Other errors */
        }
        :
    }
}

```
### L13-7(1)
공유 자원에 접근하려는 다른 task는 동일한 절차를 사용하여 공유자원에 접근해야한다.

## Binary Semaphores (4)
세마포어는 task가 입출력 장치를 공유할 때 특히 유용하다. 두 작업이 동시에 프린터로 문자를 보내는 것을 허용한다면 어떤 일이 일어날지 상상해보라. 프린터에 각 task에서 인터리빙된 데이터가 들어있을 것이다. 예를 들어 task1에서 출력한 "I am Task 1"과 task2에서 출력한 "I am Task 2"는 “I Ia amm T Tasask k1 2”로 출력될 수 있다. 이 경우 1로 초기화한 세마포어를 사용할 수 있다. 규칙은 간단하다: 각 task는 프린터에 접근하려면 먼저 해당 리소스의 세마포어를 얻어야 한다. Fig 13-1은 프린터에 대한 독점적 접근을 얻기 위해 세마포어를 경쟁하는 task를 보여준다. 프린터를 사용하기 위해 각 task가 키를 얻어야함을 보여주고, 키는 세마포어를 상징적으로 나타낸다는 것에 주목하라.

![Fig 13-1 Using a semaphore to access a printer](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/ff1301f4-9147-43a5-bc59-83d8b077ee6f)

위의 예는 각 task가 공유 자원에 접근할려면 세마포어가 필요하다는 것을 알고 있음을 의미한다. 임계 구역과 그 보호 매커니즘을 캡슐화하는 것이 좋다. 만약 캡슐화한다면, 각 task는 자원에 접근할 때 세마포어를 획득하고 있다는 것을 알지 못할 것이다. 예를 들어 RS-232C 포트는 Fig 13-2와 같이 명령을 보내고 다른 끝에 연결된 장치로부터 응답을 받기 위해 사용한다.

![Fig 13-2 Hiding a semaphore from a task](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/c7e1e2ba-05c5-4934-b129-55c5b4a9a1ac)

CommSendCmd() 함수는 명령어를 포함하는 ASCII 문자열, 장치의 응답 문자열에 대한 포인터, 마지막으로 장치가 일정 시간 내에 응답하지 않을 경우의 타임아웃 등 세 가지 argument로 호출된다. 이 함수에 대한 의사 코드는 L13-8에 나타나 있다.

```c
APP_ERR CommSendCmd (CPU_CHAR *cmd,
                    CPU_CHAR *response,
                    OS_TICK timeout)
{
    Acquire serial port’s semaphore;
    Send “cmd” to device;
    Wait for response with “timeout”;
    if (timed out) {
        Release serial port’s semaphore;
        return (error code);
    } else {
        Release serial port’s semaphore;
        return (no error);
    }
}
//L13-8 Encapsulating the use of a semaphore
```
장치에 명령어를 전송해야하는 각 task는 이 함수를 호출해야한다. 세마포어는 통신 드라이버 초기화 루틴에 의해 1로 초기화된 것으로 가정한다. CommSendCmd()를 호출하는 첫 번째 task는 세마포어를 획득하고, 명령어를 전송하고, 응답을 기다린다. 포트가 busy상태일 때 두 번쨰 task가 명령어를 전송하려고 하면, 이 두 번째 task는 세마포어가 해제될 때까지 일시 중단된다. 두 번째 task는 단순히 함수가 자신의 임무를 완전히 수행할 때까지 리턴하지 않을, 평범한 함수를 호출한 것으로 본다. 첫 번째 task에 의해 세마포어가 방출되면, 두 번째 task는 세마포어를 획득하고 RS-232C 포트를 사용하도록 허용된다.

## Counting Semaphores
카운팅 세마포어는 자원을 둘 이상의 task가 동시에 사용할 수 있을 때 사용된다. 예를 들어, Fig 13-3과 같이 버퍼 풀의 관리에 카운팅 세마포어가 사용된다. 처음에 버퍼 풀은 10개의 버퍼가 있다고 하자. task는 BufReq()를 호출하여 버퍼 매니저로부터 버퍼를 얻는다. 버퍼가 더 이상 필요하지 않을 때 task는 BufRel()을 호출하여 버퍼 매니저에게 반환한다. 이러한 함수들의 의사코드는 L13-9에 있다.

버퍼 매니저는 세마포어가 10으로 초기화되었기 때문에 처음 10개의 버퍼 요청은 받아들인다. 버퍼를 모두 사용하면, 버퍼를 요청한 task는 버퍼가 사용 가능해질 때까지 일시 중단된다.  μC/OS-III의 OSMemGet()과 OSMemPut() (343페이지의 17장 "Memory Management" 참조)을 사용하여 버퍼 풀로부터 버퍼를 얻는다. 획득한 버퍼로 작업을 끝내면, task는 BufRel()을 호출하여 버퍼 매니저에게 버퍼를 반환하고, 세마포어가 방출되기 전에 버퍼를 링크드리스트에 삽입한다. BufReq()와 BufRel()은 캡슐화되어있으므로, 호출자는 실제 구현 내용에 관심을 가질 필요가 없다.

![Fig 13-3 Using a counting semaphore](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/408eb366-24e8-469c-b44f-c168530b00f3)

```c
BUF *BufReq (void)
{
    BUF *ptr;

    Wait on semaphore;
    ptr = OSMemGet(...) ; /* Get a buffer */
    return (ptr);
}
void BufRel (BUF *ptr)
{
    OSMemPut(..., (void *)ptr, ...); /* Return the buffer */
    Signal semaphore;
}
```
이것은 343페이지의 17장 "Memory Management"에서 논의될 예정이므로, 메모리 파티션을 생성하는 상세사항은 생략하였다는 것에 유의한다. 세마포어는 여기서 μC/OS-III의 메모리 관리 능력을 확장하고, 차단 방법을 제공하기 위해 사용된다. 단, task만이 BufReq()와 BufRel()을 호출할 수 있다.

## Notes On Semaphores
공유 자원에 접근하기 위해 세마포어를 사용하는 것은 interrupt latency를 증가시키지 않는다. ISR 또는 현재 task가 공유 자원에 접근하는 동안 더 높은 우선순위의 task가 ready-to-run이 되면, 더 높은 우선순위의 task는 즉시 실행된다.

애플리케이션은 다양한 리소스를 보호하기 위해 필요한 만큼의 세마포어를 가질 수 있다. 예를 들어, 하나의 세마포어는 공유 디스플레이에 접근하기 위해, 다른 하나는 공유 프린터에 접근하기 위해, 다른 하나는 공유 자료구조를 위해, 또 다른 하나는 버퍼 풀을 보호하기 위해 사용될 수 있다. 다만 메모리보다 I/O 디바이스들에 대한 접근을 보호하기 위해 세마포어를 사용하는 것이 바람직하다.

세마포어는 종종 남용된다. 단순히 공유 변수에 접근하기 위한 세마포어의 사용은 대부분의 상황에서 지나치다. 세마포어를 획득하고 방출하는데 관련된 오버헤드는 귀중한 CPU 시간을 소모한다. 인터럽트를 비활성화하고 활성화함으로써 그 작업을 더 효율적으로 수행할 수 있지만, 인터럽트를 비활성화할 때 드는 간접적인 비용이 있다. 예를 들어 두 task가 32비트 정수 변수를 공유한다고 가정하자. 첫 번째 task는 변수를 증가시키고, 두 번째 task는 변수를 초기화한다. 프로세서가 어느 하나의 task를 수행하는 데 걸리는 시간을 고려할 때 해당 변수에 대한 독점적인 접근 권한을 얻기 위해 세마포어가 필요하지 않다는 것을 쉽게 알 수 있다. 각 task는 단순히 해당 변수에 대한 작업을 수행하기 전에 인터럽트를 비활성화하고 작업이 완료되면 인터럽트를 활성화하면 된다. 변수가 부동 소수점 변수이고 마이크로프로세서가 하드웨어 부동 소수점 작업을 지원하지 않는 경우 세마포어를 사용해야한다. 이 경우에, 인터럽트가 비활성화되는 시간이 길어져(부동 소수점 변수 처리 시간 때문에) interrupt latency에 영향을 줄 수 있다.

세마포어는 실시간 시스템에서 우선순위 반전(priority inversion)이라 불리는 심각한 문제를 일으킬 수 있다. 이는 254페이지 [13-3-5 "Priority Inversions](#prioirty-inversions)에 설명되어 있다.

## Semaphores Internals(For Resource Sharing)
앞서 언급한 바와 같이, 세마포어는 OS_SEM 타입으로 정의되는 커널 객체이며, 이는 L13-10에 나와 있는 os_sem(os.h 참조) 구조체로 부터 나온다.

세마포어 관리를 위해 μC/OS-III에서 제공하는 서비스는 os_sem.c 파일에 구현되어있다. 세마포어 서비스는 os_cfg.h에서 설정 상수 OS_CFG_SEM_EN을 1로 설정하여 컴파일 시 활성화된다.

```c
typedef struct os_sem OS_SEM;   (1)

struct os_sem {
    OS_OBJ_TYPE     Type;       (2)
    CPU_CHAR        *NamePtr;   (3)
    OS_PEND_LIST    PendList;   (4)
    OS_SEM_CTR      Ctr;        (5)
    CPU_TS          TS;         (6)
};
//L13-10 OS_SEM data type
```
### L13-10(1)
μC/OS-III에서 모든 구조체를 표현하는 자료형이 주어진다. 모든 자료형은 "OS_"로 시작하며 대문자이다. 세마포어를 선언할 때, 단순히 OS_SEM을 세마포어를 선언할 떄 사용되는 변수의 자료형으로 사용한다.

### L13-10(2)
구조체는 "Type" 필드로 시작하는데, 이 필드를 통해 μC/OS-III가 세마포어로 인식할 수 있다. 다른 커널 객체들도 구조체의 첫 번째 멤버는 ".Type"이다. 함수가 커널 객체를 전달받았다면, μC/OS-III은 적절한 자료형(os_cfg.h에서 OS_CFG_OBJ_TYPE_CHK_EN이 1로 설정되었다고 가정할 때)이 전달되었는지 확인한다. 예를 들어 메시지 큐(OS_Q)를 세마포어 서비스(예를 들어 OSSemPend())로 전달하면 μC/OS-III은 잘못된 객체가 전달되었음을 인식하고 그에 따라 오류 코드를 반환할 것이다.

### L13-10(3)
각 커널 객체는 디버거나 μC/Probe가 더 쉽게 인식할 수 있도록 이름을 부여할 수 있다. 이 멤버는 단순히 NUL문자로 끝나는 ASCII 문자열에 대한 포인터이다.

### L13-10(4)
세마포어에서 여러 task가 대기할 수 있기 때문에, 세마포어 객체는 197페이지의 10장 "Pend Lists(or Wait Lists)" 에 설명된 바와 같은 대기 목록을 포함한다.

### L13-10(5)
세마포어는 카운터를 포함한다. 앞서 설명한 바와 같이. 카운터는 os_type.h에 있는 자료형 OS_SEM_CTR은 어떻게 선언하는가에 따라 8비트, 16비트, 32비트 값으로 구현될 수 있다.

μC/OS-III는 이진 세마포어와 카운팅 세마포어를 구분하지 않는다. 세마포어를 만들 때 차이가 생긴다. 초기 값이 1인 세마포어는 이진 세마포어이다. 값이 1보다 큰 세마포어는 카운팅 세마포어이다. 다음 장에서 세마포어가 시그널링 메커니즘으로 더 자주 사용되므로 세마포어 카운터가 0으로 초기화된다는 것을 발견할 것이다.

### L13-10(6)
세마포어는 세마포어가 마지막으로 방출된 시간을 나타내기 위해 사용되는 타임스탬프를 포함한다. μC/OS-III는 애플리케이션이 시간 측정을 할 수 있게 해주는 free-running 카운터가 있다고 가정한다. 세마포어가 방출될 때 free-running 카운터를 읽고, 그 값은 OSSemPend()의 반환값중 하나인 이 필드에 대입된다. 이 필드의 값은 자원 공유 메커니즘이 아니라 시그널링 메커니즘으로 사용될 때(273페이지의 14장 "Synchronization" 참조)더 유용하다.

## Semaphores Internals(For Resource Sharing) (2)
OS_SEM 자료형 내부를 이해하더라도 애플리케이션 코드는 이 자료구조의 어떤 필드에도 직접 겁근해서는 절대 안된다. 대신 μC/OS-III와 함께 제공된 API를 항상 사용해야 한다.

앞서 언급한 바와 같이, 세마포어는 애플리케이션이 사용하기 전에 생성되어야 한다.

task는 L13-11과 같이 OSSemPend()를 호출하여 공유 자원에 접근하기 전에 세마포어 상에서 대기한다(argument에 대한 자세한 내용은 443페이지의 Appendix A, “μC/OS-III API Reference” 참조)

```c
OS_SEM MySem;

void MyTask (void *p_arg)
{
    OS_ERR err;
    CPU_TS ts;

    :
    while (DEF_ON) {
        :
        OSSemPend(&MySem, /* (1) Pointer to semaphore */
                    10, /* Wait up until this time for the semaphore */
                    OS_OPT_PEND_BLOCKING, /* Option(s) */
                    &ts, /* Returned timestamp of when sem. was released */
                    &err); /* Pointer to Error returned */
        :
        /* Check “err” */ /* (2) */
        :
        OSSemPost(&MySem, /* (3) Pointer to semaphore */
                    OS_OPT_POST_1, /* Option(s) ... always OS_OPT_POST_1 */
                    &err); /* Pointer to Error returned */
        /* Check “err” */
        :
        :
    }
}
//L13-11 Pending on and Posting to a Semaphore
```
### L13-11(1)
OSSemPend()가 호출되면, 이 함수에 전달된 argument가 유요한 값을 가지는지 확인하는 것으로 시작한다(os_cfg.h에서 OS_CFG_ARG_CHK_EN이 1로 설정되었다고 가정).

세마포어 커운터(OS_SEM의 .Ctr)가 0보다 크면 카운터가 감소하고 OSSemPend()가 리턴된다. OSSemPend()가 오류 없이 반환되면 task는 이제 공유자원을 소유한다.

세마포어 카운터가 0이면 다른 task가 세마포어를 소유한 것이므로, OSSemPend()를 호출한 task는 세마포어가 방출될 때까지 기다려야한다. 옵션으로 OS_OPT_PEND_NON_BLOCKING을 지정하면 (애플리케이션이 task가 자원을 기다린다고 일시정지되는 것을 원하지 않는 것이다) OSSemPend()가 즉시 리턴되고, 반환된 오류 코드는 세마포어를 사용할 수 없음을 나타낸다. task가 자원을 사용할 수 있을 때까지 기다리지 않고 다른 작업을 수행한 후 나중에 다시 확인하는 것을 선호하는 경우 이 옵션을 사용한다.

OS_OPT_END_BLOCKING 옵션을 지정하면, OSSemPend()를 호출한 task는 세마포어를 대기하는 목록에 들어간다. task는 우선순위에 따라 목록에 들어가므로, 세마포어 상에서 대기하는 가장 높은 우선순위의 task는 목록의 처음에 있다.

타임아웃을 0이 아니게 지정하면, tick list에도 들어간다. 타임아웃을 0으로 지정하면 task는 세마포어가 방출될 때까지 영원히 기다릴 것이다. 대부분의 경우 자원 공유에서 세마포어를 사용할 때 타임아웃을 0으로 지정할 것이다. 타임아웃을 추가하면 일시적으로 교착 상태가 깨질 수 있지만 애플리케이션 레벨에서 교착 상태를 방지하는 더 나은 방법(예를 들어, 동시에 두 개 이상의 세마포어를 보유하지 않는 것, 자원 순서 지정 등)이 있다.

task가 blocking(세마포어를 기다리기 위해 task가 일시정지됨)되면, 현재 task가 더 이상 실행될 수 없으므로, 스케줄러가 호출된다. 그러면 스케줄러는 다음으로 우선순위가 높은 ready-to-run 상태의 task를 실행한다.

세마포어가 방출되고 OSSemPend()을 호출한 task가 다시 가장 높은 우선순위의 task가 되면, μC/OS-III는 task 상태를 조사하여 OSSemPend()가 리턴되는 이유를 파악한다. 가능성은 다음과 같다:

1) 세마포어가 주어졌다.
2) 다른 task에 의해 대기가 취소되었다.
3) 세마포어가 특정 타임아웃 내로 방출되지 않았다.
4) 세마포어가 삭제되었다.

OSSemPend()가 리턴되면, 호출자는 에러코드를 통해 위 결과를 알게 된다.

### L13-11(2)
OSSemPend()가 err를 OS_ERR_NONE으로 설정 후 리턴하는 경우, 코드는 이제 자원에 접근할 수 있다고 생각할 수 있다.

err에 다른 내용이 포함된 경우, 즉 OSSemPend()가 타임아웃되었거나(타임아웃 argument가 0이 아닌 경우), 대기가 다른 task에 의해 중단되었거나, 세마포어가 다른 task에 의해 삭제된 경우가 있다. 따라서 반환된 오류 코드를 검사하고 모든 것이 잘 되었다고 생각하지 않는 것이 항상 중요하다.

### L13-11(3)

## Prioirty Inversions