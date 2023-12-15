---
title: "μC/OS-III ch.12 Timer Managment"
last_modified_at: 2023-11-01T19:18:12+09:00
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

μC/OS-III는 애플리케이션 프로그래머에게 타이머 서비스를 제공하고, os_tmr.c에 타이머를 처리하는 코드가 있다. os_cfg.h에서 OS_CFG_TMR_EN을 1로 설정하면 타이머 서비스가 활성화된다.

타이머는 카운터가 0에 도달했을 때 동작을 수행하는 다운 카운터이다. 사용자는 콜백 함수(또는 단순히, 콜백(callback))을 통해 동작을 제공한다. 콜백은 타이머가 만료되면 호출될, 사용자가 선언한 함수이다. 콜백은 불을 켜고 끄거나, 모터를 구동하는 등 동작을 수행하는데 사용될 수 있다. 그러나 콜백 함수내에서 절대로 blocking 호출을 하지 않는 것이 중요하다(즉, OSTimeDly(), OSTimeDlyHMSM(), OS???Pend(), 또는 타이머 task를 막거나 삭제하는 기타 모든 것)

타이머는 프로토콜 스택(예를 들어, 재전송 타이머)에서 유용하며, 또한 미리 정의된 간격으로 I/O장치를 폴링하는데 사용될 수 있다.

애플리케이션은 임의의 수의 타이머들을 가질 수 있다 (사용 가능한 RAM의 양에 의해서만 제한됨). μC/OS-III에서의 타이머 서비스들 (즉, 함수들)은 OSTmr????() 접두사로 시작하며, 애플리케이션 프로그래머가 사용 가능한 서비스들은 443페이지 Appendix A "μC/OS-III API Reference"에 기술되어 있다.

μC/OS-III에 의해 관리되는 모든 타이머의 해상도는 헤르츠(Hz)로 표현되는 설정 상수 OS_CFG_TMR_TASK_RATE_HZ에 의해 결정된다. 따라서, timer task rate를 10으로 설정하면, 모든 타이머는 1/10초의 해상도(뒤에 다이어그램에서 틱)를 갖게 된다. 

μC/OS-III는 표 12-1에 요약된 바와 같이 타이머를 관리하기 위한 여러 가지 서비스를 제공한다.

| Function Name | Operation |
|---------------|-----------|
|OSTmrCreate()               |작동 모드를 지정하여 타이머를 만든다.           |
|OSTmrDel()               |타이머를 지운다.           |
|OSTmrRemainGet()               |타이머가 만료되기까지 남은 시간을 구한다.           |
|OSTmrStart()               |타이머를 시작 또는 재시작한다.           |
|OSTmrStateGet()               |타이머의 현재 상태를 구한다.           |
|OSTmrStop()               |타이머의 카운트다운 프로세스를 중단한다.           |

(표 12-1 Timer API summary)

타이머를 사용하기 전에 생성해야 한다. OSTmrCreate()를 호출하여 타이머를 만들고 타이머가 어떻게 동작하는지에 따라 이 함수에 많은 argument를 지정한다. 일단 타이머 동작이 지정되면 타이머를 삭제하고 다시 만들지 않으면 작동 모드를 변경할 수 없다. OSTmrCreate()의 함수 프로토타입은 빠른 참조로 아래에 나와 있다:

```c
void OSTmrCreate (OS_TMR *p_tmr, /* Pointer to timer */
                    CPU_CHAR *p_name, /* Name of timer, ASCII */
                    OS_TICK dly, /* Initial delay */
                    OS_TICK period, /* Repeat period */
                    OS_OPT opt, /* Options */
                    OS_TMR_CALLBACK_PTR p_callback, /* Fnct to call at 0 */
                    void *p_callback_arg, /* Arg. to callback */
                    OS_ERR *p_err)
```
일단 생성되면, 타이머는 필요한 만큼 시작(또는 재시작)하고 중지할 수 있다. 타이머는 One-shot,Periodic(초기 지연 없음), Periodic(초기 지연 있음)의 세 가지 모드 중 하나로 동작하도록 생성할 수 있다.

# One-Shot Timers
이름에서 알 수 있듯이 one-shot 타이머는 초기 값에서 카운트다운을 하고 0이 되면 콜백 함수를 호출한 후 중지한다. Figure 12-1은 이러한 동작의 타이밍 다이어그램을 보여주고 있다. 카운트다운은 OSTmrStart()를 호출함으로써 시작된다. 시간 지연이 완료되면, 타이머를 만들었을 때 콜백 함수가 제공되었다는 가정하에, 콜백 함수가 호출된다. 콜백 함수 호출이 완료되면, 타이머는 아무것도 하지 않으며 OSTmrCreate()를 호출하여 재시작해야 프로세스가 다시 시작된다.

