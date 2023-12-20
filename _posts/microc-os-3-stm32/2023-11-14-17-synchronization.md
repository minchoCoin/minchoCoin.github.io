---
title: "μC/OS-III ch.14 Synchronization"
last_modified_at: 2023-11-14T19:18:12+09:00
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

이 장에서는 task가 ISR(Interrupt Service Routine) 또는 다른 task와 동기화할 수 있는 방법을 설명한다.

ISR이 실행될 때, 이벤트가 발생했음을 task에게 신호를 줄 수 있다. task가 시그널링된 후(신호를 받은 후), ISR은 종료되고, 신호를 받은 task 우선순위에 따라 스케줄러가 실행된다. 신호를 받은 task는 이후 인터럽트를 발생시킨 디바이스를 서비스하거나 이벤트를 처리할 수 있다. 인터럽트를 발생시킨 디바이스를 task 레벨에서 서비스하는 것은 인터럽트가 비활성화되는 시간을 감소시키고 디버그하기 더 쉽기 때문에 task 레벨에서 서비스하는 것이 바람직하다.

μC/OS-III에서 동기화를 위한 두 가지 기본 메커니즘은 세마포어와 이벤트 플래그이다.

# Semaphores
231페이지의 13장 "Resource Management"에서 정의한 바와 같이, 세마포어는 대부분의 멀티태스킹 커널에서 제공하는 프로토콜 메커니즘이다. 세마포어는 원래 공유 자원에 대한 접근을 제어하기 위해 사용되었다. 그러나 12장에서 설명한 바와 같이 공유 자원에 대한 접근을 보호하기 위해 더 나은 메커니즘이 존재한다. 세마포어는 ISR을 task와 동기화하거나 Fig 14-1과 같이 task를 다른 task와 동기화하는 데 잘 사용된다.

세마포어는 이벤트의 발생을 알리기위해 사용된다는 것을 나타내기 위해 플래그(깃발)로 그려지는 것에 유의한다. 세마포어 초기 값은 보통 0이고, 이는 이벤트가 아직 발생하지 않았음을 나타낸다.

깃발 옆에 있는 값 "N"은 세마포어가 이벤트 또는 크레딧을 누적할 수 있음을 나타낸다. ISR 또는 task는 세마포어에 여러 번 post(또는 signal)할 수 있고, 세마포어는 post된 횟수를 기억할 것이다. 0이 아닌 다른 값으로 세마포어를 초기화하는 것이 가능한데, 이것은 세마포어가 처음에 그 수의 이벤트를 포함하고 있음을 나타낸다.

또한 신호를 수신하는 task 근처에 있는 작은 모래시계는 task가 타임아웃을 지정하는 옵션을 가지고 있음을 나타낸다. 이 타임아웃은 task가 특정 시간 내에 세마포어가 signal(또는 post)되기를 기다릴 것임을 나타낸다. 세마포어가 그 시간 내에 signal되지 않으면, μC/OS-III는 task를 재개하고, 세마포어가 signal되지 않고 타임아웃 때문에 task가 실행준비되었음을 나타내는 에러 코드를 반환한다.

![Figure 14-1 μC/OS-III Semaphore Services](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/b1eeae3b-5c4b-49d3-a5bf-8a4300f02851)

세마포어를 처리하는 함수는 표 14-1과 Fig 14-1에 정리된 바와 같이 여러 가지가 있다. 단, 이장에서는 가장 많이 사용되는 세 가지 함수, 즉 OSSemCreate(), OSSemPend(), OSSemPost()에 대해서만 논의하기로 한다. 다른 함수들은  443페이지의 Appendix A "μC/OS-III API Reference"에 설명되어 있다. 또한 모든 세마포어 함수는 task에서 호출할 수 있지만, ISR에서는 OSSemPost()만 호출할 수 있다.

| Function name | Operation |
|---------------|-----------|
|OSSemCreate()|세마포어를 만든다.|
|OSSemDel()|세마포어를 삭제한다.|
|OSSemPend()|세마포어를 기다린다.|
|OSSemPendAbort()|세마포어를 기다리는 것을 중지한다.|
|OSSemPost()|세마포어를 signal한다.|
|OSSemSet() |세마포어 count 값을 원하는 값으로 설정한다.|

동기화를 위해 사용될 때, 세마포어는 카운터를 사용하여 그것이 얼마나 많이 signal되었는지를 추적한다.  카운터는 세마포어 메커니즘이 각각 8비트, 16비트, 32비트중 어느것으로 구현되었는지에 따라 0과(255, 65,535 또는 4,294,967,295)사이의 값을 취할 수 있다. μC/OS-III의 경우, 세마포어의 최대 값은 OS_SEM_CTR(os_type.h 참조)에 의해 결정된다. 세마포어의 값과 함께, μC/OS-III는 세마포어가 signal되기를 기다리는 task를 추적한다.

## Unilateral Rendez-vous
Fig 14-2는 세마포어를 사용하여 task가 ISR(또는 다른 task)와 동기화할 수 있음을 보여준다. 이 경우 데이터는 교환되지 않으나, ISR 또는 (왼쪽의) task가 발생하였음을 나타낸다. 이러한 종류의 동기화를 위해 세마포어를 사용하는 것을 단방향 랑데부(unilateral rendez-vous)라고 부른다.

![Figure 14-2 Unilateral Rendezvous](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/36de9526-ca11-430a-a908-0e5af71759f9)

단방향 랑데부는 task가 I/O operation을 시작하고 세마포어가 signal(post)되기를 기다릴 때(즉, OSSemPend()) 사용된다. I/O operation이 완료되면, ISR(또는 다른 task)가 세마포어에 신호를 보내고(즉, OSSemPost()), task를 재개한다. 이 과정은 Fig 14-3의 타임라인에도 나타나 있으며, 후술한다. ISR과 task에 대한 코드는 L14-1에 있다.

![Figure 14-3 Unilateral Rendezvous, Timing Diagram](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/ec1dda9b-334e-4e3d-acdd-7e404961ff07)

### F14-3(1)
우선 순위가 높은 task가 실행 중이다. task는 ISR과 동기화(즉, ISR이 발생할 때까지 대기)가 필요하고, OSSemPend()를 호출한다.

### F14-3(2)(3)(4)
ISR이 발생하지 않았으므로 task는 이벤트가 발생할 때까지 세마포어의 대기 목록에 놓이게 된다. 그런 다음 μC/OS-III의 스케줄러는 다음으로 중요한 task를 고르고 해당 task로 문맥 교환을 할 것이다.

### F14-3(5)
낮은 우선순위의 task가 실행된다.

### F14-3(6)
원래 task가 대기하고 있던 이벤트가 발생한다. 우선순위가 낮은 task는 즉시 선점(인터럽트가 활성화되었다고 가정)되고, CPU는 해당 이벤트에 대한 인터럽트 핸들러를 실행한다.

### F14-3(7)(8)
ISR은 인터럽트를 발생시킨 장치를 처리한 다음, OSSemPost()를 호출하여 세마포어를 signal한다. ISR이 완료되면 μC/OS-III를 호출한다(즉, OSIntExit()).

### F14-3(9)(10)
μC/OS-III는 우선순위가 높은 task가 이 이벤트가 발생하기를 기다리고 있음을 알게 되고 컨텍스트가 원래 task로 다시 전환된다.

### F14-3(11)
원래 task가 재개되고, OSSemPend() 이후부터 실행된다.

