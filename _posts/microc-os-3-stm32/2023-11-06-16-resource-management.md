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

# 인터럽트 활성화/비활성화
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

# 인터럽트 활성화/비활성화 (2)
CPU_CRITICAL_ENTER() 와 CPU_CRITICAL_EXIT() 는 항상 쌍으로 사용된다. 인터럽트를 비활성화하면 인터럽트에 대한 시스템의 반응에 영향을 미치므로 인터럽트는 가능한 짧은 시간 동안 비활성화되어야 한다. 이를 interrupt latency라고 한다. 인터럽트 활성화/비활성화 방식은 몇 개의 변수를 변경하거나 복사할 때만 사용된다.

이것이 task가 ISR과 변수 또는 자료 구조를 공유할 수 있는 유일한 방법임에 유의한다.

μC/CPU는 interrupt latency를 실제로 측정하는 방법을 제공한다.

μC/OS-III을 사용할 때 interrupt latency에 영향을 주지 않으면서 μC/OS-III가 비활성화하는 시간 만큼 인터럽트가 비활성화될 수도 있다. 분명한 것은 μC/OS-III가 인터럽트를 비활성화하는 기간을 아는 것이 중요한데, 이는 사용되는 CPU에 의존한다.

이 방법은 작동하지만 실시간 이벤트에 대한 시스템의 응답성에 영향을 미치므로 인터럽트를 비활성화하지 않아야 한다.