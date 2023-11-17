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
ISR은 인터럽트를 발생시킨 장치를 처리한 다음, OSSemPost()를 호출하여 세마포어를 signal한다. ISr이 완료되면 μC/OS-III를 호출한다(즉, OSIntExit()).

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
앞서 언급한 바와 같이 세마포어는 몇 번 signal(또는 post)되었는지를 기억한다. 즉, 이벤트를 대기하고 있는 task가 가장 우선순위가 높은 task가 되기 전에 ISR이 여러 번 발생하면, 세마포어는 자신이 signal된 횟수를 유지할 것이다. task가 가장 우선순위가 높은 실행준비 상태의 task가 되면, ISR이 발생한 횟수만큼 task가 blocking 되지 않고 실행될 것이다. 이를 credit tracking이라 하며, Fig 14-4에 예시가 있으며, 아래에서 설명한다.

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
세마포어는 카운터를 포함한다. 카운터는 os_type.h의 OS_SEm_CTR이 어떻게 선언되었는지에 따라 8비트, 16비트, 32비트 값으로 구현될 수 있다. μC/OS-III는 이 카운터로, 세마포어가 시그널링되는 횟수를 추적하며, 이 필드는 일반적으로 OSSemCreate()에 의해 0으로 초기화된다.

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
                “My Semaphore”, (3)
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
OSsemCreate()는 호출의 결과에 따라 오류 코드를 반환한다. 모든 인수가 유효하면 err는 OS_ERR_NONE 이다.

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
OSSemPend()가 호출되면, 이 함수에 전달된 argument가 유요한 값을 갖는지 확인하는 것으로 시작한다(os_cfg.h에서 OS_CFG_OBJ-TYPE_CHK_EN이 1로 설정되어있다고 가정).

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

에빈트가 발생했을 때 어떤 task를 시그널링해야하는지를 코드가 알고 있다면 이 기능을 사용할 수 있다. 예를 들어 이더넷 컨트롤러로부터 인터럽트를 받으면 수신된 패킷의 처리를 담당하는 task에 신호를 보낼 수 있는데 이 처리는 ISR 대신 task를 사용하여 수행하는 것이 바람직하기 때문이다.

![Figure 14-7 Semaphore built-into a Task](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/4bcd373a-bc23-4135-b5d4-9b665f127484)

표 14-2에 요약된, task 세마포어에 대해 수행할 다양한 함수가 있다.

| Function Name | Operation |
|---------------|-----------|
|OSTaskSemPend()|task 세마포어를 대기한다.           |
|OSTaskSemPendAbort()|task 세마포어 대기를 중단한다.           |
|OSTaskSemPost()|task를 시그널링한다.           |
|OSTaskSemSet()|세마포어 카운트를 원하는 값으로 설정한다.           |

(표 14-2 Task Semaphore API summary)
# Reference
 - uC/OS-III: The Real-Time Kernel For the STM32 ARM Cortex-M3, Jean J. Labrosse, Micrium, 2009

[책 링크](https://micrium.atlassian.net/wiki/spaces/osiiidoc/overview)