## Unilateral Rendez-vous(2)
```c
OS_SEM MySem;

void MyISR (void)
{
    OS_ERR err;
    /* Clear the interrupting device */
    OSSemPost(&MySem, (7)
                OS_OPT_POST_1,
                &err);
    /* Check “err” */
}
void MyTask (void *p_arg)
{
    OS_ERR err;
    CPU_TS ts;
    :
    :
    while (DEF_ON) {
        OSSemPend(&MySem, (1)
                    10,
                    OS_OPT_PEND_BLOCKING,
                    &ts,
                    &err);
        /* Check “err” */ (11)
        :
        :
    }
}
//L14-1 Pending (or waiting) on a Semaphore
```
몇 가지 흥미로운 점들에 주목할 필요가 있다. 첫째, task는 세부적인 내용을 알 필요가 없다. task는 자신이 기다리고 있는 이벤트가 발생하면 리턴되는 함수 OSSemPend()를 호출하기만 하면된다. 둘째, μC/OS-III는 그다음으로 가장 중요한 task를 선택함으로써 CPU의 사용을 극대화하는데, 이 task는 ISR이 발생할 때까지 실행된다. 사실 ISR은 수 밀리초 동안 발생하지 않을 수 있고, 그 시간 동안 CPU는 다른 task들을 실행할 것이다. 세마포어를 기다리는 task는 기다리는 동안 CPU 시간을 소모하지 않는다. 마지막으로, 세마포어를 기다리는 task는(가장 중요한 task라 가정)이벤트가 발생한 후에 즉시 실행될 것이다.

## Credit Tracking
앞서 언급한 바와 같이 세마포어는 몇 번 signal(또는 post)되었는지를 기억한다. 즉, 이벤트를 대기하고 있는 task가 가장 우선순위가 높은 task가 되기 전에 ISR이 여러 번 발생하면, 세마포어는 자신이 signal된 횟수를 유지할 것이다. task가 가장 우선순위가 높은 실행준비 상태의 task가 되면, ISR이 발생한 횟수만큼 task가 blocking 되지 않고 실행될 것이다. 이를 *credit tracking* 이라 하며, Fig 14-4에 예시가 있으며, 아래에서 설명한다.

![Figure 14-4 Semaphore Credit Tracking](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/551bfd3f-3af2-4937-b993-49b862ec4e1a)

### F14-4(1)
높은 우선순위의 task가 실행 중이다.

### F14-4(2)(3)
우선순위가 낮은 task를 위한 이벤트가 발생한다. ISR은 세마포어를 post한다. 이 시점에서 카운트는 1이다.

### F14-4(4)(5)(6)
ISR이 더 높은 우선순위의 task를 실행 준비 상태로 만들었는지 알아보기 위해 ISR 마지막에 μC/OS-III가 호출된다. 해당 ISR은 더 낮은 우선순위의 task가 대기하고 있던 이벤트였기 때문에 μC/OS-III은 더 높은 우선순위의 task의 실행을 재개한다.

### F14-4(7)
높은 우선순위의 task가 재개된다.

### F14-4(8)(9)
인터럽트가 다시 발생한다. ISR은 세마포어를 post한다. 이 때 세마포어 카운트는 2이다.

### F14-4(10)(11)(12)
ISR이 더 높은 우선순위의 task를 실행 준비 상태로 만들었는지 알아보기 위해 ISR 마지막에 μC/OS-III가 호출된다. 해당 ISR은 더 낮은 우선순위의 task가 대기하고 있던 이벤트였기 때문에 μC/OS-III은 더 높은 우선순위의 task의 실행을 재개한다.

### F14-4(13)(14)
우선순위가 높은 task는 실행을 재개하고 작업이 끝난다. 그런 다음 이 task는 μC/OS-III 서비스들 중 하나를 호출하여 이벤트가 발생하기를 기다릴 것이다.

### F14-4(15)(16)
그런 다음 μC/OS-III는 다음으로 중요한 task를 선택할 것이며, 이 task는 해당 이벤트를 기다리는 task이며 상황에 따라 해당 task로 전환될 것이다.

### F14-4(17)
새 task는 세마포어 카운트가 2이므로 ISR이 두번 발생했음을 실행되고 알 것이다. task는 이 이벤트를 처리할 것이다.

## Multiple Tasks Waiting On A Semaphore
![Figure 14-5 Multiple Tasks waiting on a Semaphore](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/f9a866d3-89e8-418f-90be-1b37e0d0bef7)

세마포어가(ISR이나 task에 의해) 신호를 받으면, μC/OS-III는 세마포어에서 대기하는 가장 높은 우선순위의 task를 실행 준비 상태로 만든다. 그러나 세마포어에서 대기하는 모든 task를 실행 준비 상태로 만드는 것도 가능하다. 이것을 브로드캐스팅(broadcasting)이라고 하며 OSSemPost()를 호출할 때 옵션으로 OS_OPT_POST_ALL을 지정하면 수행된다. 세마포어를 대기하고있는 task 중 어떤 것이라도 이전에 실행된 task보다 더 높은 우선순위를 가진다면 μC/OS-III는 OSSemPost()에 의해 실행 준비가 된 가장 높은 우선순위의 task를 실행할 것이다.

브로드캐스팅은 여러 task를 동기화하고, 동시에 실행하도록 하게 하는데 쓰이는 일반적인 기술이다. 그러나 동기화하고자하는 task 중 일부는 세마포어를 기다리지 않을 수도 있다. 세마포어와 이벤트 플래그를 결합하면 이 문제를 해결하기가 쉽다. 이에 대해서는 이벤트 플래그를 설명한 후 설명한다.

## Semaphore Internals (For Synchronization)
여기에 제시된 자료 중 일부는 231페이지의 13장 "Resource Management"에도 수록되어 있는데, 그 장에서도 세마포어에 대해 논의한 바 있다. 다만 여기에 제시된 자료는 동기화에 사용되는 세마포어에도 적용 가능할 것이므로, 다소 차이가 있을 것이다.

카운팅 세마포어는 세마포어 메커니즘이 각각 8비트, 16비트, 32비트로 구현되었는지에 따라 0에서 255, 65535, 또는 4294967295 사이의 값을 허용한다. μC/OS-III의 경우, 세마포어의 최대값은 필요에 따라 변경할 수 있는 자료형 OS_SEM_CTR(os_type.h 참조)에 의해 결정된다. 세마포어의 값과 함께 μC/OS-III는 세마포어를 대기하는 task들을 추적한다.

애플리케이션 프로그래머는 무제한의 세마포어(사용가능한 RAM양에만 제한됨)를 만들 수 있다. μC/OS-III의 세마포어 서비스는  OSSem???()으로 시작하며 애플리케이션 프로그래머가 이용할 수 있는 서비스는 443페이지의 Appendix A "μC/OS-III API Reference"에 설명되어 있다. 세마포어 서비스는 os_cfg.h에 정의되어있는 상수 OS_CFG_SEM_EN을 1로 설정함으로써 컴파일 타임에 활성화 된다.

앞서 언급한 바와 같이, 세마포어는 OS_SEM 자료형으로 정의된 커널 객체이며, 이는 L14-2와 같이 os_sem(os.h 참조)로 정의된다. 세마포어를 관리하기 위해 μC/OS-III가 제공하는 서비스들은 os_sem.c 파일에 구현되어있다.

```c
typedef struct os_sem OS_SEM; (1)

struct os_sem {
    OS_OBJ_TYPE Type; (2)
    CPU_CHAR *NamePtr; (3)
    OS_PEND_LIST PendList; (4)
    OS_SEM_CTR Ctr; (5)
    CPU_TS TS; (6)
};
//L14-2 OS_SEM data type
```

### L14-2(1)
μC/OS-III에서는 모든 구조체를 자료형으로 다시 정의해놓았다. 모든 자료형은 "OS_"로 시작하며 모두 대문자이다. 세마포어를 선언할 때는 단순히 세마포어를 선언할 때 변수의 데이터 타입으로 OS_SEM을 사용한다. 

