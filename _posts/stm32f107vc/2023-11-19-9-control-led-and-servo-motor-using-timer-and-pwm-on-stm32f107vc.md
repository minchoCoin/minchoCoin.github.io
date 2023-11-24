---
title: "control LED and Servo Motor using TImer and PWM on stm32f107vc"
last_modified_at: 2023-11-19T17:28:12+09:00
categories:
    - stm32f107vc
tags:
    - stm32f107vc
    - embedded-system

toc: true
toc_label: "My Table of Contents"
author_profile: true
classes: wide
---
# 목표
- 타이머의 이해
- 타이머의 종류의 이해
- 분주 계산 방법 이해
- PWM 이해

# 실험 원리

## 타이머
주기적 시간 처리에 사용되는 디지털 카운터 회로 모듈로, 펄스폭 계측, 주기적인 인터럽트 발생 등에 사용된다. 일반적으로 기본적으로 생성되는 신호의 주파수는 매우 높기 때문에, prescaler를 사용하여 주파수를 낮춘 후, 낮아진 주파수로 8,16비트 등의 카운터 회로를 사용하여 주기를 얻는다.

STM32에는 SysTick Timer, Watchdog timer, Advanced-control timer(TIM1,TIM8), General-purpose Timer(TIM2 to TIM5), Basic Timer(TIM6, TIM7)가 있다.

## 타이머의 종류

### SysTick Timer
Real-time operating system 전용이지만 standard down counter로 사용할 수 있
다. 24bit down counter(counter가 0에 도달하면 설정에 따라 인터럽트 발생)이고,
autoreload capability이다.

### Watchdog TImer
Watchdog는 CPU가 올바르게 작동하지 않을 시 강제로 리셋시키는 기능을 의미
한다. 소프트웨어 고장으로 인한 오작동을 감지하고 해결하는 역할을 한다.

#### Independent Watchdog (IWDG)
자체 전용 low-spped clock(LSI)에 의해 카운트되므로 메인 클록에 장애가 발
생하더라도 활성상태를 유지한다. 타이밍 정확도 제약이 낮은 애플리케이션에
적합하다. Free-running downcounter이며, 독립적인 RC oscillator에 의해 카운
트되며, watchdog가 활성화되어있고, downcounter 값이 0에 도달하면 리셋된
다.

#### Window Watchdog (WWDG)
7bit down counter이고, APB1의 클럭을 prescale하여 정의가능하다. 비정상적
인 애플리케이션 동작 감지를 위해 설정 가능한 time-window가 있고, time-window 내에서 반응하도록 요구하는 애플리케이션에 적합하다. 카운터가
0x40보다 작을 경우 또는 카운터가 time-window 밖에서 Reload 되었을 경우
리셋된다. 설정을 통해 카운터가 0x40과 같을 때 Early wakeup interrupt(EWI)
가 발생하게 설정 가능하다.

### Advanced-control timer (TIM1 and TIM8)
Prescaler(1 to 65536)를 이용해 설정가능한 16bit auto-reload counter(down, up, up/down)를 초함한다. 입력 신호 펄스 길이 측정(input capture) 또는 출력 파형 생성(output compare, PWM, complementary PWM with dead-time insertion) 등에 사용 가능하다.

Advanced-control timer와 general-purpose timer는 자원을 공유하지 않는 독립적인 구조이며, 동기화 시키는 것도 가능하다.

### Basic timer (TIM6 and TIM7)
16bit auto-reload upcounter이고, 16bit prescaler를 통해 주파수를 나누는 것(1 to 65536)도 가능하다. 이 타이머는 DAC와 연결되어 있고 타이머의 트리거 출력을 통해 DAC가 구동된다. 카운터 오버플로우 발생 시 인터럽트/DMA가 발생한다.

### General-purpose timer (TIM2 to TIM5)
Prescaler(1 to 65536)을 이용해 설정 가능한 16bit auto-reload counter(up, down, up/down)을 포함하고 있다. 입력 신호의 펄스 길이 측정(input capture) 또는 출력 파형 발생(output compare and PWM) 등 다양한 용도로 사용될 수 있다. 펄스 길이와 파형 주기는 timer prescaler와 RCC clock controller prescaler를 사용하여 몇 𝜇𝑠 에서 몇 𝑚𝑠 까지 변조할 수 있다. 완전히 독립적이며, 어떤 자원도 공유하지 않으나 동기화 가능하다.

![General-purpose timer block diagram](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/43e6898d-b7e7-4015-b1f1-22b96c783ab1)

(표 1: General-purpose timer block diagram)

![Counter timing diagram with prescaler division change from 1 to 4](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/b9cbfa44-1f12-4e60-afc7-2d6d2153e0b7)

(표 2: Counter timing diagram with prescaler division change from 1 to 4)

