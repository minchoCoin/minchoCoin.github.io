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

(표11-1 Time Services API summary)

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
HPT의 실행시간이 있을 때, 시간 지연은 요청한 것처럼 정확히 2개의 틱이 아니다. 사실 정확히 원하는만큼의 틱의 지연을 구하는 것은 사실상 불가능하다. 두 틱의 지연을 요청할 수도 있지만, 바로 다음 틱은 OSTimeDly()를 호출한 직후에 발생할 수 있다! 만약 모든 HPT가 실행하는 데 시간이 더 걸리고 (3)과 (4)를 오른쪽으로 더 밀어냈을 때 어떤 일이 일어날지 상상해 보아라. 이 경우 지연은 실제로 두 틱이 아니라 한 틱으로 나타날 것이다.

# OSTimeDly() (3)
OSTimeDly()는 L11-2와 같이, OS_OPT_TIME_PERIODIC 옵션으로 호출될 수도 있다. 이 옵션은 틱 카운터가 어떤 주기적 매치 값에 도달할 때까지 task를 지연시킬 수 있으며, 따라서 시간 내의 간격이 CPU 부하 변동의 영향을 받지 않으므로 항상 동일함을 보장한다.

μC/OS-III는 OSTickCtr의 "match value"를 계산하여, 원하는 주기를 기반으로 task가 언제 깨어나야하는지를 확인한다. 이는 Figure 11-2에 나타나있다. μC/OS-III는 match를 계산하어 match가 그때 이미 지나간 값을 나타내는 경우 지연이 0이 되도록 한다.

![Figure 11-2 OSTimeDly() - Periodic](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/d1641b02-0108-4a5f-9cc0-3cd9ffe421da)

```c
void MyTask (void *p_arg)
{
    OS_ERR err;
    :
    :
    while (DEF_ON) {
        OSTimeDly(4, (1)
                    OS_OPT_TIME_PERIODIC, (2)
                    &err);
        /* Check “err” */ (3)
        :
        :
    }
}
//L11-2 OSTimeDly() - Periodic
```
## L11-2(1)
첫번째 argument는 task가 실행될 주기를 지정하며, 여기서는 4틱을 지정한다. 물론, task의 우선순위가 낮은 경우, μC/OS-III는 우선순위에 기반하여 task를 스케줄링하고 실행할 뿐이다.

## L11-2(2)
OS_OPT_TIME_PERIODIC으로 지정하면, 틱 카운터가 이전 호출에서부터 시작하여 원하는 주기에 도달할 때 task가 ready-to-run 상태가 된다.

## L11-2(3)
μC/OS-III에서 반환된 오류 코드를 항상 확인해야 한다.

# OSTimeDly() (4)
상대적(relative) 모드와 주기적(periodic)모드가 똑같아보일 수 있지만, 다르다. 상대적 모드에서, 시스템이 바빠서 틱을 놓칠 수 있다. 주기적 모드에서, task는 여전히 일정시간 대기하지만, 항상 원하는 틱 수에 동기화될 것이다. 사실, 주기적 모드는 time-of-day 클록을 구현하는데 사용된다.

마지막으로 절대적(absolute) 모드를 사용하여 전원을 켠 후 일정 시간 후에 특정 동작을 수행할 수 있다. 예를 들어, 제품 전원을 켠 후, 10초 후에 불을 끌 수 있다. 이 경우 실제로 도달하고자하는 OSTickCtr 값은 dly 값이다.

요약하면, task는 OSTickCtr이 아래와 같은 값에 도달했을 때 깨어난다.

```c
Value of "opt"          Task wakes up when

OS_OPT_TIME_DLY         OSTickCtr + dly

OS_OPT_TIME_PERIODIC    OSTCBCurPtr->TickCtrPrev + dly

OS_OPT_TIME_MATCH       dly
```

# OSTimeDlyHMSM()
task는 보다 사용자 친화적인 방식으로 시간의 길이를 지정함으로써, 일정 시간동안 실행을 중지하도록 이 함수를 호출할 수 있다. 구체적으로 지연될 시간(Hours), 분(Minutes), 초(Seconds), 밀리초(Milliseconds)를 지정할 수 있다(따라서 HMSM). 이 함수는 상대적(relative)모드에서만 작동한다.

L11-3은 OSTimeDlyHMSM()을 어떻게 쓰는지 보여준다.

```c
void MyTask (void *p_arg)
{
    OS_ERR err;
    :
    :
    while (DEF_ON) {
        :
        :
        OSTimeDlyHMSM(0, (1)
                        0,
                        1,
                        0,
                        OS_OPT_TIME_HMSM_STRICT, (2)
                        &err); (3)
        /* Check “err” */
        :
        :
    }
}
//L11-3
```
## L11-3(1)
처음 네 개의 argument는 함수를 호출한 시점부터 시간 지연의 양(시간, 분, 초, 밀리초 단위)을 지정한다. 위의 예에서, task는 1초 동안 지연되어야한다. resolution은 tick rate에 의존한다. 예를 들어 tick rate(os_cfg_app.h에 OS_CFG_TICK_RATE_HZ)가 1000Hz로 설정되면, 기술적으로 1밀리초의 resolution을 가진다. tick rate가 100Hz라면 현재 task의 지연은 10밀리초 단위이다. 즉, 실제 지연은 정확하지 않을 수 있다.