### L14-2(2)
구조체는 "Type" 필드로 시작하며, 이를 통해 μC/OS-III가 세마포어로 인식할 수 있다. 즉, 다른 커널 객체들도 구조체의 첫 번째 멤버로서 "Type"을 가질 것이다. 만약 함수가 커널 객체를 전달받으면, μC/OS-III는 올바른 자료형을(os_cfg.h에서 OS_CFG_OBJ_TYPE_CHK_EN을 1로 설정한다고 가정) 전달하고 있으믕ㄹ 확인할 것이다. 예를 들어 메시지 큐(OS_Q)를 세마포어 서비스(예를 들어 OSSemPend())로 전달하면, μC/OS-III는 잘못된 객체가 전달된 것을 인식하고, 그에 따른 에러 코드를 반환할 것이다.

### L14-2(3)
각 커널 객체는 디버거나 μC/Probe에서 쉽게 알아볼 수 있도록 이름을 부여받을 수 있다. 이 멤버는 단순히 NUL문자로 끝나는 ASCII 문자열을 가리키는 포인터이다.

### L14-2(4)
세마포어 상에서 다순의 task들이 대기하는 것이 가능하기 때문에, 세마포어는 197페이지의 10장 "Pend Lists(or Wait Lists)"에 설명한 pend list를 포함한다.

### L14-2(5)
세마포어는 카운터를 포함한다. 카운터는 os_type.h의 OS_SEM_CTR이 어떻게 선언되었는지에 따라 8비트, 16비트, 32비트 값으로 구현될 수 있다. μC/OS-III는 이 카운터로, 세마포어가 시그널링되는 횟수를 추적하며, 이 필드는 일반적으로 OSSemCreate()에 의해 0으로 초기화된다.

### L14-2(6)
세마포어는 타임스탬프를 포함하며, 이는 세마포어가 마지막으로 시그널링된 시간(또는 post된 시간)을 나타내기 위해 사용된다. μC/OS-III는 응용 프로그램이 시간 측정을 할 수 있게 해주는 free-running 카운터가 존재한다고 가정한다. 세마포어가 시그널링될 때 free-running 카운터가 읽혀지고 값이 이 필드에 놓이게 되는데, 이 값은 OSSemPend()가 호출될 때 반환된다. 이 값을 통해 응용 프로그램은 시그널링이 언제 발생하였는지, 또는 세마포어가 시그널링된 후부터 task가 CPU를 제어하는데 걸리는 시간을 알 수 있다. 후자의 경우, OS_TS_GET()을 호출하여 현재 타임스탬프를 확인하고, 차이를 계산해야한다.

## Semaphore Internals (For Synchronization) (2)
OS_SEM 내부를 이해하는 사용자라도, 이 자료구조의 어떤 필드에도 애플리케이션 코드가 직접 접근해서는 절대 안된다. 대신 항상 μC/OS-III와 함께 제공되는 API를 사용해야 한다.

응용 프로그램이 세마포어를 사용하려면 먼저 세마포어를 만들어야 한다. L14-3은 세마포어를 만드는 방법을 보여준다.

```c
OS_SEM MySem; (1)

void MyCode (void)
{
    OS_ERR err;
    :
    OSSemCreate(&MySem, (2)
                "My Semaphore", (3)
                (OS_SEM_CTR)0, (4)
                &err); (5)
    /* Check “err” */
    :
}
//L14-3 Creating a Semaphore
```

### L14-3(1)
응용 프로그램은 OS_SEM 타입의 변수를 선언해야한다. 이 변수는 세마포어 서비스에서 참조될 것이다.