표1의 회로도에서, TIMx_ETR을 통해 클럭이 들어온다. 그리고 해당 클럭은 PSC로 이동하여 카운터에 들어가기 전, 원하는 값으로 조정된다.

Auto-reload register는 preload된다. 업데이트 이벤트는 카운터가 오버플로우(또는 downcount 시 언더플로우) 그리고 TIMx_CR1 레지스터의 UDIS 비트가 0이면 전송된다. 카운터는 prescaler의 출력 CK_CNT에 의해 카운트되며, 이는 TIMx_CR1 레지스터의 CEN 비트가 1일때만 활성화된다. Prescaler는 counter clock을 1에서 65536 사이의 임의의 인자로 나눌 수 있다.

Upcounting 모드에서, 카운터는 0부터 TIMx_ARR에 있는 값까지 카운트한 후, 0부터 다시 시작하여 counter overflow 이벤트를 생성한다. Update 이벤트는 각 counter overflow 이벤트 또는 TIMx_EGR 레지스터의 UG비트가 1이 되어있을 때 발생한다. Update event는 TIMx_CR1 레지스터의 UDIS 비트를 1로 하여 비활성화 될 수 있다. Update가 발생하면 TIMx_SR 레지스터의 UIF 비트가 1로 되고, prescaler 값은 TIMx_PSC 값으로 로드되고, auto-reload shadow register가 TIMx_ARR 값으로 업데이트된다.

Downcount 모드에서, 카운트는 TIMx_ARR의 값(auto-reload value)부터 0까지 카운트한 후, auto-reload value 부터 다시 시작하여 count underflow 이벤트를 생성한다.

분주는 MCU에서 제공하는 주파수를 원하는 값으로 바꾸는 것을 말하는 것으로, 원하는 주기 T는 다음과 같이 계산하여 생성할수 있다.

$$T=\frac{Prescaler \times Period}{TimerClockFrequency} $$

즉 prescaler 값에 period(몇 번 카운트할지)를 곱하고, 원래 클럭 주파수로 나누어 주면 된다.

## 라이브러리에서 설정된 timer clock frequency 확인하기
메인함수에서 호출한 SystemInit() 함수는 다시 SetSysClock()을 호출한다. 

```c
/**
  * @brief  Configures the System clock frequency, HCLK, PCLK2 and PCLK1 prescalers.
  * @param  None
  * @retval None
  */
static void SetSysClock(void)
{
#ifdef SYSCLK_FREQ_HSE
  SetSysClockToHSE();
#elif defined SYSCLK_FREQ_24MHz
  SetSysClockTo24();
#elif defined SYSCLK_FREQ_36MHz
  SetSysClockTo36();
#elif defined SYSCLK_FREQ_48MHz
  SetSysClockTo48();
#elif defined SYSCLK_FREQ_56MHz
  SetSysClockTo56();  
#elif defined SYSCLK_FREQ_72MHz
  SetSysClockTo72();
#endif
 
 /* If none of the define above is enabled, the HSI is used as System clock
    source (default after reset) */ 
}
//\Libraries\CMSIS\DeviceSupport\system_stm32f10x.c
```
현재 SYSCLK_FREQ_72MHz가 정의되어있기 때문에, SetSysClockTo72()가 호출되어 시스템클럭이 72MHz로 설정된다.

## PWM(Pulse Width Modulation) 과 서보모터
일정한 주기 내에서 Duty ratio를 변화시켜서 평균 전압을 제어하는 방법이다. 예를 들어 0~5V의 전력 범위에서 2.5V 전압을 가하고 싶다면 50% 듀티 사이클을 적용하면 된다. PWM을 서보모터 각도 조절에 이용할 경우, 20ms period에 1~2ms pulse를 입력하면 된다(1ms는 -90도, 1.5ms는 0도, 2ms는 90도이다).

# 세부 실험 내용
- TFT LCD에 팀 이름, LED 토글 ON/OFF 상태, LED ON/OFF 버튼을 생성한다.
- LCD의 버튼 터치 시 TIM2 interrupt, TIM3 PWM을 활용하여 LED 2개와 서보모터 제어
    - LED : 1초 마다 LED1 TOGGLE, 5초 마다 LED2 TOGGLE
    - LED : 1초 마다 LED1 TOGGLE, 5초 마다 LED2 TOGGLE
- LCD 버튼 한번더 터치 시 서보모터 : 1초 마다 반대쪽 방향으로 조금씩(100) 이동(LED는 OFF)