OSTmrStop()을 호출하여 타이머의 카운트다운이 0에 도달하기 전에 종료할 수 있다. 이 경우 콜백 함수 호출 여부를 지정할 수 있다.

![Figure 12-1 One Shot Timers (dly is bigger than 0, period == 0)](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/91c0b910-c872-4831-8169-781f622d2a55)

Figure 12-2와 같이 one-shot 타이머는 타이머가 0에 도달하기 전에 OSTmrStart()를 호출하여 다시 트리거할 수 있다. 이 기능은 watchdog 및 유사한 보호 장치를 구현하는데 사용될 수 있다(워치독 타이머는 프로그램 오류 등으로 컴퓨터가 워치독을 재가동하는데 실패하면, 타임아웃 신호를 생성한다).

![Figure 12-2 Retriggering a One Shot Timer](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/347f6c72-5f23-4a4b-bc56-95b1a9ead25a)

# Periodic (No Initial Delay)
Figure 12-3와 같이 주기적 모드로 타이머를 설정할 수 있다. 카운트다운이 만료되면(즉, 0이되면) 콜백 함수를 호출하고, 타이머가 다시 로드되고 프로세스를 반복한다. 타이머를 만들 때 지연을 0(즉, dly==0)으로 지정하고 시작하면 타이머는 즉시 "period"로 다시 로드한다. 카운트다운의 어느 시점에서든, OSTmrStart()를 호출하여 프로세스를 다시 시작할 수 있다.

![Figure 12-3 Periodic Timers (dly == 0, period is bigger than 0)](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/80ea2801-6047-4782-b094-d08f2d7e1132)

# Periodic (With Initial Delay)
Figure 12-4와 같이 타이머는 주기와는 다른 초기 지연을 가지는 주기적 모드로 설정할 수 있다. 첫 번째 카운트다운 값은 OSTmrCreate()호출에서 전달된 dly이고, 다시 로드될 때는 period이다. OSTmrStart()를 호출하여 초기 지연을 포함한 프로세스를 재시작할 수 있다.

![Figure 12-4 Periodic Timers (dly is bigger than 0, period is bigger than 0)](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/d1088e63-6797-4abf-b48c-c67138de8355)

# Timer Management Internals

## Timer Management Internals - Timers States
Figure 12-5는 타이머의 상태 다이어그램을 보여준다.

task는 OSTmrStateGet()을 호출하여 타이머의 상태를 알 수 있다. 또한 카운트다운 과정에서 언제든지 애플리케이션 코드는 OSTmrRemainGet()을 호출하여 타이머가 0에 도달하기까지 남은 시간을 알 수 있다. 반환되는 값은 timer tick 단위로 표시된다. 타이머가 10Hz 속도로 감소하면, 50의 카운트는 5초에 해당한다. 타이머가 중지 상태에 있으면 남은 시간은 초기 지연(one-shot 또는 초기 지연이 있는 주기 모드) 또는 주기(초기 지연 없는 주기 모드)이다.

![Figure 12-5 Timer State Diagram](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/b49461d0-03da-4155-bc76-09e1dd963908)

### F12-5(0)
"Unused" 상태는 생성되지 않았거나 삭제된 타이머이다. 즉, μC/OS-III는 이 타이머에 대해 알지 못한다.

### F12-5(1)
만약 타이머가 만들어지거나 OSTmrStop()을 호출하면, 타이머는 "stopped" 상태이다.

### F12-5(2)
OSTmrStart()를 호출했을 때, 타이머는 "running" 상태이다. 타이머는 멈추거나, 삭제되거나, one-shot을 완료했을 때를 제외하고, 이 상태에 있다.

### F12-5(3)
one-shot 타이머에서 지연이 끝났을 때 타이머는 "Completed" 상태에 있다.

## Timer Management Internals - OS_TMR
타이머는 L12-1과 같이 OS_TMR 데이터 타입(os.h 참조)으로 정의된 커널 객체이다.

타이머 관리를 위해 μC/OS-III에서 제공하는 서비스는 os_tmr.c 파일에 구현되어 있다. 컴파일 시에는 os_cfg.h에서 설정 상수 OS_CFG_TMR_EN를 1로 설정하여 타이머 서비스를 활성화한다.