### L14-3(2)
OSSemCreate()를 호출하고, [L14-3(1)](#l14-31)에서 선언한 세마포어의 주소를 넘겨줌으로써 세마포어를 생성한다.

### L14-3(3)
세마포어에 ASCII 이름을 지정할 수 있으며, 이 이름은 디버거 또는 μC/Probe에서 세마포어를 쉽게 식별할 수 있게 한다.

### L14-3(4)
세마포어를 시그널링 메커니즘으로 사용하는 경우, 세마포어를 0으로 초기화해야한다.

### L14-3(5)
OSSemCreate()는 호출의 결과에 따라 오류 코드를 반환한다. 모든 인수가 유효하면 err는 OS_ERR_NONE 이다.

## Semaphore Internals (For Synchronization) (3)
OSSemCreate()는 이 함수에 전달되는 argument를 검사하고(os_cfg.h에서 OS_CFG_ARG_CHK_EN이 1로 설정되어있다고 가정) 시그널링에 사용되는 OS_SEM 타입의 변수만 초기화한다.

task는 L14-4에 나타낸 바와 같이 OSSemPend()를 호출하여 ISR또는 다른 task로부터의 신호를 기다린다(argument에 대한 자세한 내용은 443페이지 Appendix A “μC/OS-III API Reference” )를 참조한다.

```c
void MyTask (void *p_arg)
{
    OS_ERR err;
    CPU_TS ts;
    :
    while (DEF_ON) {
        OSSemPend(&MySem, (1)
                    0,
                    OS_OPT_PEND_BLOCKING,
                    &ts,
                    &err);
        /* Check “err” */ (2)
        :
    }
}
```
### L14-4(1)
OSSemPend()가 호출되면, 이 함수에 전달된 argument가 유요한 값을 갖는지 확인하는 것으로 시작한다(os_cfg.h에서 OS_CFG_OBJ_TYPE_CHK_EN이 1로 설정되어있다고 가정).

세마포어 카운터(OS_SEM의 .Ctr)가 0보다 크면 카운터가 감소하고 OSSemPend()가 리턴되는데, 이는 신호가 발생했음을 의미한다. 이는 기대하는 결과이다.

세마포어 카운터가 0이면 신호가 발생하지 않았음을 의미하며, OSSemPend()를 호출한 task는 세마포어가 방출될 때까지 기다려야할 수 있다. OS_OPT_PEND_NON_BLOCKING 옵션을 지정하면 OSSemPend()는 즉시 리턴되며, 반환된 오류 코드는 신호가 발생하지 않았음을 표시한다.

옵션으로 OS_OPT_PEND_BLOCKING을 지정하면 세마포어가 시그널링되기를 기다리는 task의 목록에 OSSemPend()를 호출한 task가 삽입된다. task는 Fig14-6과 같이 우선순위 순서에 따라 list에 삽입된다(가장 높은 우선순위의 task가 list의 시작부분에 있음).

0이 아닌 타임아웃을 지정하면, task는 tick list에도 삽입된다. 0 타임아웃 값은 OSSemPend()를 호출한 task가 세마포어가 시그널링될 때까지 영원히 기다릴 의향이 있음을 나타낸다.

현재 task가 실행될 수 없으므로, 스케줄러가 호출된다. 스케줄러는 실행 준비가 된, 다음으로 우선순위가 높은 task를 실행할 것이다.

세마포어가 시그널링되고 OSSemPend()를 호출한 task가 가장 높은 우선순위의 task가 되면, task상태를 검사하여 OSSemPend()가 리턴되는 이유를 확인한다. 가능성은 다음과 같다.
1. 세마포어가 시그널링되었다. 이것이 원하는 결과이다.
2. 다른 task에 의해 대기가 중단되었다.
3. 세마포어가 특정 타임아웃 내로 시그널링되지 않았다.
4. 세마포어가 삭제되었다.

OSSemPend()가 리턴되었을 때, OSSemPend()를 호출한 task는 오류 코드로 위 결과를 전달받을 것이다.

### L14-4(2)
OSSemPend()가 err이 OS_ERR_NONE으로 지정되어 리턴되면, 세마포어가 시그널링되었다고 가정하고 ISR또는 세마포어를 시그널링한 task를 처리할 수 있다. err에 다른 것이 포함되어 있는 경우, 즉 타임아웃 되었거나(타임아웃 argument가 0이 아닌 경우), 다른 task에 의해 대기가 중단되었거나, 세마포어가 삭제된 경우가 있을 수 있다. 따라서 반환된 오류 코드를 검사하고 모든 것이 예상대로 진행되었다고 가정하지 않는 것이 항상 중요하다.

## Semaphore Internals (For Synchronization) (4)
ISR이나 task가 다른 task를 시그널링하기 위해서는 L14-5에 나와있는 것처럼 OSSemPost()를 호출하면 된다.

```c
OS_SEM MySem;

void MyISR (void)
{
    OS_ERR err;
    :
    OSSemPost(&MySem, (1)
                OS_OPT_POST_1, (2)
                &err); (3)
    /* Check “err” */
    :
    :
}
```
### L14-5(1)
task가 OSSemPost()를 호출하여 세마포어를 시그널링한다. 세마포어의 주소를 전달하여, post할 세마포어를 지정한다. 세마포어는 미리 생성되어있어야한다.

### L14-5(2)
다음 argument는 task가 post하고자하는 방법을 명시한다. 선택할 수 있는 옵션은 여러 가지가 있다.

OS_OPT_POST_1을 지정하면 (세마포어에 대기 중인 task가 여러 개인 경우) 하나의 task만 post하고자 함을 나타내고 있다. 실행 준비가 될 task는 대기 중인 task 중 가장 높은 우선순위의 task이다. 만일 동일한 우선순위에 있는 task가 여러 개인 경우, 그 중 하나만 실행준비상태가 될 것이다. Fig14-6에서 보는 바와 같이 대기 중인 task의 목록은 우선순위 순으로 배치된다(HPT는 높은 우선순위의 task, LPT는 낮은 우선순위의 task). 따라서 list에서 HPT를 추출하는 것은 빠른 연산이다.

만약 OS_OPT_POST_ALL을 지정하면, 세마포어에서 대기하는 모든 task가 post되고 실행준비상태가 될 것이다.

OS_OPT_POST_NO_SCHED 옵션을 이전의 두 옵션 중 하나에 추가하여 스케줄러가 OSSemPost() 마지막에 호출되지 않도록 할 수 있는데, 이는 추가적인 post가 수행될 것이어서, 스케줄링을 모든 post가 완료된 후 수행해야할 때 필요하다. 이는 시그널링이 되지만, 더 높은 우선순위의 task가 해당 세마포어를 대기하고 있었더라도 스케줄러가 호출되지 않음을 의미한다. 이를 통해 OSSemPost()를 호출한 task는 추가로 post할 수 있으며, 모든 post를 동시에 유효하게 할 수 있다. OS_OPT_POST_NO_SCHED는 이전 옵션 중 하나와 함께 사용할 수 있음을 주목하라. 따라서 다음과 같이 옵션을 지정할 수 있다:
```c
OS_OPT_POST_1
OS_OPT_POST_ALL
OS_OPT_POST_1 + OS_OPT_POST_NO_SCHED
OS_OPT_POST_ALL + OS_OPT_POST_NO_SCHED
```
![Figure 14-6 Tasks waiting for semaphore](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/f3c16088-fdfd-4ab5-ae92-49b5604e6c90)

### L14-5(3)
OSSemPost()는 호출 결과에 따라 에러 코드를 반환한다. 호출이 성공했다면, err는 OS_ERR_NONE이다. 그렇지 않다면 에러코드는 에러의 이유를 나타낼 것이다(OSSemPost()에 대한 가능한 에러 코드는 443페이지의 Appendix A "μC/OS-III API Reference" 참조)

# Task Semaphore
세마포어를 사용하여 task를 시그널링하는 것은 보편적인 동기화 방법으로, μC/OS-III에서는 각 task들은 고유하게 내장된 세마포어를 가진다. 이 기능은 코드를 단순화할 뿐만 아니라 별도의 세마포어 객체를 사용하는 것보다 더 효율적이다. 각 task에 내장된 세마포어는 Fig 14-7과 같다.

μC/OS-III의 task 세마포어 서비스는 OSTaskSem??() 접두사로 시작하며, 애플리케이션 프로그래머가 사용할 수 있는 서비스는 443페이지의 Appendix A "μC/OS-III API Reference"에 설명되어있다. task 세마포어는 μC/OS-III에 내장되어 있으며, 컴파일 시 다른 서비스처럼 비활성화할 수 없다. task 세마포어에 대한 코드는 os_task.c에 있다.

이벤트가 발생했을 때 어떤 task를 시그널링해야하는지를 코드가 알고 있다면 이 기능을 사용할 수 있다. 예를 들어 이더넷 컨트롤러로부터 인터럽트를 받으면 수신된 패킷의 처리를 담당하는 task에 신호를 보낼 수 있는데 이 처리는 ISR 대신 task를 사용하여 수행하는 것이 바람직하기 때문이다.

![Figure 14-7 Semaphore built-into a Task](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/4bcd373a-bc23-4135-b5d4-9b665f127484)

표 14-2에 요약된, task 세마포어에 대해 수행할 다양한 함수가 있다.

| Function Name | Operation |
|---------------|-----------|
|OSTaskSemPend()|task 세마포어를 대기한다.           |
|OSTaskSemPendAbort()|task 세마포어 대기를 중단한다.           |
|OSTaskSemPost()|task를 시그널링한다.           |
|OSTaskSemSet()|세마포어 카운트를 원하는 값으로 설정한다.           |

(표 14-2 Task Semaphore API summary)

## Pending(i.e., Waiting) On A Task Semaphore
task가 생성되면 초기값이 0인 내부 세마포어가 자동으로 생성된다. L14-6과 같이 task 세마포어를 대기하는 것은 상당히 간단하다.

```c
void MyTask (void *p_arg)
{
    OS_ERR err;
    CPU_TS ts;
    :
    while (DEF_ON) {
        OSTaskSemPend(10, (1)
                        OS_OPT_PEND_BLOCKING, (2)
                        &ts, (3)
                        &err); (4)
        /* Check “err” */
    :
    :
    }
}
```

### L14-6(1)
task는 OSTaskSemPend()를 호출함으로써 task 세마포어를 대기한다. 현재 task가 대기하는 것으로 가정하기 때문에, 어떤 task를 특정할 필요가 없다. 첫 번째 argument는 clock tick 단위로 지정된 타임아웃이다. 타임아웃은 tick rate에 의존한다. tick rate(os_cfg_app.h 참조)가 1000으로 설정되면, 10틱의 타임아웃은 10 밀리초를 나타낸다. 타임아웃을 0으로 지정하는 것은 task가 task 세마포어를 영원히 대기하는 것을 의미한다.

### L14-6(2)
두 번째 argument는 대기 방법을 지정한다. OS_OPT_PEND_BLOCKING 과 OS_OPT_PEND_NON_BLOCKING 두 가지 옵션이 있다. blocking 옵션은 task 세마포어가 시그널링(또는 post)되지 않은 경우, 세마포어가 시그널링될 때까지 또는 다른 task에 의해 대기가 중단되거나, 타임아웃이 만료될 때까지 task가 대기한다는 것을 의미한다.

### L14-6(3)
세마포어가 시그널링되면, μC/OS-III는 타임스탬프를 읽어 (신호를 받는)task의 OS_TCB에 넣는다. OSTaskSemPend()가 리턴될 때 타임스탬프 값을 로컬 변수 ts에 넣는다. 이 기능은 시그널링이 실제로 발생한 시점을 알려준다. OS_TS_GET()을 호출하여 현재 타임스탬프를 읽고 차이를 계산할 수 있다. 그러면 task가 post를 한 task나 ISR로부터 시그널링되는데 걸린 시간을 확인할 수 있다.

### L14-6(4)
OSTaskSemPend()는 호출 결과에 따라 에러 코드를 반환한다. 호출이 성공적이었다면 err는 OS_ERR_NONE이다. 그렇지 않다면 에러 코드는 에러 원인을 나타낼 것이다(443페이지의 Appendix A "μC/OS-III API Reference" 참조).

## Posting(i.e., Signaling) A Task Semaphore
ISR 또는 task는 L14-7과 같이 OSTaskSemPost()를 호출하여 task에 신호를 보낸다.
```c
OS_TCB MyTaskTCB;

void MyISR (void *p_arg)
{
    OS_ERR err;
    :
    OSTaskSemPost(&MyTaskTCB, (1)
                    OS_OPT_POST_NONE, (2)
                    &err); (3)
    /* Check “err” */
    :
    :
}
//L14-7 Posting (or signaling) a Semaphore
```

### L14-7(1)
task는 OSTaskSemPost()를 호출하여 task를 post(또는 signal)한다. 원하는 task의 OS_TCB주소를 전달해야 하며, 물론 task가 존재해야한다.

### L14-7(2)
다음 argument는 post하고자하는 방법을 선택한다. 선택할 수 있는 방법은 2가지이다.

OS_OPT_POST_NONE는 세마포어를 post한 후 스케줄러를 호출하는 기본 옵션이다.

또는 OS_OPT_POST_NO_SCHED를 통해 스케줄러가 OSTaskSemPost()의 마지막에 호출되지 않도록 표시한다. 추가 post가 있을 수 있고, 모든 post 후 스케줄링이 일어나기 원하기 때문일 수 있다(마지막 post는 이 옵션을 지정하지 않을 것이다).

OSTaskSemPost()는 호출의 결과에 따라 에러 코드를 반환한다. 성공적이였다면 에러는 OS_ERR_NONE이다. 그렇지 않다면 에러 코드는 에러 이유를 나타낼 것이다(OSTaskSemPost()에 대한 가능한 에러 코드 목록은 Appendix A, 443페이지의 "μC/OS-III API Reference"를 참조한다).

## Bilateral Rendez-Vous
두 개의 task는 Fig 14-8과 같이 두 개의 task 세마포어를 사용하여 동기화할 수 있으며, 이를 *bilateral rendez-vous* 라고 한다. bilateral rendez-vous는 진행하기 전에 두 개의 task가 서로 동기화되있어야 한다는 점을 제외하고 일반적인 rendez-vous와 유사하다. ISR은 세마포어를 대기할 수 없기 때문에 task와 ISR사이에서 bilateral rendez-vous를 수행할 수 없다.

![Figure 14-8 Bilateral Rendezvous](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/750a5835-0d73-407d-b224-55d64e2b4963)

L14-8은 bilateral rendez-vous를 수행하기 위한 코드이다. 물론 bilateral rendez-vous는 두 개의 서로 다른 세마포어를 사용할 수도 있지만, 내장된 task 세마포어를 사용하면 이러한 유형의 동기화를 설정할 수 있다.

```c
OS_TCB MyTask1_TCB;
OS_TCB MyTask2_TCB;

void Task1 (void *p_arg)
{
    OS_ERR err;
    CPU_TS ts;

    while (DEF_ON) {
    :
    OSTaskSemPost(&MyTask2_TCB, (1)
                    OS_OPT_POST_NONE,
                    &err);
    /* Check ’err” */
    OSTaskSemPend(0, (2)
                    OS_OPT_PEND_BLOCKING,
                    &ts,
                    &err);
    /* Check ’err” */
    :
    }
}
void Task2 (void *p_arg)
{
    OS_ERR err;
    CPU_TS ts;

    while (DEF_ON) {
    :
    OSTaskSemPost(&MyTask1_TCB, (3)
                    OS_OPT_POST_NONE,
                    &err);
    /* Check ’err” */
    OSTaskSemPend(0, (4)
                    OS_OPT_PEND_BLOCKING,
                    &ts,
                    &err);
    /* Check ’err” */
    :
    }
}
//L14-8 Tasks synchronizing their activities
```

### L14-8(1)
Task 1이 실행되고, Task 2 세마포어를 시그널링한다.

### L14-8(2)
Task 1은 Task 2와 동기화 하기 위해 자신의 내부 세마포어를 대기한다. Task #2가 아직 실행되지 않았기 때문에, Task 1은 시그널링될 자신의 세마포어를 대기하면서 실행이 중단된다. μC/OS-III는 Task 2로 문맥교환한다.

### L14-8(3)
Task 2가 실행되고, Task 1 세마포어를 시그널링한다.

### L14-8(4)
이미 시그널링되었기 때문에, Task 2는 Task 1과 동기화된다. Task 1이 Task2보다 우선순위가 높으면, μC/OS-III는 Task 1으로 다시 전환될 것이다. 그렇지 않으면, Task 2는 실행을 계속한다.

# Event Flags
이벤트 플래그는 task가 여러 이벤트 발생과 동기화해야 할 때 사용된다. task는 이벤트 중 어느 하나라도 발생했을 때 동기화될 수 있는데 이를 disjunctive synchronization(logical OR)라고 한다. task가 모든 이벤트가 발생했을 때 동기화될 수 있는데, 이를 conjunctive synchronization(logical AND)라고 한다. disjunctive와 conjuctive synchronization은 Fig 14-9와 같다.

애플리케이션 프로그래머는 무제한의 이벤트 플래그 그룹(사용 가능한 RAM에만 제한됨)을 만들 수 있다. μC/OS-III에서 이벤트 플래그 서비스는 OSFlag??() 접두사로 시작한다. 애플리케이션 프로그래머가 사용할 수 있는 서비스는 443페이지 Appendix A, "μC/OS-III API Reference"에 설명되어 있다.

이벤트 플래그 서비스에 대한 코드는 os_flag.c 파일에 있으며, os_cfg.h에 OS_CFG_FLAG_EN을 1로 설정하여 활성화할 수 있다.

![Figure 14-9 Event Flags](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/dd8cc0f7-f9ab-450a-9980-4ac7253439a9)

## F14-9(1)
μC/OS-III "이벤트 플래그 그룹"은 OS_FLAG_GRP 타입의 커널 객체이며, 일련의 비트들(os_type.h에 정의된 자료형 OS_FLAGS에 따라 8,16,32비트)로 구성된다. 이벤트 플래그 그룹은 비트들 중 일부(또는 모든)가 set돠거나 clear되기를 기다리는 task들의 목록을 가지고 있다. 이벤트 플래그 그룹은 task과 ISR이 사용하기 전에 먼저 생성되어야 한다. 이벤트 플래그는 μC/OS-III를 시작하기 전에, 또는 애플리케이션 코드의 시작 task에 의해 생성되어야 한다.

## F14-9(2)
task 또는 ISR은 이벤트 플래그에 post할 수 있다. task만 이벤트 플래그 그룹을 생성하고, 삭제하고, 다른 task가 대기하는 것을 중단할 수 있다.

## F14-9(3)
task는 이벤트 플래그 그룹의 임의의 수의 비트들에 대해 대기할 수 있다. 모든 μC/OS-III pend와 마찬가지로, task는 원하는 비트들이 지정된 시간(틱)내에 post되지 않으면 task가 재개되도록 타임아웃 값을 지정할 수 있다.

## F14-9(4)
task는 비트들의 subset 중 어떤 bit라도 set(또는 clear)될 때까지 기다릴 것인지(OR), 또는 subset의 모든 bit가 set(또는 clear)될 때까지 기다릴 것인지(AND) 지정할 수 있다.

# Event Flags(2)
표 14-3에 요약된 바와 같이, 이벤트 플래그와 관련된 다양한 함수가 있다.

| Function Name | Operation |
|---------------|-----------|
|OSFlagCreate()|이벤트 플래그 그룹을 만든다.|
|OSFlagDel()|이벤트 플래그 그룹을 삭제한다.|
|OSFlagPend()|이벤트 플래그 그룹을 기다린다.|
|OSFlagPendAbort()|이벤트 플래그 그룹 대기를 중단한다.|
|OSFlagPendGetFlagsRdy()|task를 ready상태로 만든 플래그를 얻는다.|
|OSFlagPost()|이벤트 플래그 그룹에 플래그를 post한다.|

(표 14-3 Event Flags API summary)

## Using Event Flags
task 또는 ISR이 이벤트 플래그 그룹에 post하면, 대기하고 있던 task는 모두 재개된다.

이벤트 플래그 그룹의 각 비트가 의미하는 바를 결정하는 것은 애플리케이션에 달려 있으며, 필요한 만큼 많은 이벤트 플래그 그룹을 사용하는 것이 가능하다. 예를 들어 이벤트 플래그 그룹에서 0번째 비트는 온도 센서가 너무 낮음을 나타내고, 1번째 비트는 배터리 전압이 낮음을 나타내고, 2번째 비트는 스위치가 눌려졌음을 나타낸다고 정의할 수 있다. 이러한 조건을 감지하는 코드(task 또는 ISR)는 OSFlagPost()를 호출하여 적절한 이벤트 플래그를 설정하고, 이러한 조건에 응답하는 task(들)는 OSFlagPend()를 호출한다

L14-9는 이벤트 플래그를 어떻게 사용하는지 보여준다.

```c
#define TEMP_LOW (OS_FLAGS)0x0001 (1)
#define BATT_LOW (OS_FLAGS)0x0002
#define SW_PRESSED (OS_FLAGS)0x0004

OS_FLAG_GRP MyEventFlagGrp; (2)

void main (void)
{
    OS_ERR err;

    OSInit(&err);
    :
    OSFlagCreate(&MyEventFlagGrp, (3)
                "My Event Flag Group",
                (OS_FLAGS)0,
                &err);
    /* Check ’err” */
    :
    OSStart(&err);
}
void MyTask (void *p_arg) (4)
{
    OS_ERR err;
    CPU_TS ts;

    while (DEF_ON) {
            OSFlagPend(&MyEventFlagGrp, (5)
            TEMP_LOW + BATT_LOW,
            (OS_TICK )0,
            (OS_OPT)OS_OPT_PEND_FLAG_SET_ANY,
            &ts,
            &err);
        /* Check ’err” */
        :
    }
}

void MyISR (void) (6)
{
    OS_ERR err;
    :
    OSFlagPost(&MyEventFlagGrp, (7)
                BAT_LOW,
                (OS_OPT)OS_OPT_POST_FLAG_SET,
                &err);
    /* Check ’err” */
    :
}
```

### L14-9(1)
이벤트 플래그 그룹에 사용할 비트를 정의해야 한다.

### L14-9(2)
OS_FLAG_GRP 타입의 변수를 선언해야한다. 이 변수는 이 이벤트 플래그 그룹을 사용하는 모든 μC/OS-III함수에서 참조될 것이다. 논의를 위해, 이벤트 플래그들이 os_type.h에서 16bit로 선언된다고 가정한다(즉, COU_INT16U 타입).

### L14-9(3)
이벤트 플래그 그룹은 사용되기 전에 먼저 생성되어야한다. 이벤트 플래그 그룹을 생성하기 위한 최적의 함수는 시작 코드이다. 시작 코드에 이벤트 플래그 그룹을 생성하면 어떤 task나 ISR도 μC/OS-III가 시작될 때까지 이벤트 플래그 그룹을 사용할 수 없도록 보장한다. 다시 말해서, 최적의 장소는 이벤트 플래그 그룹을 main()에 생성하는 것이다. 예제에서, 이벤트 플래그는 이름이 주어졌고 모든 비트는 클리어된 상태(즉, 모두 0)에서 시작한다.

### L14-9(4)
여기서 애플리케이션이 이벤트 플래그 그룹을 대기할 예정인 MyTask()를 생성했다고 하자.

### L14-9(5)
이벤트 플래그 그룹을 대기하려면, OSFlagPend()를 호출하여 원하는 이벤트 플래그 그룹의 주소를 전달해야한다.

두 번째 argument는 set될 때까지 대기할 bit를 지정한다(task가 clear될 bit가 아니라 set될 bit를 기다린다고 가정).

또한 이러한 비트가 set될 때까지 대기할 시간을 설정해야한다. 타임아웃 값이 0이면 비트가 set될 때까지 영원히 대기할 것임을 나타낸다. 0이 아닌 값은 최대로 기다릴 틱 수를 나타낸다.

OS_OPT_FLAG_SET_ANY를 지정하면, 지정된 두 비트 중 하나가 설정되면 task가 깨어남을 나타낸다.

이벤트 플래그 그룹이 post될 때 타임스탬프가 읽혀지고 저장된다. 이 타임스탬프는 이벤트에 대한 응답시간을 확인하는데 사용될 수 있다.

OSFlagPend()는 전달된 argument에 대해 여러 검사를 수행하고(즉, NULL 포인터 전달, 잘못된 옵션 전달), 호출 결과에 따라 오류 코드를 반환한다(os_cfg.h에서 OS_CFG_ARG_CHK_EN이 1로 설정되어있다고 가정). 호출이 성공하면 err는 OS_ERR_NONE이다.

### L14-9(6)
ISR은 제품의 배터리 전압이 낮을 때(제품이 배터리로 작동된다고 가정할 때) 감지하도록 설정된다. ISR은 해당 task에 신호를 보내며 해당 task가 필요한 모든 조치를 수행하도록 한다.

### L14-9(7)
post 호출에도 post할 이벤트 플래그 그룹의 위치를 지정하고, 어떤 플래그를 설정할 것인지도 설정한다. 세 번째 옵션은 비트가 set됨으로서 플래그될 것임을 지정한다. 그리고 함수는 호출의 결과에 따라 err을 설정한다.

## Using Event Flags(2)
이벤트 플래그는 일반적으로 status와 transient event 두 가지 목적으로 사용된다. 일반적으로 L14-10과 같이 이들 각각을 처리하기 위해 다른 이벤트 플래그 그룹을 사용할 것이다.

Task 또는 ISR은 특정 값을 초과한 온도, 엔진 또는 모터의 RPM이 0이거나 탱크의 연료가 있는 등의 상태 정보를 보고할 수 있다. 상태가 다른 task 또는 ISR에 의해 관리되기 때문에 이러한 상태 정보는 이러한 이벤트를 대기하는 task에 의해 처리될 수 없다. 상태 정보와 연관된 이벤트 플래그는 다른 task가 non-blocking wait call을 사용하여 모니터링 된다.

task는 또한 스위치가 눌렸거나, 모션 센서가 물체를 감지했거나, 폭발이 발생한 것과 같은 일시적인 이벤트를 보고할 수도 있다. 이러한 이벤트들에 응답하는 task는 일반적으로 그러한 이벤트들 중 임의의 것이 발생하기를 block-waiting하고, 이벤트를 처리할 것이다.

![Figure 14-10 Event Flags used for Status and Transient Events](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/f0329aac-80a8-427d-9272-f66691228828)

## Event Flags Internals
애플리케이션 프로그래머는 무제한의 이벤트 플래그 그룹을 만들 수 있다(사용가능한 RAM에만 제한됨). μC/OS-III의 이벤트 플래그 서비스는 OSFlag로 시작하며, 애플리케이션 프로그래머가 사용할 수 있는 서비스는 443페이지의 Appendix A "μC/OS-III API Reference"에 설명되어 있다. 이벤트 플래그 서비스는 os_cfg.h에서 상수 OS_CFG_FLAG_EN을 1로 설정함으로써 활성화된다.

이벤트 플래그 그룹은 OS_FLAG_GRP 자료형으로 정의된 커널 객체이며, 이는 다시 L14-10과 같은 구조체 os_flag_grp(os.h 참조)로 정의된다.

이벤트 플래그를 관리하기 위해 μC/OS-III가 제공되는 서비스들은 os_flag.c 파일에 구현되어 있다.

```c
typedef struct os_flag_grp OS_FLAG_GRP; (1)

struct os_flag_grp {
    OS_OBJ_TYPE Type; (2)
    CPU_CHAR *NamePtr; (3)
    OS_PEND_LIST PendList; (4)
    OS_FLAGS Flags; (5)
    CPU_TS TS; (6)
};
//L14-10 OS_FLAG_GRP data type
```

### L14-10(1)
μC/OS-III에서는 모든 구조체에 자료형이 부여된다. 모든 자료형은 "OS_"로 시작하여 대문자이다. 이벤트 플래그 그룹을 선언하려면 단순히 이벤트 플래그 그룹을 선언하는데 사용되는 변수의 자료형으로 OS_FLAG_GRP를 사용하면 된다.

### L14-10(2)
구조체는 "Type" 필드로 시작하며, 이를 통해  μC/OS-III가 이벤트 플래그 그룹으로 인식할 수 있다. 즉, 다른 커널 객체들도 구조의 첫 번째 멤버로서 "Type"을 가질 것이다. 만약 함수가 커널 객체를 전달받으면, μC/OS-III는 적절한 자료형( os_cfg.h에서 OS_CFG_OBJ_TYPE_CHK_EN을 1로 설정한다고 가정)이 전달되고 있음을 확인할 수 있을 것이다. 예를 들어, 메시지 큐(OS_Q)를 이벤트 플래그 서비스(예: OSFlagPend())로 전달하면, μC/OS-III는 유효하지 않은 객체가 전달되었음을 인식하고, 그에 따른 에러 코드를 반환할 수 있을 것이다.

### L14-10(3)
각 커널 객체는 디버거나 μC/Probe에서 쉽게 알아볼 수 있도록 이름을 부여받을 수 있다. 이 멤버는 단순히 NUL문자로 끝나는 ASCII 문자열을 가리키는 포인터이다.

### L14-10(4)
이벤트 플래그 그룹에 다수의 task가 대기할 수 있기 때문에, 이벤트 플래그 그룹은 197페이지의 10장 "Pend Lists(or Wait Lists)"에 기술된 바와 같은 pend list를 포함한다.

### L14-10(5)
이벤트 플래그 그룹은 일련의 플래그들(즉, 비트들)을 포함하고, 이 멤버는 이러한 플래그들의 현재 상태를 가지고 있다. 자료형 OS_FLAGS가 os_type.h에서 어떻게 선언되었는지에 따라 8,16,32비트로 구현되어있을 수 있다.

### L14-10(6)
이벤트 플래그 그룹은 이벤트 플래그 그룹이 마지막으로 post된 시점을 나타내기 위해 사용되는 타임스탬프를 가지고 있다. μC/OS-III는 사용자가 시간 측정을 할 수 있게 해주는 free-running 카운터가 있다고 가정한다. 이벤트 플래그 그룹이 post된 시점에 free-running 카운터가 읽혀지고 값이 이 필드에 놓이게 되는데, 이 값은 OSFlagPend()가 호출되면 반환된다. 이 값을 통해 애플리케이션은 언제 post가 수행되었는지, 또는 post후 (pend를 호출한 task가)CPU 제어권을 획득하는데 걸린 시간을 확인할 수 있다. 즉, OS_TS_GET()을 호출하여 현재 타임스탬프를 확인하고, 차이를 계산할 수 있다.

## Event Flags Internals(2)
OS_FLAG_GRP 자료형의 내부를 이해한다고 해도, 애플리케이션 코드는 이 구조체의 어떤 필드에도 직접 접근해서는 안 된다. 대신 항상 μC/OS-III와 함께 제공되는 API를 사용해야 한다.

이벤트 플래그 그룹은 L14-11과 같이 애플리케이션 프로그램에 의해 사용되기 전에 먼저 생성되어야 한다.

```c
OS_FLAG_GRP MyEventFlagGrp; (1)

void MyCode (void)
{
    OS_ERR err;
    :
    OSFlagCreate(&MyEventFlagGrp, (2)
                "My Event Flag Group", (3)
                (OS_FLAGS)0, (4)
                &err); (5)
    /* Check ’err” */
    :
}
```
### L14-11(1)
애플리케이션 프로그램은 OS_FLAG_GRP 타입의 변수를 선언해야한다. 이 변수는 이벤트 플래그 서비스가 참조할 것이다.

### L14-11(2)
OSFlagCreate()를 호출하여 이벤트 플래그 그룹을 생성하고 [L14-11](#l14-111)에서 할당된 이벤트 플래그 그룹에 주소를 전달한다.

### L14-11(3)
이벤트 플래그 그룹에 ASCII 이름을 지정할 수 있는데, 이 이름은 디버거나 μC/Probe에서 사용하여 이벤트 플래그 그룹을 쉽게 식별할 수 있다. μC/OS-III는 이름에 대한 포인터를 저장하므로 그 크기에 실질적인 제한이 없지만 ASCII 문자열은 NUL문자로 끝나야 한다.

### L14-11(4)
task와 ISR이 set이 아닌 clear된 bit로 이벤트를 시그널링하지 않는 한, 이벤트 플래그 그룹 내의 플래그를 모두 0으로 초기화한다. clear bit시 이벤트를 시그널링할 경우 모든 bit를 1로 초기화해야한다.

### L14-11(5)
OSFlagCreate()는 호출 결과에 따라 에러 코드를 반환한다. 만약 모든 argument가 유효하면, err는 OS_ERR_NONE이다.

## Event Flags Internals(3)
task는 L14-12와 같이 OSFlagPend()를 호출하여 ISR 또는 다른 task에서 하나 이상의 이벤트 플래그 비트를 대기한다(argument에 대한 자세한 설명은 443페이지의 Appendix A, "μC/OS-III API Reference" 참조).

```c
OS_FLAG_GRP MyEventFlagGrp;

void MyTask (void *p_arg)
{
    OS_ERR err;
    CPU_TS ts;
    OS_FLAGS which_flags;
    :
    while (DEF_ON) {
        :
        which_flags = OSFlagPend(&MyEventFlagGrp, /* (1) Pointer to event flag group */
                                    (OS_FLAGS)0x0F, /* Which bits to wait on */
                                    10, /* Maximum time to wait */
                                    OS_OPT_PEND_BLOCKING +
                                    OS_OPT_PEND_FLAG_SET_ANY, /* Option(s) */
                                    &ts, /* Timestamp of when posted to */
                                    &err); /* Pointer to Error returned */
        /* Check “err” (2) */
        :
        :
    }
}
//L14-12 Pending (or waiting) on an Event Flag Group
```

### L14-12(1)
OSFlagPend()를 호출하면, OSFlagPend()는 이 함수에 전달된 argument가 유효한 값을 가지는지 확인하는 것으로 시작한다(os_cfg.h에서 OS_CFG_ARG_CHK_EN이 1로 설정되어 있다고 가정). task가 대기 중인 bit가 set되어 있으면(또는 옵션에 따라 clear되어 있으면) OSFlagPend()가 반환되고 조건을 충족한 플래그를 보여준다. 이는 OSFlagPend()를 호출한 task가 기대한 결과이다.

이벤트 플래그 그룹에 찾고 있는 플래그가 포함되어 있지 않은 경우 OSFlagPend()를 호출한 task는 원하는 플래그가 set(또는 clear)될 때까지 기다려야 할 수도 있다. 옵션으로 OS_OPT_PEND_NON_BLOCKING을 지정하면(작업은 플래그를 대기하기 위해 중단되지 않음), OSFlagPend()는 즉시 리턴되며 반환된 오류 코드는 비트가 set(또는 clear)되지 않았음을 나타낸다.

옵션으로 OS_OPT_PEND_BLOCKING을 지정하면 이벤트 플래그 비트를 기다리는 task 목록에 삽입된다. task는 우선순위 순서로 삽입되지 않고 단순히 목록의 처음에 삽입된다. 비트가 set(또는 reset)될 때마다 이 목록의 모든 task를 검사하여 원하는 비트가 충족되었는지 확인해야 하기 때문이다.

0이 아닌 타임아웃을 지정하면 task는 tick list에도 삽입된다. 타임아웃을 0으로 지정하면 원하는 비트가 set(또는 clear)될 때까지 영원히 기다릴 것을 의미한다.

현재 task를 실행할 수 없으므로 스케줄러를 호출한다. 스케줄러는 실행준비된 다음으로 높은 우선순위의 task를 실행할 것이다.

이벤트 플래그 그룹이 post되고 OSFlagPend()를 호출한 task가 원하는 비트가 set(또는 clear)되면 task 상태를 검사하여 OSFlagPend()가 리턴되는 이유를 확인한다. 여러 이유는 다음과 같다:

1. 원하는 비트가 set(또는 clear)되었다.
2. 대기가 다른 task에 의해 중단되었다.
3. 타임아웃 값 내로 비트가 set(또는 clear)되지 않았다.
4. 이벤트 플래그 그룹이 삭제되었다.

OSFlagPend()가 리턴되면, 적절한 에러 코드를 통해 상기 결과를 알린다.

### L14-12(2)
OSFlagPend() err이 OS_ERR_NONE으로 설정된 상태로 리턴되면, 원하는 비트가 set(또는 clear)되었다고 생각하고 해당 이벤트를 생성한 ISR 또는 task를 서비스하는 작업을 진행할 수 있다. err에 다른 것이 포함된 경우(timeout되었거나, 대기가 다른 task에 의해 중단되었거나, 이벤트 플래그 그룹이 다른 task에 의해 삭제되었거나)가 있을 수 있으므로, 반환된 오류 코드를 검사하고 모든 것이 계획대로 되었다고 생각하지 않는 것이 항상 중요하다.

## Event Flags Internals(4)
ISR 또는 task에서 이벤트 플래그를 set하거나 clear하러면 L14-13과 같이 OSFlagPost()를 호출하면 된다.

```c
OS_FLAG_GRP MyEventFlagGrp;

void MyISR (void)
{
    OS_ERR err;
    OS_FLAGS flags_cur;
    :
    flags_cur = OSFlagPost(&MyEventFlagGrp, (1)
                            (OS_FLAGS)0x0C, (2)
                            OS_OPT_POST_FLAG_SET, (3)
                            &err); (4)
    /* Check ’err” */
    :
    :
}
//L14-13 Posting flags to an Event Flag Group
```

### L14-13(1)
task는 OSFlagPost()를 호출하여 이벤트 플래그 그룹을 post한다. 이벤트 플래그 그룹의 주소를 전달하여 원하는 이벤트 플래그 그룹을 지정한다. 물론 이벤트 플래그 그룹은 이전에 생성되어 있어야한다. OSFlagPost()는 post가 수행된 후 이벤트 플래그 그룹의 현재 이벤트 플래그 값을 반환한다.

### L14-13(2)
다음 argument는 ISR 또는 task가 이벤트 플래그 그룹에서 set(또는 clear)할 비트를 지정한다.

### L14-13(3)
OS_OPT_POST_FLAG_SET이나 OS_OPT_POST_FLAG_CLR를 지정할 수 있다.

OS_OPT_POST_FLAG_SET로 지정하면 두 번째 argument에 있는 비트 중 1의 위치에 있는 이벤트 플래그 그룹의 비트를 set한다. 예를 들어 MyEventFlagGrp.Flags가 0x03이면 L14-13의 코드가 MyEventFlagGrp.Flags를 0x0F로 변경한다.

OS_OPT_POST_FLAG_CLR로 지정하면 두 번째 argument에 있는 비트 중 1의 위치에 있는 이벤트 플래그 그룹의 비트를 clear 한다. 예를 들어 MyEventFlagGrp.Flags가 0x0F이면 L14-13의 코드가 MyEventFlagGrp.Flags를 0x03으로 변경한다.

OSFlagPost()를 호출할 때 스케줄러를 호출하지 않는 옵션(즉, OS_OPT_POST_NO_SCHED)을 지정할 수 있다. 이는 post가 수행되지만 이벤트 플래그 그룹에 우선순위가 높은 task가 대기했더라도 스케줄러가 호출되지 않음을 의미한다. 이를 통해 다른 post를 수행하고 모든 post를 동시에 활성화시킬 수 있다.

### L14-13(4)
OSFlagPost()는 호출 결과에 따라 에러 코드를 반환한다. 호출이 성공적이였다면, err는 OS_ERR_NONE일 것이다. 만약 그렇지 않다면, 에러 코드는 에러의 이유를 나타낼 것이다(가능한 에러 코드 목록은 443페이지의 Appendix A, "μC/OS-III API Reference" 참조).

# Synchronizing Multiple Tasks
세마포어를 broadcasting 하여 여러 task를 동기화하는 것은 일반적으로 사용된다. 여러 task가 동시에 실행되기 시작하도록 하는 것이 중요할 수 있다. 분명 단일 프로세서는 한 번에 하나의 task만 실제로 실행할 것이다. 그러나 이들의 실행 시작은 동기화될 것이다. 이를 *multiple task rendez-vous* 라고 한다. 그러나 동기화된 task들 중 일부는 브로드캐스트가 수행될 때 세마포어에 대기하지 않을 수도 있다. 이 문제는 Fig 14-11과 같이 세마포어와 이벤트 플래그를 조합함으로써 해결할 수 있다. 이를 제대로 작동시키기 위해서는 왼쪽의 task가 세마포어에 대기하는 task보다 낮은 우선순위를 가져야 한다.

![Figure 14-11 Multiple Task Rendezvous](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/4797d043-ada6-4d5e-bd53-84739676611f)

## F14-11(1)
rendez-vous에서 동기화해야하는 task는 이벤트 플래그 비트를 set해야한다(그리고 OS_OPT_POST_NO_SCHED를 지정해야 한다).

## F14-11(2)
task는 세마포어가 시그널링될때까지 기다려야한다.

## F14-11(3)
브로드캐스트할 task는 각 task에 대응하는 이벤트 플래그의 모든 비트가 set될 때까지 기다려야 한다.

## F14-11(4)
모든 대기 중인 task가 준비되면, 대기 task를 동기화할 task는 세마포어를 broadcasting 한다.

# Summary
ISR 또는 task가 하나 이상의 task에 신호를 보낼 수 있도록 하기 위해 세 가지 방법이 제시된다: 세마포어, task 세마포어 및 이벤트 플래그.

세마포어와 task 세마포어는 둘 다 이들이 credit tracking을 수행하고 이벤트의 발생을 누적할 수 있게 하는 카운터를 포함한다. ISR이나 task가 단일 task를 시그널링할 필요가 있는 경우 외부 세마포어 객체를 선언할 필요 없이, task 세마포어를 사용하는 것이 합리적이다. task 세마포어는 세마포어보다 약간 빠르다.

이벤트 플래그는 task가 하나 이상의 이벤트의 발생과 동기화할 필요가 있을 때 사용된다. 그러나 카운터와는 반대로 단일 비트가 각각의 이벤트를 나타내기 때문에 이벤트 플래그는 credit tracking을 수행할 수 없다.
# Reference
 - uC/OS-III: The Real-Time Kernel For the STM32 ARM Cortex-M3, Jean J. Labrosse, Micrium, 2009

[책 링크](https://micrium.atlassian.net/wiki/spaces/osiiidoc/overview)