# 실험 방법
1. 서보모터를 PB0에 연결한다(datasheet에 의하면, PB0은 TIM3의 채널3이다).
2. RCC를 이용하여 사용할 핀과 타이머에 클럭을 인가한다.
3. GPIO을 이용하여 사용할 핀을 설정한다.
4. TIM2를 1초 간격으로 업데이트되도록 설정한다.
5. TIM3를 서보모터에 사용할 수 있도록 설정한다.
6. 서보모터에 들어가는 펄스 값을 바꾸는 함수를 작성한다.
7. NVIC를 이용하여 인터럽트 우선순위를 설정한다.
8. TIM2가 업데이트되었을 때(인터럽트가 발생하였을 때) LED가 토글되도록(LED1은 업데이트마다, LED2는 업데이트가 5번 발생하였을 때) 설정한다.
9. TFT LCD에 팀 이름과 LED 토글 ON/OFF 상태, LED ON/OFF 버튼을 생성하여 LCD에 해당 부분을 터치하면 LED가 토글되도록 한다.

# 코드 작성

## 여러 헤더파일 include
```c
#include <stdbool.h>
#include "stm32f10x.h"
#include "stm32f10x_exti.h"
#include "stm32f10x_gpio.h"
#include "stm32f10x_usart.h"
#include "stm32f10x_rcc.h"
#include "core_cm3.h"
#include "stm32f10x_adc.h"
#include "lcd.h"
#include "touch.h"
#include "misc.h"

#include "stm32f10x_tim.h"
```

## 전역변수 정의
```c
uint16_t value, x, y;
uint16_t led1=0, led2=0, count=0, toggle=0;
uint16_t curDeg = 700, minDeg= 700, maxDeg = 2600;
```

## RCC 설정
```c
void RCC_Configure(void) {
   RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOD,ENABLE);  // LED PD2,3     
   RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOB,ENABLE);  // PB0     

   RCC_APB1PeriphClockCmd(RCC_APB1Periph_TIM2,ENABLE);   // TIMER2, 3 Enable
   RCC_APB1PeriphClockCmd(RCC_APB1Periph_TIM3,ENABLE);
}
```
실험에 사용할 LED 핀(PD2, PD3)과 서보모터 핀(PB0), 그리고 TIM2, TIM3 타이머에 클럭을 인가한다.

## GPIO 설정
```c
void GPIO_Configure(void) {
   GPIO_InitTypeDef GPIO_InitStructure;

   GPIO_InitStructure.GPIO_Pin = GPIO_Pin_2 | GPIO_Pin_3; // LED PD2, 3
   GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;
   GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
   GPIO_Init(GPIOD, &GPIO_InitStructure);

   GPIO_InitStructure.GPIO_Pin = GPIO_Pin_0;             // PB0
   GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AF_PP;
   GPIO_Init(GPIOB, &GPIO_InitStructure);
}
```
LED 핀(PD2, PD3)은 General purpose output push-pull(max speed 50MHz)로 설정하고, 서보모터 핀(PB0)는 Alternative function output push-pull로 설정한다.

## TIM2 설정
```c
void TIM2_Init(void) {                                  
   TIM_TimeBaseInitTypeDef TIM2_InitStructure;

   TIM2_InitStructure.TIM_ClockDivision = TIM_CKD_DIV1;
   TIM2_InitStructure.TIM_CounterMode = TIM_CounterMode_Up;
   TIM2_InitStructure.TIM_Period = 10000;
   TIM2_InitStructure.TIM_Prescaler = 7200;

   TIM_TimeBaseInit(TIM2, &TIM2_InitStructure);
   TIM_ARRPreloadConfig(TIM2, ENABLE);
   TIM_Cmd(TIM2, ENABLE);
   TIM_ITConfig(TIM2, TIM_IT_Update, ENABLE);
}
```
Clock Division은 1, counter mode는 upcount mode, period는 10000, perscaler는 7200으로 설정하여 $T=\frac{10000 \times 7200}{72MHz} = 1(s)$ 로 설정한다.

그리고 TIM_TimeBaseInit()을 호출하여 설정한 구조체를 실제 TIM2 레지스터에 반영한다. 그리고 TIM_ARRPreloadConfig()를 통해 TIM2_CR1의 ARPE 비트를 1로 설정한다. 그리고 TIM_Cmd()를 통해 TIM2 peripherial을 활성화하고, TIM_ITConfig()를 통해 업데이트 이벤트가 발생했을 때 인터럽트가 발생하도록 한다.

![ARPE 값에 따른 업데이트 이벤트가 처리되는 방법(ARPE=0)](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/63ce7f7c-a6ca-4a98-b731-83f5244765d4)
![ARPE 값에 따른 업데이트 이벤트가 처리되는 방법(ARPE=1)](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/5253d12d-56cb-4dec-ab3b-c7f23eb3b3ef)

(그림 1: ARPE 값에 따른 업데이트 이벤트가 처리되는 방법)

Figure1과 같이, ARPE 값이 1일 경우 Auto-reload preload register 값(TIM2_ARR) 값을 변경하더라도 Auto-reload shadow register가 업데이트 이벤트가 발생한 후 바뀌기 때문에 TIM2_ARR을 바꾼 직후에 나타나는 업데이트 이벤트가 TIM2_ARR의 변경하기 전 값에서 발생하게 된다