```c
typedef struct os_tmr OS_TMR; (1)

struct os_tmr {
    OS_OBJ_TYPE         Type; (2)
    CPU_CHAR            *NamePtr; (3)
    OS_TMR_CALLBACK_PTR CallbackPtr; (4)
    void                *CallbackPtrArg; (5)
    OS_TMR              *NextPtr; (6)
    OS_TMR              *PrevPtr;
    OS_TICK             Match; (7)
    OS_TICK             Remain; (8)
    OS_TICK             Dly; (9)
    OS_TICK             Period; (10)
    OS_OPT              Opt; (11)
    OS_STATE            State; (12)
};
//L12-1 OS_TMR data type
```

### L12-1(1)
μC/OS-III에서는 모든 구조체에 자료형이 주어진다. 실제로 모든 자료형은 OS_로 시작하여 모두 대문자이다. 타이머가 선언되면 단순히 OS_TMR을 타이머를 선언하는데 사용되는 변수의 자료형으로 사용한다.

### L12-1(2)
구조체는 "Type" 필드로 시작하는데, 이 필드를 통해 μC/OS-III가 타이머로 인식할 수 있다. 다른 커널 객체들도 구조의 첫 번째 멤버로서 "Type"을 가질 것이다. 함수가 커널 객체를 전달받으면, μC/OS-III는 적절한 데이터 타입을 전달받았음을 확인할 수 있다(os_cfg.h의 OS_CFG_OBJ_TYPE_CHK_EN이  1로 설정되었다고 가정하면). 예를 들어, 만약 메시지 큐(OS_Q)가 타이머 서비스(예를 들어 OSTmrStart())로 전달되면 μC/OS-III는 잘못된 객체가 전달되었다는 것을 인식하고 그에 따라 오류 코드를 반환할 수 있다.

### L12-1(3)
각 커널 객체는 디버거나 μC/Probe에 의해 더 쉽게 인식할 수 있도록 이름을 부여할 수 있다. 이 값은 단순히 null문자로 끝나는 ASCII문자열에 대한 포인터이다.

### L12-1(4)
.CallbackPtr은 타이머가 만료되면 호출되는 함수의 포인터이다. 타이머가 생성되고 이 argument에 NULL 포인터를 주면, 타이머가 만료되었을 때 콜백이 호출되지 않는다.

### L12-1(5)
.CallbackPtr이 NULL이 아닌 경우, 애플리케이션 코드는 타이머가 만료될 때 argument와 함께 콜백이 호출되도록 지정할 수 있다. 즉 이 argument는 콜백 함수에 전달될 argument이다.

### L12-1(6)
.NextPtr과 .PrevPtr은 타이머를 doubly linked list로 구성하는데 사용하는 포인터이다. 이것은 나중에 설명한다.

### L12-1(7)
타이머 매니저 변수(timer manager variable) OSTmrTickCtr이 타이머의 .Match 필드에 지정된 값에 도달하면 타이머가 만료된다. 이에 대해서도 나중에 설명한다.

### L12-1(8)
.Remain 필드는 타이머가 만료되기까지 남은 시간이다. 이 값은 OS_CFG_TMR_WHEEL_SIZE(os_cfg_app.h 참조)마다 한 번씩 업데이트된다. 이 값은 1/OS_CFG_TMR_TASK_RATE_HZ 초의 배수로 표현된다.

### L12-1(9)
.Dly 필드는 타이머가 one-shot 타이머로 설정된 경우 one-shot 타임과 period 타이머로 생성된 경우 초기 지연 값이다. 그 값은 이 값은 1/OS_CFG_TMR_TASK_RATE_HZ 초의 배수로 표현된다(os_cfg_app.h 참조).

### L12-1(10)
.Period 필드는 타이머가 period 모드로 생성된 타이머의 주기이다. 이 값은 1/OS_CFG_TMR_TASK_RATE_HZ 초의 배수로 표현된다(os_cfg_app.h 참조).

### L12-1(11)
.Opt 필드는 OSTmrCreate()로 받은 옵션이다.

### L12-1(12)
.State 필드는 타이머의 현재 상태를 나타낸다(Figure 12-5 참조)

