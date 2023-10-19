---
title: "uart communication and led toggle with μC/OS-III and round robin scheduling"
last_modified_at: 2023-10-16T20:59:12+09:00
categories:
    - microc-os-3-stm32
tags:
    - microc-os
    - embedded-system

toc: true
toc_label: "My Table of Contents"
author_profile: true

---
# 목적
같은 우선순위의 task를 여러개 만들고 이 task를 round robin scheduling으로 각각 실행시킨다.

# UART 통신

## Serial Communication (직렬 통신)
하나의 데이터 선을 이용해 비트를 보내는 방법이다.

## UART
RX와 TX가 교차 연결되며(A와 B가 통신할 때, A의 Rx 핀은 B의 Tx 핀에 연결해야하며, A의 Tx핀은 B의 Rx 핀에 연결해야한다.), 비동기 통신 프로토골이므로, Baud rate를 일치시켜야한다.
start bit, data bit, parity bit, stop bit로 구성된다.

원래는 RCC로 Port D에 clock 인가하여 usart2의 핀인 PD5(TX), PD6(RX)를 활성화하고 baud rate, word length, stop bit를 설정해야하지만 이는 BSP_Ser_Init()함수에 구현되어 있기 때문에 생략한다. 데이터를 보낼때는 USART_SendData(USART2, 데이터)로 보낸다.

# 코드 작성

## 매크로 작성

### 우선순위 매크로 작성
```c
#define  APP_TASK_START_PRIO                        2
#define  APP_TASK_FIRST_PRIO                        2
#define  APP_TASK_SECOND_PRIO                       2
#define  APP_TASK_THIRD_PRIO                        2
//app_cfg.h
```
round robin scheduling을 사용해야하므로, 생성할 task의 우선순위를 2로 일치시킨다.

### stack 크기 매크로 작성
```c
#define  APP_TASK_START_STK_SIZE                    128
#define  APP_TASK_FIRST_STK_SIZE                    128
#define  APP_TASK_SECOND_STK_SIZE                    128
#define  APP_TASK_THIRD_STK_SIZE                    128
//app_cfg.h
```
생성할 task의 stack 크기를 128로 지정한다.

### round robin time quanta 설정

```c
#define TASK_FIRST_RR_TIME_QUANTA 1000
#define TASK_SECOND_RR_TIME_QUANTA 2000
#define TASK_THIRD_RR_TIME_QUANTA 500

```
LED를 토글할 task 3개의 time quanta를 다르게 설정한다.

### TASK_COUNT_PERIOD 정의

```c
#define TASK_COUNT_PERIOD 1000000
```
LED를 토글할 task에서 각 static 변수를 while문을 돌때마다 1 증가시키고, TASK_COUNT_PERIOD에 도달하면 LED를 토글한다.

## round robin 설정
```c
OSSchedRoundRobinCfg((CPU_BOOLEAN)DEF_TRUE, 
                         (OS_TICK    )10000,
                         (OS_ERR    *)&err);
```
Round Robin 스케줄링을 활성화하고, 시스템의 전체 Round Robin 시간 할당량을 설정한다.
여기서, ‘DEF_TRUE’는 Round Robin 스케줄링을 활성화한다. ‘10000’은 전체 시스템의 Round Robin time quanta를 의미한다.