## TIM3 설정
```c
void TIM3_Init() {
   TIM_TimeBaseInitTypeDef TIM3_InitStructure;
   TIM_OCInitTypeDef TIM_OCInitStructure;

   uint16_t prescale;
   uint16_t period = 20000;
   uint16_t frequency = 50;

   prescale = (uint16_t) (SystemCoreClock / period / frequency);

   TIM3_InitStructure.TIM_Period = period;         
   TIM3_InitStructure.TIM_Prescaler = prescale;
   TIM3_InitStructure.TIM_ClockDivision = TIM_CKD_DIV1;
   TIM3_InitStructure.TIM_CounterMode = TIM_CounterMode_Down;

   TIM_OCInitStructure.TIM_OCMode = TIM_OCMode_PWM1;
   TIM_OCInitStructure.TIM_OCPolarity = TIM_OCPolarity_High;
   TIM_OCInitStructure.TIM_OutputState = TIM_OutputState_Enable;
   TIM_OCInitStructure.TIM_Pulse = 0; 

   TIM_OC3Init(TIM3, &TIM_OCInitStructure);
   TIM_TimeBaseInit(TIM3, &TIM3_InitStructure);
   TIM_OC3PreloadConfig(TIM3, TIM_OCPreload_Disable);
   TIM_ARRPreloadConfig(TIM3, ENABLE);
   TIM_Cmd(TIM3, ENABLE);
}

```
TIM3의 period를 20000, frequency를 50Hz로 하기 위하여 prescale을 계산한다. SystemCoreClock은 system_stm32f10x.c에 정의되어있고, 72MHz로 정의되어있다. TIM3의 period는 20000, prescale은 계산한 값을 대입하고, ClockDivision은 TIM_CKD_DIV1(1로 나눔)으로 설정하고, downcounter모드로 설정한다.

그리고 TIM_OCInitTypeDef 구조체를 통해 PWM 채널 1을 사용하고, Polarity는 High 설정하고, output compare state를 활성화한다. 그리고 펄스는 0으로 초기화한다. TIM_OC3Init()과 TIM_TimeBaseInit()을 통해 설정한 구조체를 레지스터에 반영하고, TIM_OC3PreloadConfig()를 통해 TIM3의 채널3의 CCR 레지스터(펄스 폭 저장)는 영구적으로 로드되도록한다. 그리고 TIM_Cmd를 통해 TIM3의 peripherial을 활성화한다.

## 서보모터에 들어가는 펄스 폭을 바꾸는 함수 작성
```c
void delayMotor() {
   for (int i = 0; i < 100000; i ++) {}
}

void change(uint16_t per) {                  
   int pwm_pulse = per;

   TIM_OCInitTypeDef TIM_OCInitStructure;
   TIM_OCInitStructure.TIM_OCMode = TIM_OCMode_PWM1;
   TIM_OCInitStructure.TIM_OCPolarity = TIM_OCPolarity_High;
   TIM_OCInitStructure.TIM_OutputState = TIM_OutputState_Enable;
   TIM_OCInitStructure.TIM_Pulse = pwm_pulse; 

   TIM_OC3Init(TIM3, &TIM_OCInitStructure);
   delayMotor();
}

```
TIM_OCInitTypeDef 구조체와 TIM_OC3Init()함수를 이용하여 매개변수로 들어온 per로 서보모터에 들어가는 펄스를 바꾼다.

## NVIC 설정
```c
void NVIC_Configure(void) {
   NVIC_InitTypeDef NVIC_InitStructure;
   NVIC_PriorityGroupConfig(NVIC_PriorityGroup_1);

   NVIC_InitStructure.NVIC_IRQChannel = TIM2_IRQn;
   NVIC_InitStructure.NVIC_IRQChannelCmd = ENABLE;
   NVIC_InitStructure.NVIC_IRQChannelSubPriority = 0x00;
   NVIC_InitStructure.NVIC_IRQChannelPreemptionPriority = 0x00;
   NVIC_Init(&NVIC_InitStructure);

   NVIC_InitStructure.NVIC_IRQChannel = TIM3_IRQn;
   NVIC_InitStructure.NVIC_IRQChannelCmd = ENABLE;
   NVIC_InitStructure.NVIC_IRQChannelSubPriority = 0x00;
   NVIC_InitStructure.NVIC_IRQChannelPreemptionPriority = 0x00;
   NVIC_Init(&NVIC_InitStructure);
}
```
TIM2 인터럽트의 우선순위를 0으로 지정한다.