## Timer Management Internals - OS_TMR (2)
OS_TMR 데이터 타입의 내부를 이해한다 하더라도 애플리케이션 코드는 이 데이터 구조의 어떤 필드에도 직접 접근해서는 절대 안 된다. 대신 μC/OS-III와 함께 제공되는 API를 항상 사용해야 한다.

## Timer Management Internals - Timer Task
OS_TmrTask()는 μC/OS-III(os_cfg.h에서 OS_CFG_TMR_EN을 1로 설정)에 의해 생성된 task로 우선 순위는 μC/OS-III의 설정 파일 os_cfg_app.h(OS_CFG_TMR_TASK_PRIO 참조)를 통해 사용자가 설정할 수 있다. OS_TmrTask()는 일반적으로 중간 우선 순위로 설정된다.

OS_TmrTask()는 주기적인 작업이며, 클럭 틱을 생성하는데 사용되는 것과 동일한 인터럽트 소스를 사용한다. 그러나 타이머는 일반적으로 더 느린 속도(즉, 통상적으로 10Hz 정도)로 업데이트되므로, timer tick rate는 소프트웨어적으로 구현된다. 틱 속도가 1000Hz이고 원하는 타이머 속도가 10Hz인 경우, 타이머 task는 Figure 12-6과 같이 100번째 틱 인터럽트마다 신호를 받을 것이다.

![Figure 12-6 Tick ISR and Timer Task relationship](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/b5aeb948-7fe0-43b8-a011-096c7a8cd848)

Figure 12-7은 timer management task와 관련된 타이밍 다이어그램을 보여준다.

![Figure 12-7 Timing Diagram](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/bbb833ca-031f-449f-ad96-3ce766b6d68e)

### F12-7(1)
tick ISR이 실행되고, 인터럽트가 활성화되어있다고 가정한다.

### F12-7(2)
tick ISR은 tick task에게 타이머를 업데이트해야한다고 신호를 보낸다.

### F12-7(3)
tick ISR은 종료되지만, 실행해야할 더 높은 우선순위의 task들이 있을 수 있다(timer task가 낮은 우선순위를 갖는다고 가정함). 따라서 따라서 μC/OS-III는 더 높은 우선순위의 task들을 실행한다.

### F12-7(4)
모든 높은 우선순위의 task가 실행되고, μC/OS-III는 timer task로 전환하고, 3개의 타이머가 만료되었음을 확인한다.

### F12-7(5)
첫번째 타이머의 콜백함수가 실행된다.

### F12-7(6)
두번째로 만료된 타이머의 콜백함수가 실행된다.

### F12-7(7)
세번째로 만료된 타이머의 콜백함수가 실행된다.

## Timer Management Internals - Timer Task (2)
주목해야할 몇 가지 흥미로운 점들이 있다:

- 콜백 함수의 실행은 timer task의 문맥 안에서 수행된다. 이것은 애플리케이션 코드가 timer task에 콜백 함수를 처리할 충분한 스택 공간이 있는지 확인할 필요가 있음을 의미한다.
- 콜백 함수는 timer list에 있는 순서에 따라 차례로 실행된다.
- timer task의 실행 시간은 얼마나 많은 타이머가 만료되는지 및 콜백 함수 각각을 실행하는데 걸리는 시간에 크게 좌우된다. 콜백은 애플리케이션 코드에 의해 제공되기 때문에 이들은 timer task의 실행 시간에 큰 영향을 미친다.
- 타이머 콜백 함수는 이벤트를 기다리지 않아야하는데, 이는 timer task를 영원히는 아니더라도 과도한 시간 동안 지연시킬 수 있기 때문이다.
- 콜백은 스케줄러가 잠긴 상태에서 호출되므로, 콜백이 가능한 빨리 실행되도록 해야한다.

## Timer Management Internals - Timer List
μC/OS-III는 말 그대로 수백 개의 타이머를 유지해야 할 수도 있다(응용 프로그램에서 그 정도의 타이머가 필요한 경우). timer list 관리는 타이머 업데이트에 CPU 시간이 너무 많이 걸리지 않도록 구현되어야 한다. timer list은 Figure 12-8과 같이 tick list와 유사하게 작동한다.

![Figure 12-8 Empty Timer List](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/f7ec9f92-0040-4bbb-8f60-4e3b4ff589f5)

### F12-8(1)
timer list는 테이블(os_cfg_app.c에 선언된 OSCfg_TmrWheel[])과 카운터(os.h에 선언된 OSTmrTickCtr)로 구성된다.