## task 생성
```c
static  void  AppTaskCreate (void)
{
	OS_ERR  err;
	
	OSTaskCreate((OS_TCB     *)&AppTaskFirstTCB, 
							 (CPU_CHAR   *)"App First Start",
							 (OS_TASK_PTR )AppTaskFirst,
							 (void       *)0,
							 (OS_PRIO     )APP_TASK_FIRST_PRIO,
							 (CPU_STK    *)&AppTaskFirstStk[0],
							 (CPU_STK_SIZE)APP_TASK_FIRST_STK_SIZE / 10,
							 (CPU_STK_SIZE)APP_TASK_FIRST_STK_SIZE,
							 (OS_MSG_QTY  )0,
							 (OS_TICK     )TASK_FIRST_RR_TIME_QUANTA,
							 (void       *)0,
							 (OS_OPT      )(OS_OPT_TASK_STK_CHK | OS_OPT_TASK_STK_CLR),
							 (OS_ERR     *)&err);

    OSTaskCreate((OS_TCB     *)&AppTaskSecondTCB, 
							 (CPU_CHAR   *)"App Second Start",
							 (OS_TASK_PTR )AppTaskSecond,
							 (void       *)0,
							 (OS_PRIO     )APP_TASK_SECOND_PRIO,
							 (CPU_STK    *)&AppTaskSecondStk[0],
							 (CPU_STK_SIZE)APP_TASK_SECOND_STK_SIZE / 10,
							 (CPU_STK_SIZE)APP_TASK_SECOND_STK_SIZE,
							 (OS_MSG_QTY  )0,
							 (OS_TICK     )TASK_SECOND_RR_TIME_QUANTA,
							 (void       *)0,
							 (OS_OPT      )(OS_OPT_TASK_STK_CHK | OS_OPT_TASK_STK_CLR),
							 (OS_ERR     *)&err);

    OSTaskCreate((OS_TCB     *)&AppTaskThirdTCB, 
							 (CPU_CHAR   *)"App Third Start",
							 (OS_TASK_PTR )AppTaskThird,
							 (void       *)0,
							 (OS_PRIO     )APP_TASK_THIRD_PRIO,
							 (CPU_STK    *)&AppTaskThirdStk[0],
							 (CPU_STK_SIZE)APP_TASK_THIRD_STK_SIZE / 10,
							 (CPU_STK_SIZE)APP_TASK_THIRD_STK_SIZE,
							 (OS_MSG_QTY  )0,
							 (OS_TICK     )TASK_THIRD_RR_TIME_QUANTA,
							 (void       *)0,
							 (OS_OPT      )(OS_OPT_TASK_STK_CHK | OS_OPT_TASK_STK_CLR),
							 (OS_ERR     *)&err);
}

```
Task를 생성할 때 각 Task의 Round Robin 시간 할당량을 지정한다.
‘TASK_FIRST_RR_TIME_QUANTA’, ’TASK_SECOND_RR_TIME_QUANTA’, ’TASK_THIRD_RR_TIME_QUANTA’는 Time Quanta 정의에서 각각 2000,1000,500으로 정의된 각 task 별 Round Robin 시간 할당량이다.
또한 Round Robin 스케쥴링을 사용하기 위해 ‘우선순위 및 스택 크기 지정’에서 2로 동일하게 정의된 ‘APP_TASK_FIRST_PRIO’, ‘APP_TASK_SECOND_PRIO’, 
‘APP_TASK_THIRD_PRIO’를 사용하여 각 task에 같은 우선순위를 부여한다.

## 구현한 task 코드
```c
static void AppTaskFirst (void *p_arg) {
	OS_ERR      err;
	
	taskFirstCount = 0;

	while (DEF_TRUE) {
		taskFirstCount++;
		if (taskFirstCount % TASK_COUNT_PERIOD == 0) {
			BSP_LED_Toggle(1);
			USART_SendData(USART2, '*');
		}
	}
}

static void AppTaskSecond(void *p_arg){
    OS_ERR err;

    taskSecondCount=0;
    while (DEF_TRUE) {
		taskSecondCount++;
		if (taskSecondCount % TASK_COUNT_PERIOD == 0) {
			BSP_LED_Toggle(2);
			USART_SendData(USART2, '@');
		}
	}
}
static void AppTaskThird(void *p_arg){
    OS_ERR err;

    taskThirdCount=0;
    while (DEF_TRUE) {
		taskThirdCount++;
		if (taskThirdCount % TASK_COUNT_PERIOD == 0) {
			BSP_LED_Toggle(3);
			USART_SendData(USART2, '#');
		}
	}
}
```
AppTaskFirst는 ‘taskFirstCount’ 변수를 증가시키면서 실행된다.
‘taskFirstCount’ 변수의 값이 ‘TASK_COUNT_PERIOD’에 도달할 때마다 첫번째 LED를 토글하고 USART2를 통해 '*’ 문자를 전송한다.

AppTaskSecond는 ‘taskSecondCount’ 변수를 증가시키면서 실행된다.
‘taskSecondCount’ 변수의 값이 ‘TASK_COUNT_PERIOD’에 도달할 때마다 두번째 LED를 토글하고 USART2를 통해 '@’ 문자를 전송한다.

AppTaskThird는 ‘taskThirdCount’ 변수를 증가시키면서 실행된다.
‘taskThirdCount’ 변수의 값이 ‘TASK_COUNT_PERIOD’에 도달할 때마다 세번째 LED를 토글하고 USART2를 통해 '#’ 문자를 전송한다.

# 결과
![uart communication](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/399a8301-0b9e-49d6-ae33-07a39e72666b)

Round Robin을 이용한 Scheduling을 구현하였다. 따라서, AppTaskFirst, AppTaskSecond, AppTaskThird 순으로 규칙적으로 실행된다.
따라서, 시리얼 통신에서는 ‘*’, ‘@’, ‘#’ 문자가 주기적으로 나타나고 LED는 각 작업이 주기적으로 실행되면서 첫번째, 두번째, 세번째 LED 순으로 규칙적으로 번갈아 깜빡인다(Second, First, Third task 순으로 깜빡이는 횟수가 줄어든다).