## TIM2 인터럽트 핸들러 작성
```c
void TIM2_IRQHandler() {
   if(TIM_GetITStatus(TIM2,TIM_IT_Update) != RESET) {
      change((uint16_t)curDeg);
      if(toggle) {                                 // BTN ON      
         curDeg += 100;
         if (curDeg > maxDeg)
            curDeg = minDeg;
         count++;                                  // count + 1    
         printf("%d\n", count);
         if(led1 == 0) {                           // LED 1
            GPIO_ResetBits(GPIOD,GPIO_Pin_2);
            led1 = 1;
         } 
         else {
            GPIO_SetBits(GPIOD,GPIO_Pin_2);
            led1 = 0;
         }
         if(count % 5 ==0) {                       // LED 2
            if(led2 == 0){ 
               GPIO_ResetBits(GPIOD,GPIO_Pin_3);
               led2 = 1; 
            } 
            else {
               GPIO_SetBits(GPIOD,GPIO_Pin_3);
               led2 = 0;
            }
            count = 0;
         }
      } 
      else {                                       // BTN OFF
         curDeg -= 100;
         if (curDeg < minDeg)
            curDeg = maxDeg;
         GPIO_SetBits(GPIOD,GPIO_Pin_2);
         GPIO_SetBits(GPIOD,GPIO_Pin_3);
      }
   TIM_ClearITPendingBit(TIM2, TIM_IT_Update);
   }
}
```
TIM2에서 인터럽트가 발생하면, 그 인터럽트가 업데이트로 인해 발생한 인터럽트인지 확인하고, 맞다면 curDeg값으로 서보모터에 들어가는 펄스 값을 바꾸고 버튼이 ON되어있을 경우 서보모터에 다음에 들어갈 펄스 값을 100증가시키고, led1을 토글한다. Led2는 count 변수를 이용해 count가 5가될 때 토글하여 led1은 1초마다, led2는 5초마다 토글되도록 한다.

버튼이 OFF되어 있을 경우 서보모터에 들어가는 펄스 값을 100 감소시키고, led를 끈다.

토글은 전역 변수를 이용하여, 1이면 0으로 바꾸고, GPIO_SetBits()를 이용하여 LED를 끄고, 0이면 1로 바꾸고 GPIO_ResetBits()를 이용하여 LED를 켠다.
서보모터의 펄스값이 최대값을 넘어가면 최소값으로 바꾸고, 최소값보다 작아지면 최대값으로 바꾼다.

## main문 작성
```c
int main(void) {

   SystemInit();
   RCC_Configure();
   GPIO_Configure();
   TIM2_Init();
   TIM3_Init();
   NVIC_Configure();
   
   LCD_Init();
   Touch_Configuration();
   Touch_Adjust();
   LCD_Clear(WHITE);

   LCD_ShowString(80,80,"THUR_Team05", BLACK, WHITE);     
   LCD_ShowString(90,200,"BTN",BLACK,WHITE); 
   LCD_ShowString(80,100,"OFF",BLACK,WHITE);       
   LCD_DrawRectangle(60, 180, 150, 220);

   while (1) {
      Touch_GetXY(&x,&y,1);    
      Convert_Pos(x,y,&x,&y);
      if(60<=x && x<=180 && 150<=y && y<=220) { 
         if (toggle==0) {                            //on    
            LCD_ShowString(80,100,"    ",BLACK,WHITE);
            toggle=1;
            LCD_ShowString(80,100,"ON",BLACK,WHITE);
         } 
         else {                                      //off    
            LCD_ShowString(80,100,"    ",BLACK,WHITE);
            toggle=0;
            LCD_ShowString(80,100,"OFF",BLACK,WHITE);   
         }     
      }
   }
   return 0;
}
```
LCD_ShowString을 이용해 팀이름과 버튼, 버튼 상태를 출력하고, LCD_DrawRectangle()을 통해 버튼 테두리를 그려준다. 그리고 무한반복문을 돌며 터치가 된 곳의 좌표를 얻고, 터치된 곳의 좌표가 버튼이 있는 위치이면, 버튼이 눌렸었을 때는 눌리지 않은 것으로, 눌리지 않았었을 때는 눌리지 않은 것으로 바꾸고(toggle 전역 변수 이용,) LCD에 ON표시 또는 OFF 표시를 한다.

# 실험 결과
{% include video id="gg2aGDYL3nQ" provider="youtube" %}

LCD의 버튼이 ON되어있을 때, LED1이 1초마다 토글되고, LED2는 5초마다 토글된다. 서보모터는 반시계방향으로 조금씩 움직인다.

LCD의 버튼이 OFF되어있을 때, 서보모터가 시계방향으로 조금씩 움직인다. LED는 OFF되어있다.

# 알게 된 점
타이머를 이용하여 일정 간격으로 어떤 작업을 하는 방법을 알게 되었으며, 타이머와 PWM을 이용해 서보모터를 제어하는 방법을 알게 되었다.