### F12-8(2)
테이블은 컴파일 시간 설정 값인 OS_CFG_TMR_WHEEL_SIZE 개의 항목을 포함한다(os_cfg_app.h 참조) 항목 수는 프로세서가 사용할 수 있는 RAM의 양 과 애플리케이션 내의 최대 타이머 수에 의존한다. OS_CFG_TMR_WHEEL_SIZE의 좋은 시작점은 timer의 개수/ 4일 수 있다. OS_CFG_TMR_WHEEL_SIZE를 timer task rate의 짝수 배로 하는 것은 추천되지 않는다. 예를 들어, timer task가 10Hz이면, OS_CFG_TMR_WHEEL_SIZE를 10 또는 100으로 설정하는 것을 피하는 것이 좋다(대신 11 또는 101 사용). 또한 timer wheel size에 대해 소수를 사용한다. 런타임에 발생할 것을 컴파일 시간에 계획하는 것이 실제로 가능하지는 않지만, 이상적으로 테이블의 각 항목에서 대기 중인 타이머의 수는 균일하게 분포된다.

### F12-8(3)
테이블의 각 항목에는 세 개의 필드가 있다: .NbrEntriesMax, .NbrEntries 및 .FirstPtr. .NbrEntries는 테이블 항목에 연결된 타이머의 수를 나타낸다. .NbrEntriesMax는 해당 항목에 연결되었던 타이머의 수의 최댓값을 나타낸다. .FirstPtr은 해당 목록에 속한 이중으로 연결된 timer list에 대한 포인터를 포함한다(task의 OS_TMR을 통해).

## Timer Management Internals - Timer List(2)
tick ISR이 OS_TmrTask()에 신호를 보낼 때 마다, OSTmrTickCtr이 증가한다.

타이머는 OSTmrStart()를 호출하여 timer list에 들어간다. 타이머가 사용되기 전에 먼저 선언되어야한다.

timer list에 타이머를 넣는 과정을 설명하는 예는 다음과 같다. timer list가 완전히 비어있고, OS_CFG_TMR_WHEEL_SIZE가 9이고, Figure 12-9와 같이 OSTmrTickCtr의 현재 값이 12라고 가정하자. OSTmrStart()를 호출할 때 타이머가 timer list에 들어가는데 타이머가 1만큼 지연하는 것으로 생성되었으며, 이 타이머는 다음과 같이 one-shot 타이머가 된다고 가정하자:

```c
OS_TMR MyTmr1;
OS_TMR MyTmr2;
void MyTmrCallbackFnct1 (void *p_arg)
{
    /* Do something when timer #1 expires */
}
void MyTmrCallbackFnct2 (void *p_arg)
{
    /* Do something when timer #2 expires */
}
void MyTask (void *p_arg)
{
    OS_ERR err;
    while (DEF_ON) {
        :
        OSTmrCreate((OS_TMR *)&MyTmr1,
                    (OS_CHAR *)"My Timer #1",
                    (OS_TICK )1,
                    (OS_TICK )0,
                    (OS_OPT )OS_OPT_TMR_ONE_SHOT,
                    (OS_TMR_CALLBACK_PTR)MyTmrCallbackFnct1,
                    (void *)0,
                    (OS_ERR *)&err);
        /* Check ’err” */
        OSTmrStart ((OS_TMR *)&MyTmr1,
                    (OS_ERR *)&err);
        /* Check “err” */
        // Continues in the next code listing!

//L12-2 Creating and Starting a timer
```
OSTmrTickCtr의 값이 12이기 때문에, 타이머는 OSTmrTickCtr이 13에 도달할 때 도는 timer task가 다음으로 신호를 받을 동안에 만료된다. 타이머는 다음 식을 이용하여 OSCfg_TmrWheel[] 테이블에 삽입된다:

```
MatchValue                  = OSTmrTickCtr + dly
Index into OSCfg_TmrWheel[] = MatchValue % OS_CFG_TMR_WHEEL_SIZE
```
여기서 "dly"는 OSTmrCreate()의 세 번째 argument(즉 이 예제에서 1)값이다. 이 예제를 사용하여 다시 다음과 같은 결과를 얻는다:

```c
MatchValue                   = 12 + 1
Index into OSCfg_TickWheel[] = 13 % 9

또는,

MatchValue                   = 13
Index into OSCfg_TickWheel[] = 4
```
테이블의 원형(circular) 특성(테이블의 크기를 이용하여 나머지 연산)때문에, 테이블은 timer wheel이라고 불리고, 각각의 항목은 wheel 내의 spoke이다.

