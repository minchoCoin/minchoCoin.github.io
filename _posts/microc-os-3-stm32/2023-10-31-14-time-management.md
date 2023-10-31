---
title: "μC/OS-III ch.11 Time Managment"
last_modified_at: 2023-10-31T21:20:12+09:00
categories:
    - microc-os-3-stm32
tags:
    - microc-os
    - embedded-system

toc: true
toc_label: "My Table of Contents"
author_profile: true

---
μC/OS-III는 애플리케이션 프로그래머에게 시간 관련 서비스를 제공한다.

175페이지의 9장 "Interrupt Management"에서, μC/OS-III는 일반적으로 (대부분의 커널들과 마찬가지로) 사용자가 time delay나 타임아웃을 사용하기 위해 주기적 인터럽트를 제공할 것을 요구한다. 이 주기적 시간 소스는 clock tick이라고 불리고 초당 10회 내지 1000회(또는 Hz) 사이에서 발생해야한다(os_cfg_app.h의 OS_CFG_TICK_RATE_HZ 참조). clock tick의 실제 주파수는 애플리케이션이 원하는 tick rate에 달려있다. 그러나, tick의 주파수가 높을 수록, 오버헤드가 크다.

μC/OS-III는 표 11-1에 정리된 바와 같이 시간을 관리하기 위한 여러 가지 서비스를 제공하며, 코드는 os_time.c에 있다.

| Function Name | Operation |
|---------------|-----------|
|OSTimeDly()               |task를 "n" tick 지연시킨다.           |
|OSTimeDlyHMSM()               |task를 HH:MM:SS.mmm 만큼 지연시킨다.           |
|OSTimeDlyResume()               |일정 시간 대기 중인 task를 재개한다(즉, 대기를 취소한다)           |
|OSTimeGet()               |현재 tick counter값을 얻는다.           |
|OSTimeSet()               |tick counter를 새로운 값으로 설정한다           |
|OSTimeTick()               |clock tick 발생을 알린다.           |

애플리케이션 프로그래머는 이러한 서비스에 대한 상세한 설명을 보기 위해 Appendix A, "μC/OS-III API Reference"를 참조해야한다.

# OSTimeDly()
task는 일정 시간 동안 실행을 중지하기 위해 이 함수를 호출한다. 이 함수를 호출한 task는 지정된 시간이 만료될 때 까지 실행되지 않을 것이다. 이 함수는 상대적(relative), 주기적(periodic), 절대적(absolute)의 세 가지 모드가 있다.

L11-1은 상대적 모드의 OSTimeDly()를 쓰는 방법을 보여준다.

```c
void MyTask (void *p_arg)
{
    OS_ERR err;
    :
    :
    while (DEF_ON) {
        :
        :
        OSTimeDly(2, (1)
                    OS_OPT_TIME_DLY, (2)
                    &err); (3)
        /* Check “err” */ (4)
        :
        :
    }
}
```

## L11-1(1)
첫 번째 argument는 함수가 호출될 때부터 시작하여 얼마나 시간 지연을 할 것인지를 틱 단위로 지정한다. 예를 들어, L11-1의 경우 tick rate(os_cfg_app.h의 OS_CFG_TICK_RATE_HZ)가 1000Hz로 설정되면, 사용자는 현재 task를 대략 2밀리초 동안 중단하도록 요청하는 것이다. 그러나 카운트가 다음 틱부터 시작되기 때문에 그 값은 정확하지 않다. 이에 대해서는 간략하게 설명하기로 한다.

## L11-1(2)
OS_OPT_TIME_DLY는 상대적 모드로 이 함수를 쓰는 것을 의미한다.

## L11-1(3)
대부분의 μC/OS-III 서비스와 마찬가지로 에러 반환 값이 반환될 것이다. 예제는 argument들이 모두 유효하므로 OS_ERR_NONE을 반환해야 한다.

## L11-1(4)
μC/OS-III에 의해 반환되는 오류 코드를 항상 확인해야 한다. "err"가 OS_ERR_NONE가 아니라면, OSTimeDly()는 의도한 작업을 수행하지 않은 것이다. 예를 들어, 다른 task에서 OSTimeDlyResume()을 호출하여 time delay를 중단하고, MyTask()로 돌아왔을 때, 시간이 만료되었었기 때문에, 반환되지 않을 것이다.

# OSTimeDly() (2)
위에서 말한 대로, delay는 정확하지 않다. Figure 11-1과 설명을 참고하여 왜 그런지 확인해보자.

![Figure 11-1 OSTimeDly() - Relative](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/1aa71530-7df5-43bd-848a-79d655b9132c)

## F11-1(1)
틱 인터럽트가 발생하고, μC/OS-III은 ISR을 제공한다.

## F11-1(2)
ISR이 끝나면, 모든 높은 우선순위 task (HPT)가 실행된다. HPT의 실행 시간은 알 수 없고 다양할 수 있다.

## F11-1(3)
모든 HPT가 실행되면 μC/OS-III는 L11-1과 같이 OSTimeDly()를 호출하는 task를 실행한다. 논의를 위해, 이 task는 낮은 우선순위의 task(LPT)라고 가정한다.

## F11-1(4)
task는 OSTimeDly()를 호출하고 "상대적" 모드에서 두 틱 동안 지연되도록 지정한다. 이때 μC/OS-III는 현재 task를 tick list에 올려 두 틱이 만료되기를 기다린다. 일정 시간 대기하는 task는 시간이 만료되기를 기다리는 동안 CPU 시간을 소모하지 않는다.

## F11-1(5)
다음 틱이 발생한다. 만약 이 특정 틱을 기다리는 HPT가 있다면,  μC/OS-III는 ISR이 끝날 때 실행되도록 스케줄링할 것이다.

## F11-1(6)
HPT가 실행된다.

## F11-1(7)
다음 틱 인터럽트(tick interrupt)가 발생한다. 이는 LPT가 대기하고 있던 틱이며, μC/OS-III에 의해 Ready-to-run상태가 될 것이다.

## F11-1(8)
이 틱에서 실행할 HPT가 없으므로 μC/OS-III는 LPT로 전환된다.

## F11-1(9)
HPT의 실행시간이 있을 때, 시간 지연은 요청한 것처럼 정확히 2개의 틱이 아니다. 실 정확히 원하는만큼의 틱의 지연을 구하는 것은 사실상 불가능하다. 두 틱의 지연을 요청할 수도 있지만, 바로 다음 틱은 OSTimeDly()를 호출한 직후에 발생할 수 있다! 만약 모든 HPT가 실행하는 데 시간이 더 걸리고 (3)과 (4)를 오른쪽으로 더 밀어냈을 때 어떤 일이 일어날지 상상해 보아라. 이 경우 지연은 실제로 두 틱이 아니라 한 틱으로 나타날 것이다.

# OSTimeDly() (3)
OSTimeDly()는 L11-2와 같이, OS_OPT_TIME_PERIODIC 옵션으로 호출될 수도 있다. 이 옵션은 틱 카운터가 어떤 주기적 매치 값에 도달할 때까지 task를 지연시킬 수 있으며, 따라서 시간 내의 간격이 CPU 부하 변동의 영향을 받지 않으므로 항상 동일함을 보장한다.