# 참고 문헌
- STM32F107 Datasheet https://www.st.com/resource/en/datasheet/stm32f107vc.pdf 
- STM32F107 Reference Manual https://www.st.com/resource/en/reference_manual/rm0008- stm32f101xxstm32f102xxstm32f103xx-stm32f105xx-and-stm32f107xxadvancedarmbased 32bitmcusstmicroelectronics.pdf 
- STM32F107VCT6 schematic
- SG90 servo motor datasheet
http://www.ee.ic.ac.uk/pcheung/teaching/DE1_EE/stores/sg90_datasheet.pdf

# 참고 - Timer 관련 라이브러리 구조체 및 함수

## TIM_TimeBaseInitTypeDef

```c
/** 
  * @brief  TIM Time Base Init structure definition
  * @note   This structure is used with all TIMx except for TIM6 and TIM7.    
  */

typedef struct
{
  uint16_t TIM_Prescaler;         /*!< Specifies the prescaler value used to divide the TIM clock.
                                       This parameter can be a number between 0x0000 and 0xFFFF */

  uint16_t TIM_CounterMode;       /*!< Specifies the counter mode.
                                       This parameter can be a value of @ref TIM_Counter_Mode */

  uint16_t TIM_Period;            /*!< Specifies the period value to be loaded into the active
                                       Auto-Reload Register at the next update event.
                                       This parameter must be a number between 0x0000 and 0xFFFF.  */ 

  uint16_t TIM_ClockDivision;     /*!< Specifies the clock division.
                                      This parameter can be a value of @ref TIM_Clock_Division_CKD */

  uint8_t TIM_RepetitionCounter;  /*!< Specifies the repetition counter value. Each time the RCR downcounter
                                       reaches zero, an update event is generated and counting restarts
                                       from the RCR value (N).
                                       This means in PWM mode that (N+1) corresponds to:
                                          - the number of PWM periods in edge-aligned mode
                                          - the number of half PWM period in center-aligned mode
                                       This parameter must be a number between 0x00 and 0xFF. 
                                       @note This parameter is valid only for TIM1 and TIM8. */
} TIM_TimeBaseInitTypeDef;

/** @defgroup TIM_Counter_Mode 
  * @{
  */

#define TIM_CounterMode_Up                 ((uint16_t)0x0000)
#define TIM_CounterMode_Down               ((uint16_t)0x0010)
#define TIM_CounterMode_CenterAligned1     ((uint16_t)0x0020)
#define TIM_CounterMode_CenterAligned2     ((uint16_t)0x0040)
#define TIM_CounterMode_CenterAligned3     ((uint16_t)0x0060)
#define IS_TIM_COUNTER_MODE(MODE) (((MODE) == TIM_CounterMode_Up) ||  \
                                   ((MODE) == TIM_CounterMode_Down) || \
                                   ((MODE) == TIM_CounterMode_CenterAligned1) || \
                                   ((MODE) == TIM_CounterMode_CenterAligned2) || \
                                   ((MODE) == TIM_CounterMode_CenterAligned3))
/**
  * @}
  */ 

/** @defgroup TIM_Clock_Division_CKD 
  * @{
  */

#define TIM_CKD_DIV1                       ((uint16_t)0x0000)
#define TIM_CKD_DIV2                       ((uint16_t)0x0100)
#define TIM_CKD_DIV4                       ((uint16_t)0x0200)
#define IS_TIM_CKD_DIV(DIV) (((DIV) == TIM_CKD_DIV1) || \
                             ((DIV) == TIM_CKD_DIV2) || \
                             ((DIV) == TIM_CKD_DIV4))
/**
  * @}
  */

```

## TIM_TimeBaseInit()

```c
/**
  * @brief  Initializes the TIMx Time Base Unit peripheral according to 
  *         the specified parameters in the TIM_TimeBaseInitStruct.
  * @param  TIMx: where x can be 1 to 17 to select the TIM peripheral.
  * @param  TIM_TimeBaseInitStruct: pointer to a TIM_TimeBaseInitTypeDef
  *         structure that contains the configuration information for the 
  *         specified TIM peripheral.
  * @retval None
  */
void TIM_TimeBaseInit(TIM_TypeDef* TIMx, TIM_TimeBaseInitTypeDef* TIM_TimeBaseInitStruct)
```

## TIM_ARRPreloadConfig()

```c
/**
  * @brief  Enables or disables TIMx peripheral Preload register on ARR.
  * @param  TIMx: where x can be  1 to 17 to select the TIM peripheral.
  * @param  NewState: new state of the TIMx peripheral Preload register
  *   This parameter can be: ENABLE or DISABLE.
  * @retval None
  */
void TIM_ARRPreloadConfig(TIM_TypeDef* TIMx, FunctionalState NewState)
```