타이머는 timer wheel(OSCfg_TmrWheel[])의 인덱스 4에 들어간다. 이 경우, OS_TMR은 리스트의 맨 앞에 배치되고(즉, OSCfg_TmrWheel[4].FirstPtr이 가리키고 있음), 인덱스 4의 항목의 수는 1 증가한다(즉, OSCfg_TmrWheel[4].NbrEntries는 1이 될 것이다). "MatchValue"는 OS_TMR 필드 .Match에 입력된다. 이것은 timer list의 인덱스 4에 들어간 첫 번째 타이머이기 때문에, .NextPtr 및 .PrevPtr은 모두 NULL을 가리킨다.

![Figure 12-9 Inserting a timer in the timer list](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/3aafea5b-98c0-462a-a39b-68d622268cfe)

아래 코드는 다른 타이머를 만들고 시작하는 코드이다. 이전 timer task가 신호를 받기 전에 실행되었다.

```c
    // Continuation of code from previous code listing.
    :
    :
    OSTmrCreate((OS_TMR *)&MyTmr2,
                (OS_CHAR *)"My Timer #2",
                (OS_TICK )10,
                (OS_TICK )0,
                (OS_OPT )OS_OPT_TMR_ONE_SHOT,
                (OS_TMR_CALLBACK_PTR)MyTmrCallbackFnct2,
                (void *)0,
                (OS_ERR *)&err);
    /* Check ’err” */
    OSTmrStart ((OS_TMR *)&MyTmr,
                (OS_ERR *)&err);
    /* Check ’err” */
    }
}
//L12-3 Creating and Starting a timer - continued
```
μC/OS-III는 match value와 index를 아래와 같이 계산한다.

```c
MatchValue                  = 12 + 10
Index into OSCfg_TmrWheel[] = 22 % 9

또는,

MatchValue                   = 22
Index into OSCfg_TickWheel[] = 4
```
'두 번쨰 타이머'는 Figure 12-10과 같이 같은 테이블 항목에 들어가지만, 대기 시간이 남아있는, 대기 시간이 가장 적은 타이머를 맨 앞에 배치하고, 대기 시간이 가장 긴 타이머를 맨 끝에 배치하도록 정렬된다.

![Figure 12-10 Inserting a second timer in the tick list](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/998e8005-b207-45df-b624-5e9c47073bb5)

timer task가 실행되면(os_tmr.c의 OS_TmrTask() 참조), OSTmrTickCtr을 증가시킴으로써 시작하여 업데이트가 필요한 테이블 항목(즉, spoke)를 결정한다. 그런 다음 그 항목 목록에 타이머가 있으면(즉, .FirstPtr이 NULL이 아니면), 각 OS_TMR을 검사하여 .Match 값이 "OSTmrTickCtr"과 일치하는지 여부를 결정하고, 만약 그렇다면, 해당 OS_TMR은 목록에서 제거되고, 타이머가 생성될 때 콜백 함수가 정의되었다고 가정하면, OS_TmrTask()는 타이머 콜백 함수를 호출한다. 리스트에서 검색은 OSTmrTickCtr이 타이머의 .Match 값과 일치하지 않는 순간 종료된다. 즉, 리스트는 이미 정렬되어 있으므로, 리스트에서 더 이상 탐색해도 의미가 없다.

OS_TmrTask()는 대부분의 작업을 스케줄러가 잠긴 상태에서 수행한다. 그러나 리스트가 정렬되고 리스트 검색에서 더 이상 일치하지 않는 즉시 종료되기 때문에 임계 구역은 상당히 짧다.

# Summary
타이머는 카운터가 0에 도달했을 때 지정된 행동을 하는 다운 카운터이다. 행동은 사용자가 정의한 콜백 함수로 제공된다.

μC/OS-III는 애플리케이션 코드가 임의의 수의 타이머를 생성할 수 있게 한다(RAM양에만 제한됨).

콜백 함수는 스케줄러가 잠긴 상태에서 timer task의 문맥에서 실행된다. 콜백 함수는 가능한 짧고 빠르게 실행되어야하고, blocking call을 하면 안된다.
# Reference
 - uC/OS-III: The Real-Time Kernel For the STM32 ARM Cortex-M3, Jean J. Labrosse, Micrium, 2009

[책 링크](https://micrium.atlassian.net/wiki/spaces/osiiidoc/overview)