## L11-3(2)
OS_OPT_TIME_HMSM_STRICT를 지정하면 사용자가 시간, 분, 초 및 밀리초의 유효한 값을 엄격하게 전달하는지 확인한다. 유효 시간은 0~99, 유효 분은 0~59, 유효 초는 0~59, 및 유효 밀리초는 0~999이다.

OS_OPT_TIME_HMSM_NON_STRICT를 지정하면, 함수는 시(0~999), 분(0~9999), 초(최대 65,535) 및 밀리초(최대 4,294,967,295)의 값은 거의 모든 값이 허용된다. OSTimeDlyHMSM(203, 101, 69, 10000)은 허용될 수 있다. 이것이 말이 되는지 아닌지는 다른 이야기이다.

시간이 999로 제한되는 이유는 시간 지연이 일반적으로 32비트 값을 사용하여 틱을 추적하기 때문이다. 틱 속도가 1000 Hz로 설정되면 1,193시간에 해당하는 4,294,967초만 추적할 수 있으므로 999는 합리적인 한계이다.

## L11-3(3)
대부분의 μC/OS-III 서비스와 마찬가지로 사용자는 오류 반환 값을 받게 될 것이다. 예제는 L11-3의 인수들이 모두 유효하므로 OS_ERR_NONE을 반환해야 한다. 가능한 오류 코드들의 목록에 대해서는 443페이지의 Appendix A "μC/OS-III API Reference"를 참조한다.

# OSTimeDlyHMSM() (2)
μC/OS-III가 task에 매우 긴 지연을 허용하더라도 실제로 task를 장시간 지연시키는 것은 권장되지 않는다. 그 이유는  남아있는 지연 시간의 양을 모니터링할 수 없다면 task가 실제로 "살아있다"고 인식할 수 없기 때문이다. task가 대략 1분 마다 깨어나고, 여전히 괜찮다고 말하도록 하는 것이 더 좋다.

OSTimeDly()는 주기적인 task(주기적으로 실행되는 task)을 생성하는 데 종종 사용된다. 예를 들어, 50밀리초마다 키보드를 스캔하는 task와 10밀리초마다 아날로그 입력을 읽는 다른 task 등을 가질 수 있다.

# OSTimeDlyResume()
task는 OSTimeDlyResume()을 호출하여, OSTimeDly() 또는 OSTimeDlyHMSM()을 호출한 다른 task를 재개할 수 있다. L11-4는 OSTimeDlyResume()의 사용 방법을 보여준다. 일정 시간을 기다리고 있었던 task는 OSTimeDlyResume()에 의해 재개되었다고 알 수 없고, 지연 시간이 끝났다고 생각할 것이다. 이 때문에 이 함수를 신중하게 사용해야한다.

```c
OS_TCB MyTaskTCB;
void MyTask (void *p_arg)
    {
    OS_ERR err;
    :
    :
    while (DEF_ON) {
        :
        :
        OSTimeDly(10,
                    OS_OPT_TIME_DLY,
                    &err);
        /* Check “err” */
        :
        :
    }
    }

void MyOtherTask (void *p_arg)
{
    OS_ERR err;
    :
    :
    while (DEF_ON) {
        :
        :
        OSTimeDlyResume(&MyTaskTCB,
                        &err);
        /* Check “err” */
        :
        :
    }
}
//L11-4 OSTimeDlyResume()
```

# OSTimeSet() And OSTimeGet()
μC/OS-III는 틱 인터럽트가 발생할 때마다 틱 카운터를 증가시킨다. 이 카운터는 애플리케이션이 대략적인 시간 측정을 하고, 시간 개념을 갖게 해준다.

OSTimeGet()은 사용자가 틱 카운터의 값을 확인할 수 있게 한다. 이 값을 사용하여 특정 개수의 틱 동안 task를 지연시키는 것을 반복할 수 있다(실제 tick list에 들어가지 않고).

OSTimeSet()은 사용자가 틱 카운터의 현재 값을 변경할 수 있게 해준다. μC/OS-III가 이를 허용하지만, 이 기능은 매우 주의하여 사용하는 것이 좋다.

# OSTimeTick()
틱 인터럽트 서비스 루틴(ISR)은 틱 인터럽트가 발생할 때마다 이 함수를 호출해야 한다. μC/OS-III은 이 함수를 사용하여 다른 시스템콜에서 사용되는 time delay와 타임아웃을 업데이트한다. OSTimeTick()은 μC/OS-III의 내부 함수로 간주된다.

# Summary
μC/OS-III는 task가 사용자가 정의한 시간 만큼 실행을 중단할 수 있도록 애플리케이션들에 서비스들을 제공한다. 지연들은 클록 틱들의 수 또는 시간, 분, 초, 및 밀리초 단위로 지정한다.

애플리케이션 코드는 OSTimeDlyResume()을 호출하여 일정 시간 대기 중인 task를 재개할 수 있다. 그러나 재개된 작업은 시간 지연이 만료된 것이 아니라 재개되었음을 알 수 없기 때문에 사용을 권장하지 않는다.

μC/OS-III는 켜지고 난 이후 또는 마지막으로 OSTimeSet()에 의해 틱 카운터가 변경된 이후 발생한 틱 수를 추적한다. 카운터는 OSTimeGet()을 이용하여 어플리케이션 코드가 읽을 수 있다.

# Reference
 - uC/OS-III: The Real-Time Kernel For the STM32 ARM Cortex-M3, Jean J. Labrosse, Micrium, 2009

[책 링크](https://micrium.atlassian.net/wiki/spaces/osiiidoc/overview)