## TIM_Cmd()
```c
/**
  * @brief  Enables or disables the specified TIM peripheral.
  * @param  TIMx: where x can be 1 to 17 to select the TIMx peripheral.
  * @param  NewState: new state of the TIMx peripheral.
  *   This parameter can be: ENABLE or DISABLE.
  * @retval None
  */
void TIM_Cmd(TIM_TypeDef* TIMx, FunctionalState NewState)
```

## TIM_ITConfig()

```c
/**
  * @brief  Enables or disables the specified TIM interrupts.
  * @param  TIMx: where x can be 1 to 17 to select the TIMx peripheral.
  * @param  TIM_IT: specifies the TIM interrupts sources to be enabled or disabled.
  *   This parameter can be any combination of the following values:
  *     @arg TIM_IT_Update: TIM update Interrupt source
  *     @arg TIM_IT_CC1: TIM Capture Compare 1 Interrupt source
  *     @arg TIM_IT_CC2: TIM Capture Compare 2 Interrupt source
  *     @arg TIM_IT_CC3: TIM Capture Compare 3 Interrupt source
  *     @arg TIM_IT_CC4: TIM Capture Compare 4 Interrupt source
  *     @arg TIM_IT_COM: TIM Commutation Interrupt source
  *     @arg TIM_IT_Trigger: TIM Trigger Interrupt source
  *     @arg TIM_IT_Break: TIM Break Interrupt source
  * @note 
  *   - TIM6 and TIM7 can only generate an update interrupt.
  *   - TIM9, TIM12 and TIM15 can have only TIM_IT_Update, TIM_IT_CC1,
  *      TIM_IT_CC2 or TIM_IT_Trigger. 
  *   - TIM10, TIM11, TIM13, TIM14, TIM16 and TIM17 can have TIM_IT_Update or TIM_IT_CC1.   
  *   - TIM_IT_Break is used only with TIM1, TIM8 and TIM15. 
  *   - TIM_IT_COM is used only with TIM1, TIM8, TIM15, TIM16 and TIM17.    
  * @param  NewState: new state of the TIM interrupts.
  *   This parameter can be: ENABLE or DISABLE.
  * @retval None
  */
void TIM_ITConfig(TIM_TypeDef* TIMx, uint16_t TIM_IT, FunctionalState NewState)
```

## TIM_OCInitTypeDef
```c
typedef struct
{
  uint16_t TIM_OCMode;        /*!< Specifies the TIM mode.
                                   This parameter can be a value of @ref TIM_Output_Compare_and_PWM_modes */

  uint16_t TIM_OutputState;   /*!< Specifies the TIM Output Compare state.
                                   This parameter can be a value of @ref TIM_Output_Compare_state */

  uint16_t TIM_OutputNState;  /*!< Specifies the TIM complementary Output Compare state.
                                   This parameter can be a value of @ref TIM_Output_Compare_N_state
                                   @note This parameter is valid only for TIM1 and TIM8. */

  uint16_t TIM_Pulse;         /*!< Specifies the pulse value to be loaded into the Capture Compare Register. 
                                   This parameter can be a number between 0x0000 and 0xFFFF */

  uint16_t TIM_OCPolarity;    /*!< Specifies the output polarity.
                                   This parameter can be a value of @ref TIM_Output_Compare_Polarity */

  uint16_t TIM_OCNPolarity;   /*!< Specifies the complementary output polarity.
                                   This parameter can be a value of @ref TIM_Output_Compare_N_Polarity
                                   @note This parameter is valid only for TIM1 and TIM8. */

  uint16_t TIM_OCIdleState;   /*!< Specifies the TIM Output Compare pin state during Idle state.
                                   This parameter can be a value of @ref TIM_Output_Compare_Idle_State
                                   @note This parameter is valid only for TIM1 and TIM8. */

  uint16_t TIM_OCNIdleState;  /*!< Specifies the TIM Output Compare pin state during Idle state.
                                   This parameter can be a value of @ref TIM_Output_Compare_N_Idle_State
                                   @note This parameter is valid only for TIM1 and TIM8. */
} TIM_OCInitTypeDef;



/** @defgroup TIM_Output_Compare_and_PWM_modes 
  * @{
  */

#define TIM_OCMode_Timing                  ((uint16_t)0x0000)
#define TIM_OCMode_Active                  ((uint16_t)0x0010)
#define TIM_OCMode_Inactive                ((uint16_t)0x0020)
#define TIM_OCMode_Toggle                  ((uint16_t)0x0030)
#define TIM_OCMode_PWM1                    ((uint16_t)0x0060)
#define TIM_OCMode_PWM2                    ((uint16_t)0x0070)
#define IS_TIM_OC_MODE(MODE) (((MODE) == TIM_OCMode_Timing) || \
                              ((MODE) == TIM_OCMode_Active) || \
                              ((MODE) == TIM_OCMode_Inactive) || \
                              ((MODE) == TIM_OCMode_Toggle)|| \
                              ((MODE) == TIM_OCMode_PWM1) || \
                              ((MODE) == TIM_OCMode_PWM2))
#define IS_TIM_OCM(MODE) (((MODE) == TIM_OCMode_Timing) || \
                          ((MODE) == TIM_OCMode_Active) || \
                          ((MODE) == TIM_OCMode_Inactive) || \
                          ((MODE) == TIM_OCMode_Toggle)|| \
                          ((MODE) == TIM_OCMode_PWM1) || \
                          ((MODE) == TIM_OCMode_PWM2) ||	\
                          ((MODE) == TIM_ForcedAction_Active) || \
                          ((MODE) == TIM_ForcedAction_InActive))
/**
  * @}
  */

  /** @defgroup TIM_Output_Compare_state 
  * @{
  */

#define TIM_OutputState_Disable            ((uint16_t)0x0000)
#define TIM_OutputState_Enable             ((uint16_t)0x0001)
#define IS_TIM_OUTPUT_STATE(STATE) (((STATE) == TIM_OutputState_Disable) || \
                                    ((STATE) == TIM_OutputState_Enable))
/**
  * @}
  */ 

  /** @defgroup TIM_Output_Compare_N_state 
  * @{
  */

#define TIM_OutputNState_Disable           ((uint16_t)0x0000)
#define TIM_OutputNState_Enable            ((uint16_t)0x0004)
#define IS_TIM_OUTPUTN_STATE(STATE) (((STATE) == TIM_OutputNState_Disable) || \
                                     ((STATE) == TIM_OutputNState_Enable))
/**
  * @}
  */ 
/** @defgroup TIM_Output_Compare_Polarity 
  * @{
  */

#define TIM_OCPolarity_High                ((uint16_t)0x0000)
#define TIM_OCPolarity_Low                 ((uint16_t)0x0002)
#define IS_TIM_OC_POLARITY(POLARITY) (((POLARITY) == TIM_OCPolarity_High) || \
                                      ((POLARITY) == TIM_OCPolarity_Low))
/**
  * @}
  */

  /** @defgroup TIM_Output_Compare_N_Polarity 
  * @{
  */
  
#define TIM_OCNPolarity_High               ((uint16_t)0x0000)
#define TIM_OCNPolarity_Low                ((uint16_t)0x0008)
#define IS_TIM_OCN_POLARITY(POLARITY) (((POLARITY) == TIM_OCNPolarity_High) || \
                                       ((POLARITY) == TIM_OCNPolarity_Low))
/**
  * @}
  */

  /** @defgroup TIM_Output_Compare_Idle_State 
  * @{
  */

#define TIM_OCIdleState_Set                ((uint16_t)0x0100)
#define TIM_OCIdleState_Reset              ((uint16_t)0x0000)
#define IS_TIM_OCIDLE_STATE(STATE) (((STATE) == TIM_OCIdleState_Set) || \
                                    ((STATE) == TIM_OCIdleState_Reset))
/**
  * @}
  */ 
  /** @defgroup TIM_Output_Compare_N_Idle_State 
  * @{
  */

#define TIM_OCNIdleState_Set               ((uint16_t)0x0200)
#define TIM_OCNIdleState_Reset             ((uint16_t)0x0000)
#define IS_TIM_OCNIDLE_STATE(STATE) (((STATE) == TIM_OCNIdleState_Set) || \
                                     ((STATE) == TIM_OCNIdleState_Reset))
/**
  * @}
  */ 
```

## TIM_OC3Init()
```c
/**
  * @brief  Initializes the TIMx Channel3 according to the specified
  *         parameters in the TIM_OCInitStruct.
  * @param  TIMx: where x can be  1, 2, 3, 4, 5 or 8 to select the TIM peripheral.
  * @param  TIM_OCInitStruct: pointer to a TIM_OCInitTypeDef structure
  *         that contains the configuration information for the specified TIM peripheral.
  * @retval None
  */
void TIM_OC3Init(TIM_TypeDef* TIMx, TIM_OCInitTypeDef* TIM_OCInitStruct)
```

## TIM_OC3PreloadConfig()
```c
/**
  * @brief  Enables or disables the TIMx peripheral Preload register on CCR3.
  * @param  TIMx: where x can be  1, 2, 3, 4, 5 or 8 to select the TIM peripheral.
  * @param  TIM_OCPreload: new state of the TIMx peripheral Preload register
  *   This parameter can be one of the following values:
  *     @arg TIM_OCPreload_Enable
  *     @arg TIM_OCPreload_Disable
  * @retval None
  */
void TIM_OC3PreloadConfig(TIM_TypeDef* TIMx, uint16_t TIM_OCPreload)
```
