---
title: "GPIO control and UART communication using interrupt on stm32f107vc"
last_modified_at: 2023-10-29T00:53:12+09:00
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
- Interrupt 방식을 활용한 GPIO 제어 및 UART 통신
- 라이브러리 함수 사용법 숙지

# 실험 원리

## 폴링과 인터럽트
폴링(Polling)은 CPU가 특정 이벤트를 처리하기 위해 이벤트가 발생할 때까지 장치나 프로그램의 상태를 주기적으로 확인하는 방식을 말한다. CPU의 사용률이 상대적으로 높고, 반응속도가 느리다. 주기적인 작업이 필요한 장치가 없을 경우 비효율적이다.

인터럽트(Interrupt)는 CPU가 특정 이벤트 발생시 현재 작업을 멈추고 해당 인터럽트 서비스 루틴을 수행 후 다시 이전 작업으로 돌아가는 방식이다. 시스템 부하가 적고 빠른 속도로 대응이 가능하다. 

## 하드웨어 인터럽트와 소프트웨어 인터럽트
하드웨어 인터럽트는 비동기식 이벤트 처리로 주변장치의 요청에 의해 발생하는 인터럽트이며 높은 우선순위를 갖는다. 하드디스크 읽기 요청, 디스크 읽기 끝남, 키보드 입력 등에서 발생한다.

소프트웨어 인터럽트는 동기식 이벤트 처리로 사용자가 프로그램 내에서 인터럽트가 발생하도록 설정하는 인터럽트이며 낮은 우선순위를 갖는다. 트랩(trap), 예외(Exception)이 여기에 포함된다.

## EXTI(External interrupt/event controller)
외부에서 신호가 입력될 경우 장치에서 이벤트 또는 Interrupt가 발생되는 기능을 가지고 있으며 입력 받을 수 있는 신호는 Rising-Edge, Falling-Edge, Rising and Falling Edge가 있다.

![External interrupt/event GPIO mapping](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/9d09d408-f171-4472-af6a-b0374cd916bf)

(그림1: External interrupt/event GPIO mapping)

EXTICRx Register를 통해 입력받을 Port를 선택할 수 있다. 각 Port의 n번 Pin(PAn, PBn, PCn, ..., PGn)은 EXTIn에 연결되어 있다. 또한 (PAn, PBn, PCn, ..., PGn) 중 어떤 것을 선택할지는 AFIO_EXTICR1 register(EXTI0, EXTI1, EXTI2, EXTI3), AFIO_EXTICR2 register(EXTI4, EXTI5, EXTI6, EXTI7), AFIO_EXTICR3 register(EXTI8, EXTI9, EXTI10, EXTI11), AFIO_EXTICR4 register(EXTI12, EXTI13, EXTI14, EXTI15)로 설정할 수 있다.

![그림2: EXTI0, EXTI1, EXTI2, EXTI3를 설정할 수 있는 External interrupt configuration register EXTIx에 들어가는 값에 따라 PA(x) 부터 PG(x)까지 설정할 수 있다.](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/ecebd4de-cb78-4c5b-af1b-3729829e89c9)

(그림2: EXTI0, EXTI1, EXTI2, EXTI3를 설정할 수 있는 External interrupt configuration register. EXTIx에 들어가는 값에 따라 PA(x) 부터 PG(x)까지 설정할 수 있다.)

![그림3: AFIO의 주소. AFIO 주소는 0x4001 0000 이므로, 그림2의 AFIO_EXTICR1 register의 주소는 0x4001 0008 이다.](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/d3763dbb-633d-49c3-9095-90c6eca44730)

(그림3: AFIO의 주소. AFIO 주소는 0x4001 0000 이므로, 그림2의 AFIO_EXTICR1 register의 주소는 0x4001 0008 이다.)

EXTI에서 Event모드와 Interrupt 모드를 선택할 수 있으며 Interrupt 모드일 경우 Interrupt 발생시 해당하는 Interrupt Handler가 동작한다. 우선순위가 높은 순으로 Interrupt를 처리한다. 20개의 Edge Detector Line으로 구성되어 각 Line이 설정에 따라 Rising/Falling trigger를 감지한다.

![그림4: External interrupt/event controller block diagram](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/55be9dcc-d1f8-4653-b656-875f35bd03b9)

(그림4: External interrupt/event controller block diagram)

Input line에서 Edge signal이 발생하면 Rising tirgger selection register와 Falling trigger selection register와 연결되어있는 Edge detect circuit가 Rising edge/ Falling edge가 발생했는지 확인하고, 소프트웨어 인터럽트 이벤트 레지스터의 결과와 합친다. 그리고 interrupt mask register와 and 연산하여 enable 된 interrupt 신호만 통과하고, 통과한 신호는 Pending request register의 bit를 set한다. 프로세서는 pending register를 검사하여 발생된 interrupt 중 가장 높은 interrupt를 처리하게 된다.

![그림5: EXTI16, EXTI17, EXTI18, EXTI19는 위 그림과 같은 이벤트를 감지한다](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/7107a294-472b-40a5-8c7a-5752a244c6a5)

(그림5: EXTI16, EXTI17, EXTI18, EXTI19는 위 그림과 같은 이벤트를 감지한다)

![그림6: Libraries\CMSIS\DeviceSupport\Startup\startup_stm32f10x_cl.s](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/167d4e9b-36e3-4041-a20a-0b681b81a8fd)

(그림6: Libraries\CMSIS\DeviceSupport\Startup\startup_stm32f10x_cl.s)

그림6과 같이, startup_stm32f10x_cl.s에는 각 인터럽트 핸들러에서 호출되는 함수의 프로토타입이 정의되어 있다. 4번 핀을 사용시 EXTI4_IRQHandler, 8번핀 사용시 EXTI9_5_IRQHandler로 핸들러를 선언한다(13번핀 사용시 EXTI15_10_IRQHandler).

## NVIC(Nested Vectored Interrupt Controller)

인터럽트 처리 중 또다른 인터럽트가 발생하면 우선순위에 따라 우선순위가 높은 인터럽트부터 처리 후 다른 인터럽트를 처리한다. 값이 작을 수록 우선순위가 높다.

```c
typedef struct
{
  uint8_t NVIC_IRQChannel;                    /*!< Specifies the IRQ channel to be enabled or disabled.
                                                   This parameter can be a value of @ref IRQn_Type 
                                                   (For the complete STM32 Devices IRQ Channels list, please
                                                    refer to stm32f10x.h file) */

  uint8_t NVIC_IRQChannelPreemptionPriority;  /*!< Specifies the pre-emption priority for the IRQ channel
                                                   specified in NVIC_IRQChannel. This parameter can be a value
                                                   between 0 and 15 as described in the table @ref NVIC_Priority_Table */

  uint8_t NVIC_IRQChannelSubPriority;         /*!< Specifies the subpriority level for the IRQ channel specified
                                                   in NVIC_IRQChannel. This parameter can be a value
                                                   between 0 and 15 as described in the table @ref NVIC_Priority_Table */

  FunctionalState NVIC_IRQChannelCmd;         /*!< Specifies whether the IRQ channel defined in NVIC_IRQChannel
                                                   will be enabled or disabled. 
                                                   This parameter can be set either to ENABLE or DISABLE */   
} NVIC_InitTypeDef;
//Libraries\STM32F10x_StdPeriph_Driver_v3.5\inc\misc.h
```
NVIC_InitTypeDef 구조체를 통해 어떤 인터럽트를 설정할지 설정하고(NVIC_IRQChannel), 우선순위를 설정하여(NVIC_IRQChannelPreemptionPriority) 선점 우선순위가 결정되고, sub priority를 설정하여(NVIC_IRQChannelSubPriority) 아직 대기 중인 ISR들의 순서가 결정된다.

## 레지스터 설정에 구조체 사용

```c
typedef struct
{
  uint16_t GPIO_Pin;             /*!< Specifies the GPIO pins to be configured.
                                      This parameter can be any value of @ref GPIO_pins_define */

  GPIOSpeed_TypeDef GPIO_Speed;  /*!< Specifies the speed for the selected pins.
                                      This parameter can be a value of @ref GPIOSpeed_TypeDef */

  GPIOMode_TypeDef GPIO_Mode;    /*!< Specifies the operating mode for the selected pins.
                                      This parameter can be a value of @ref GPIOMode_TypeDef */
}GPIO_InitTypeDef;
```

```cpp
/* LED pin setting*/
    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_2 | GPIO_Pin_3 | GPIO_Pin_4 | GPIO_Pin_7;
    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;
    GPIO_Init(GPIOD, &GPIO_InitStructure);
```
구조체에 레지스터 설정값을 넣고 xxxx_Init()에 설정할 레지스터와 구조체의 주소를 넣으면 레지스터가 구조체에 있는 설정값으로 설정된다.

# 세부 실험 내용
- Datasheet 및 Reference Manual 참고하여 해당 레지스터 및 주소에 대한 설정 이해
- NVIC와 EXTI를 이용하여 GPIO에 인터럽트 핸들링 세팅(ISR 동작은 최대한 빨리 끝나야함)
- 보드를 켜면 LED 물결 기능 유지 (LED 1->2->3->4->1->2->3->4->1->… 반복)
- A: LED 물결 방향 변경 - 1->2->3->4 
- B: LED 물결 방향 변경 - 4->3->2->1 
- 물결 속도는 delay를 이용하여 천천히 동작, ISR에서는 delay가 없어야 한다.
- 전원 스위치에 가까운 LED = 1이고 차례대로 2,3,4
- S1 버튼 : A 동작, S2 버튼 : B 동작
- PC의 Putty에서 a, b 문자 입력하여 보드 제어 (PC -> 보드 명령) (‘a’ : A 동작, ‘b’ : B 동작)
- S3 버튼을 누를 경우 Putty로 “TEAMXX.\r\n“ 출력

# 실험 과정

## RCC_APB2에 클럭 인가

```cpp
void RCC_Configure(void)
{
   /* UART TX/RX port clock enable */
   RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);

   // -PC
   RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOC, ENABLE);
   // -PB
   RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOB, ENABLE);

   /* LED port clock enable */
   RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOD, ENABLE);

   /* USART1 clock enable */
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_USART1, ENABLE);

   /* Alternate Function IO clock enable */
   RCC_APB2PeriphClockCmd(RCC_APB2Periph_AFIO, ENABLE);
}
```
이번 실험에서 쓸 GPIOA, GPIOB, GPIOC, GPIOD, USART1, AFIO 핀을 활성화시킨다. 활성화 시킬 때 stm32f10x_rcc.h에 정의된 RCC_APB2PeriphClockCmd를 활용한다.

RCC_APB2PeriphClockCmd에 대한 설명은 아래와 같다.

```c
/**
  * @brief  Enables or disables the High Speed APB (APB2) peripheral clock.
  * @param  RCC_APB2Periph: specifies the APB2 peripheral to gates its clock.
  *   This parameter can be any combination of the following values:
  *     @arg RCC_APB2Periph_AFIO, RCC_APB2Periph_GPIOA, RCC_APB2Periph_GPIOB,
  *          RCC_APB2Periph_GPIOC, RCC_APB2Periph_GPIOD, RCC_APB2Periph_GPIOE,
  *          RCC_APB2Periph_GPIOF, RCC_APB2Periph_GPIOG, RCC_APB2Periph_ADC1,
  *          RCC_APB2Periph_ADC2, RCC_APB2Periph_TIM1, RCC_APB2Periph_SPI1,
  *          RCC_APB2Periph_TIM8, RCC_APB2Periph_USART1, RCC_APB2Periph_ADC3,
  *          RCC_APB2Periph_TIM15, RCC_APB2Periph_TIM16, RCC_APB2Periph_TIM17,
  *          RCC_APB2Periph_TIM9, RCC_APB2Periph_TIM10, RCC_APB2Periph_TIM11     
  * @param  NewState: new state of the specified peripheral clock.
  *   This parameter can be: ENABLE or DISABLE.
  * @retval None
  */
void RCC_APB2PeriphClockCmd(uint32_t RCC_APB2Periph, FunctionalState NewState)

//Libraries\STM32F10x_StdPeriph_Driver_v3.5\src\stm32f10x_rcc.c
```

## GPIO 설정
이번 실험에 사용되는 핀은 PA9(TX), PA10(RX), PB10(KEY2), PC4(KEY1), PC13(KEY3), PD2(LED1), PD3(LED2), PD4(LED3), PD7(LED4)이다. 또한 input mode(input with pull-up)는 KEY1, KEY2, KEY3이고, input mode(input with pull-down)은 RX이다. output mode(alternative output push-pull)는 TX이고 output mode(General purpose pusl-pull)은 LED이다.

구조체를 이용하여 레지스터를 설정하는 방법은 다음과 같다. GPIO_InitTypeDef 구조체를 하나 선언 후, 구조체 내부의 GPIO_Pin, GPIO_Speed, GPIO_Mode를 설정한다.

```c
typedef struct
{
  uint16_t GPIO_Pin;             /*!< Specifies the GPIO pins to be configured.
                                      This parameter can be any value of @ref GPIO_pins_define */

  GPIOSpeed_TypeDef GPIO_Speed;  /*!< Specifies the speed for the selected pins.
                                      This parameter can be a value of @ref GPIOSpeed_TypeDef */

  GPIOMode_TypeDef GPIO_Mode;    /*!< Specifies the operating mode for the selected pins.
                                      This parameter can be a value of @ref GPIOMode_TypeDef */
}GPIO_InitTypeDef;
//Libraries\STM32F10x_StdPeriph_Driver_v3.5\inc\stm32f10x_gpio.h
```

GPIO_Pin에 들어갈 수 있는 값은 아래와 같다.
```cpp
/** @defgroup GPIO_pins_define 
  * @{
  */

#define GPIO_Pin_0                 ((uint16_t)0x0001)  /*!< Pin 0 selected */
#define GPIO_Pin_1                 ((uint16_t)0x0002)  /*!< Pin 1 selected */
#define GPIO_Pin_2                 ((uint16_t)0x0004)  /*!< Pin 2 selected */
#define GPIO_Pin_3                 ((uint16_t)0x0008)  /*!< Pin 3 selected */
#define GPIO_Pin_4                 ((uint16_t)0x0010)  /*!< Pin 4 selected */
#define GPIO_Pin_5                 ((uint16_t)0x0020)  /*!< Pin 5 selected */
#define GPIO_Pin_6                 ((uint16_t)0x0040)  /*!< Pin 6 selected */
#define GPIO_Pin_7                 ((uint16_t)0x0080)  /*!< Pin 7 selected */
#define GPIO_Pin_8                 ((uint16_t)0x0100)  /*!< Pin 8 selected */
#define GPIO_Pin_9                 ((uint16_t)0x0200)  /*!< Pin 9 selected */
#define GPIO_Pin_10                ((uint16_t)0x0400)  /*!< Pin 10 selected */
#define GPIO_Pin_11                ((uint16_t)0x0800)  /*!< Pin 11 selected */
#define GPIO_Pin_12                ((uint16_t)0x1000)  /*!< Pin 12 selected */
#define GPIO_Pin_13                ((uint16_t)0x2000)  /*!< Pin 13 selected */
#define GPIO_Pin_14                ((uint16_t)0x4000)  /*!< Pin 14 selected */
#define GPIO_Pin_15                ((uint16_t)0x8000)  /*!< Pin 15 selected */
#define GPIO_Pin_All               ((uint16_t)0xFFFF)  /*!< All pins selected */

//Libraries\STM32F10x_StdPeriph_Driver_v3.5\inc\stm32f10x_gpio.h
```
GPIO_Pin에는 위와 같이 0번핀부터 15번핀(또는 모든 핀)을 선택할 수 있다.

GPIO_Speed에는 아래와 같은 값을 설정할 수 있다.

```c
typedef enum
{ 
  GPIO_Speed_10MHz = 1,
  GPIO_Speed_2MHz, 
  GPIO_Speed_50MHz
}GPIOSpeed_TypeDef;
//Libraries\STM32F10x_StdPeriph_Driver_v3.5\inc\stm32f10x_gpio.h
```
Speed를 10MHz, 2MHz 50MHz로 설정할 수 있다.

GPIO_Mode에는 아래와 같은 값이 들어갈 수 있다.

```c
typedef enum
{ GPIO_Mode_AIN = 0x0,
  GPIO_Mode_IN_FLOATING = 0x04,
  GPIO_Mode_IPD = 0x28,
  GPIO_Mode_IPU = 0x48,
  GPIO_Mode_Out_OD = 0x14,
  GPIO_Mode_Out_PP = 0x10,
  GPIO_Mode_AF_OD = 0x1C,
  GPIO_Mode_AF_PP = 0x18
}GPIOMode_TypeDef;
//\Libraries\STM32F10x_StdPeriph_Driver_v3.5\inc\stm32f10x_gpio.h
```
GPIO_Mode를 통해 input mode(pull up, pull down, 아날로그 input 등), output mode(push/pull 또는 open-drain)를 설정할 수 있다.

```c
/**
  * @brief  Initializes the GPIOx peripheral according to the specified
  *         parameters in the GPIO_InitStruct.
  * @param  GPIOx: where x can be (A..G) to select the GPIO peripheral.
  * @param  GPIO_InitStruct: pointer to a GPIO_InitTypeDef structure that
  *         contains the configuration information for the specified GPIO peripheral.
  * @retval None
  */
void GPIO_Init(GPIO_TypeDef* GPIOx, GPIO_InitTypeDef* GPIO_InitStruct)

//Libraries\STM32F10x_StdPeriph_Driver_v3.5\src\stm32f10x_gpio.c
```

마지막으로 GPIO_Init() 함수와 GPIO_InitTypeDef를 이용하여 레지스터를 설정한다. GPIO_Init의 첫번째 인자는 GPIOA 부터 GPIOG까지, 두번째 인자는 방금 설정한 GPIO_InitTypeDef 구조체이다.

따라서 구조체와 함수를 활용하여 아래와 같이 GPIO를 설정한다.

```c
void GPIO_Configure(void)
{
    GPIO_InitTypeDef GPIO_InitStructure;

   // TODO: Initialize the GPIO pins using the structure 'GPIO_InitTypeDef' and the function 'GPIO_Init'

    GPIO_InitTypeDef GPIOA_input;
    GPIO_InitTypeDef GPIOA_output;
    GPIO_InitTypeDef GPIOB_input;
    GPIO_InitTypeDef GPIOC_input;
    GPIO_InitTypeDef GPIOD_input;

    /* Button S1, S2, S3 pin setting */
    GPIOC_input.GPIO_Pin = GPIO_Pin_4 | GPIO_Pin_13;
    GPIOB_input.GPIO_Pin = GPIO_Pin_10;
    GPIOC_input.GPIO_Mode = GPIO_Mode_IPU;
    GPIOB_input.GPIO_Mode = GPIO_Mode_IPU;
    GPIO_Init(GPIOC, &GPIOC_input);
    GPIO_Init(GPIOB, &GPIOC_input);
    
    /* LED pin setting*/
    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_2 | GPIO_Pin_3 | GPIO_Pin_4 | GPIO_Pin_7;
    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;
    GPIO_Init(GPIOD, &GPIO_InitStructure);
   
    /* UART pin setting */
    //TX
    GPIOA_output.GPIO_Pin = GPIO_Pin_9;
    GPIOA_output.GPIO_Speed = GPIO_Speed_50MHz;
    GPIOA_output.GPIO_Mode = GPIO_Mode_AF_PP;
    GPIO_Init(GPIOA, &GPIOA_output);
    
   //RX
    GPIOA_input.GPIO_Pin = GPIO_Pin_10;
    GPIOA_input.GPIO_Mode = GPIO_Mode_IPD;
    GPIO_Init(GPIOA, &GPIOA_input);
   
}
```
Button s1,s2,s3는 PC4, PB10, PC13이고 input mode로 설정한다.

LED는 PD2, PD3, PD4, PD7이고, output mode push pull(50MHz)로 설정한다.

TX는 PA9이고, output mode alternative function push pull(50MHz)로 설정한다.

RX는 PD10이고, input mode로 설정한다.

## EXTI 설정
```c
void EXTI_Configure(void)
{
    EXTI_InitTypeDef EXTI_InitStructure;

   // TODO: Select the GPIO pin (Joystick, button) used as EXTI Line using function 'GPIO_EXTILineConfig'
   // TODO: Initialize the EXTI using the structure 'EXTI_InitTypeDef' and the function 'EXTI_Init'
   
    /* Button S1 */
   GPIO_EXTILineConfig(GPIO_PortSourceGPIOC, GPIO_PinSource4);
    EXTI_InitStructure.EXTI_Line = EXTI_Line4;
    EXTI_InitStructure.EXTI_Mode = EXTI_Mode_Interrupt;
    EXTI_InitStructure.EXTI_Trigger = EXTI_Trigger_Falling;
    EXTI_InitStructure.EXTI_LineCmd = ENABLE;
    EXTI_Init(&EXTI_InitStructure);

    /* Button S2 */
   GPIO_EXTILineConfig(GPIO_PortSourceGPIOB, GPIO_PinSource10);
    EXTI_InitStructure.EXTI_Line = EXTI_Line10;
    EXTI_InitStructure.EXTI_Mode = EXTI_Mode_Interrupt;
    EXTI_InitStructure.EXTI_Trigger = EXTI_Trigger_Falling;
    EXTI_InitStructure.EXTI_LineCmd = ENABLE;
    EXTI_Init(&EXTI_InitStructure);

   /* Button S3 */
   GPIO_EXTILineConfig(GPIO_PortSourceGPIOC, GPIO_PinSource13);
    EXTI_InitStructure.EXTI_Line = EXTI_Line13;
    EXTI_InitStructure.EXTI_Mode = EXTI_Mode_Interrupt;
    EXTI_InitStructure.EXTI_Trigger = EXTI_Trigger_Falling;
    EXTI_InitStructure.EXTI_LineCmd = ENABLE;
    EXTI_Init(&EXTI_InitStructure);
   // NOTE: do not select the UART GPIO pin used as EXTI Line here
}

```
버튼핀인 PC4, PB10, PC13을 EXTI Line으로 쓰기 위해 GPIO_EXTILineConfig함수를 사용한다. GPIO_EXTILineConfig 함수에는 GPIO 포트 이름과 핀 번호가 들어간다. 그리고 EXTI_InitTypeDef 구조체를 사용하여 EXTI를 초기화한다. 이 구조체에 들어가는 값은 EXTI_Line, EXTI_Mode, EXTI_Trigger, EXTI_LineCmd가 있다. EXTI_Line에는 사용할 EXTI 라인을 설정할 수 있다. 4번핀, 10번핀, 13번핀이므로 EXTI4, EXTI10, EXTI13 라인을 활성화시킨다. 

그리고 EXTI_Mode에는 인터럽트모드와 이벤트모드가 들어갈 수 있으며, 인터럽트로 활용할 것이기 때문에 EXTI_Mode_Interrupt로 설정한다(이벤트로 설정하려면 EXTI_Mode_Event로 설정한다). 

EXTI_Trigger에는 Rising, Falling, Rising_Falling을 설정할 수 있는데, 여기서는 EXTI_Trigger_Falling으로 설정한다(Rising으로 설정하고 싶으면 EXTI_Trigger_Rising, Rising_Falling으로 설정하고 싶으면 EXTI_Trigger_Rising_Falling으로 설정한다). Trigger_Falling으로 설정하는 이유는 버튼이 pull-up 방식으로 연결되어있어, 버튼을 누르면 1에서 0으로 falling이 일어나기 때문이다.

EXTI_LineCmd에는 그 라인을 활성화할 것인지 여부를 설정한다. ENABLE와 DISABLE이 들어갈 수 있으며, 활성화해야하므로 ENABLE로 설정한다.

마지막으로 EXTI_Init()을 통해 EXTI 레지스터를 EXTI_InitStruct에 설정한대로 설정한다.

### 참고

#### EXTI_InitTypeDef
EXTI_InitTypeDef 구조체와 관련 구조체 구조, 그리고 EXTI Line은 아래와 같다.
```c

typedef struct
{
  uint32_t EXTI_Line;               /*!< Specifies the EXTI lines to be enabled or disabled.
                                         This parameter can be any combination of @ref EXTI_Lines */
   
  EXTIMode_TypeDef EXTI_Mode;       /*!< Specifies the mode for the EXTI lines.
                                         This parameter can be a value of @ref EXTIMode_TypeDef */

  EXTITrigger_TypeDef EXTI_Trigger; /*!< Specifies the trigger signal active edge for the EXTI lines.
                                         This parameter can be a value of @ref EXTIMode_TypeDef */

  FunctionalState EXTI_LineCmd;     /*!< Specifies the new state of the selected EXTI lines.
                                         This parameter can be set either to ENABLE or DISABLE */ 
}EXTI_InitTypeDef;

typedef enum
{
  EXTI_Mode_Interrupt = 0x00,
  EXTI_Mode_Event = 0x04
}EXTIMode_TypeDef;

typedef enum
{
  EXTI_Trigger_Rising = 0x08,
  EXTI_Trigger_Falling = 0x0C,  
  EXTI_Trigger_Rising_Falling = 0x10
}EXTITrigger_TypeDef;

/** @defgroup EXTI_Lines 
  * @{
  */

#define EXTI_Line0       ((uint32_t)0x00001)  /*!< External interrupt line 0 */
#define EXTI_Line1       ((uint32_t)0x00002)  /*!< External interrupt line 1 */
#define EXTI_Line2       ((uint32_t)0x00004)  /*!< External interrupt line 2 */
#define EXTI_Line3       ((uint32_t)0x00008)  /*!< External interrupt line 3 */
#define EXTI_Line4       ((uint32_t)0x00010)  /*!< External interrupt line 4 */
#define EXTI_Line5       ((uint32_t)0x00020)  /*!< External interrupt line 5 */
#define EXTI_Line6       ((uint32_t)0x00040)  /*!< External interrupt line 6 */
#define EXTI_Line7       ((uint32_t)0x00080)  /*!< External interrupt line 7 */
#define EXTI_Line8       ((uint32_t)0x00100)  /*!< External interrupt line 8 */
#define EXTI_Line9       ((uint32_t)0x00200)  /*!< External interrupt line 9 */
#define EXTI_Line10      ((uint32_t)0x00400)  /*!< External interrupt line 10 */
#define EXTI_Line11      ((uint32_t)0x00800)  /*!< External interrupt line 11 */
#define EXTI_Line12      ((uint32_t)0x01000)  /*!< External interrupt line 12 */
#define EXTI_Line13      ((uint32_t)0x02000)  /*!< External interrupt line 13 */
#define EXTI_Line14      ((uint32_t)0x04000)  /*!< External interrupt line 14 */
#define EXTI_Line15      ((uint32_t)0x08000)  /*!< External interrupt line 15 */
#define EXTI_Line16      ((uint32_t)0x10000)  /*!< External interrupt line 16 Connected to the PVD Output */
#define EXTI_Line17      ((uint32_t)0x20000)  /*!< External interrupt line 17 Connected to the RTC Alarm event */
#define EXTI_Line18      ((uint32_t)0x40000)  /*!< External interrupt line 18 Connected to the USB Device/USB OTG FS
                                                   Wakeup from suspend event */                                    
#define EXTI_Line19      ((uint32_t)0x80000)  /*!< External interrupt line 19 Connected to the Ethernet Wakeup event */

//Libraries\STM32F10x_StdPeriph_Driver_v3.5\inc\stm32f10x_exti.h
```

#### GPIO_EXTILineConfig()

GPIO_EXTILineConfig 함수 설명은 아래와 같다.
```c
/**
  * @brief  Selects the GPIO pin used as EXTI Line.
  * @param  GPIO_PortSource: selects the GPIO port to be used as source for EXTI lines.
  *   This parameter can be GPIO_PortSourceGPIOx where x can be (A..G).
  * @param  GPIO_PinSource: specifies the EXTI line to be configured.
  *   This parameter can be GPIO_PinSourcex where x can be (0..15).
  * @retval None
  */
void GPIO_EXTILineConfig(uint8_t GPIO_PortSource, uint8_t GPIO_PinSource)

//Libraries\STM32F10x_StdPeriph_Driver_v3.5\src\stm32f10x_gpio.c
```

#### EXTI_Init()

EXTI_Init 함수 설명은 아래와 같다.
```c
/**
  * @brief  Initializes the EXTI peripheral according to the specified
  *         parameters in the EXTI_InitStruct.
  * @param  EXTI_InitStruct: pointer to a EXTI_InitTypeDef structure
  *         that contains the configuration information for the EXTI peripheral.
  * @retval None
  */
void EXTI_Init(EXTI_InitTypeDef* EXTI_InitStruct)

//Libraries\STM32F10x_StdPeriph_Driver_v3.5\src\stm32f10x_exti.c
```

## USART1 설정

```c
void USART1_Init(void)
{
   USART_InitTypeDef USART1_InitStructure;

   // Enable the USART1 peripheral
   USART_Cmd(USART1, ENABLE);
   
   // TODO: Initialize the USART using the structure 'USART_InitTypeDef' and the function 'USART_Init'
   USART1_InitStructure.USART_BaudRate = 9600;
    USART1_InitStructure.USART_WordLength = USART_WordLength_8b;
    USART1_InitStructure.USART_StopBits = USART_StopBits_1;
    USART1_InitStructure.USART_Parity = USART_Parity_No;
    USART1_InitStructure.USART_Mode = USART_Mode_Rx | USART_Mode_Tx;
    USART1_InitStructure.USART_HardwareFlowControl = USART_HardwareFlowControl_None;
    USART_Init(USART1, &USART1_InitStructure);
   
   // TODO: Enable the USART1 RX interrupts using the function 'USART_ITConfig' and the argument value 'Receive Data register not empty interrupt'
    USART_ITConfig(USART1, USART_IT_RXNE, ENABLE);
   
}
```
먼저 USART_Cmd로 어느 USART를 활성화할 것인지 설정한다. 첫번째 인자는 USART1, USART2... 등 USART 번호이고, 2번째인자는 활성화 여부(ENABLE, DISABLE이다).

그리고 USART1을 초기화하기 위해 USART_InitTypeDef 구조체를 활용한다. USART_BaudRate에는 사용할 BaudRate를 대입하고 USART_WordLength를 통해 WordLength를 8bit로 설정(USART_WordLength_8b)한다. USART_Stopbits를 통해 StopBits를 1비트로 설정(USART_StopBits_1)하고 USART_Parity를 통해 Parity bit를 비활성화한다(USART_Parity_No). 그리고 USART_Mode를 통해 RX,TX 둘다 사용할 것이라고 설정한다(USART_Mode_RX or USART_Mode_Tx). 마지막으로, nCTS핀이 활성화되어있을 때만 데이터를 전송하는 HardwareFlowControl을 비활성화하기 위해 USART_HardwareFlowControl을 USART_HardwareFlowControl_None으로 설정한다.

그리고 USART_Init을 호출하여 USART1_InitStructure 구조체에 설정한대로 USART1 관련 레지스터를 설정한다. 마지막으로, USART_ITConfig()를 통해 데이터를 받았을 때 인터럽트가 발생하도록 설정한다. USART_ITConfig()의 첫번째 인자는 USART번호(USART1,USART2,USART3,UART4,UART5)이고 2번째 인자는 언제 인터럽트를 발생시킬지를 설정하는 것으로서, 보낼 데이터가 없을 때(USART_IT_TXE), 데이터를 다 보냈을 때(USART_IT_TC), 패리티 에러가 났을때(USART_IT_PE), 데이터를 받았을 때(USART_IT_RXNE) 등일때 인터럽트를 발생시킬 수 있으며, 여기서는 USART_IT_RXNE로 설정한다.

그리고 3번째 인자는 인터럽트 활성화 여부로서, 활성화해야하므로 ENABLE로 설정한다.

### 참고

#### USART_InitTypeDef

USART_InitTypeDef 구조체와 관련 매크로는 아래와 같다.
```c
typedef struct
{
  uint32_t USART_BaudRate;            /*!< This member configures the USART communication baud rate.
                                           The baud rate is computed using the following formula:
                                            - IntegerDivider = ((PCLKx) / (16 * (USART_InitStruct->USART_BaudRate)))
                                            - FractionalDivider = ((IntegerDivider - ((u32) IntegerDivider)) * 16) + 0.5 */

  uint16_t USART_WordLength;          /*!< Specifies the number of data bits transmitted or received in a frame.
                                           This parameter can be a value of @ref USART_Word_Length */

  uint16_t USART_StopBits;            /*!< Specifies the number of stop bits transmitted.
                                           This parameter can be a value of @ref USART_Stop_Bits */

  uint16_t USART_Parity;              /*!< Specifies the parity mode.
                                           This parameter can be a value of @ref USART_Parity
                                           @note When parity is enabled, the computed parity is inserted
                                                 at the MSB position of the transmitted data (9th bit when
                                                 the word length is set to 9 data bits; 8th bit when the
                                                 word length is set to 8 data bits). */
 
  uint16_t USART_Mode;                /*!< Specifies wether the Receive or Transmit mode is enabled or disabled.
                                           This parameter can be a value of @ref USART_Mode */

  uint16_t USART_HardwareFlowControl; /*!< Specifies wether the hardware flow control mode is enabled
                                           or disabled.
                                           This parameter can be a value of @ref USART_Hardware_Flow_Control */
} USART_InitTypeDef;

/** @defgroup USART_Word_Length 
  * @{
  */ 
  
#define USART_WordLength_8b                  ((uint16_t)0x0000)
#define USART_WordLength_9b                  ((uint16_t)0x1000)

/** @defgroup USART_Stop_Bits 
  * @{
  */ 
  
#define USART_StopBits_1                     ((uint16_t)0x0000)
#define USART_StopBits_0_5                   ((uint16_t)0x1000)
#define USART_StopBits_2                     ((uint16_t)0x2000)
#define USART_StopBits_1_5                   ((uint16_t)0x3000)

/** @defgroup USART_Parity 
  * @{
  */ 
  
#define USART_Parity_No                      ((uint16_t)0x0000)
#define USART_Parity_Even                    ((uint16_t)0x0400)
#define USART_Parity_Odd                     ((uint16_t)0x0600) 

/** @defgroup USART_Mode 
  * @{
  */ 
  
#define USART_Mode_Rx                        ((uint16_t)0x0004)
#define USART_Mode_Tx                        ((uint16_t)0x0008)

/** @defgroup USART_Hardware_Flow_Control 
  * @{
  */ 
#define USART_HardwareFlowControl_None       ((uint16_t)0x0000)
#define USART_HardwareFlowControl_RTS        ((uint16_t)0x0100)
#define USART_HardwareFlowControl_CTS        ((uint16_t)0x0200)
#define USART_HardwareFlowControl_RTS_CTS    ((uint16_t)0x0300)
//Libraries\STM32F10x_StdPeriph_Driver_v3.5\inc\stm32f10x_usart.h
```

#### USART_Cmd()

USART_Cmd 함수에 대한 설명은 아래와 같다.

```c
/**
  * @brief  Enables or disables the specified USART peripheral.
  * @param  USARTx: Select the USART or the UART peripheral. 
  *         This parameter can be one of the following values:
  *           USART1, USART2, USART3, UART4 or UART5.
  * @param  NewState: new state of the USARTx peripheral.
  *         This parameter can be: ENABLE or DISABLE.
  * @retval None
  */
void USART_Cmd(USART_TypeDef* USARTx, FunctionalState NewState)
//Libraries\STM32F10x_StdPeriph_Driver_v3.5\src\stm32f10x_usart.c
```

#### USART_Init()

USART_Init 함수에 대한 설명은 아래와 같다.

```c
/**
  * @brief  Initializes the USARTx peripheral according to the specified
  *         parameters in the USART_InitStruct .
  * @param  USARTx: Select the USART or the UART peripheral. 
  *   This parameter can be one of the following values:
  *   USART1, USART2, USART3, UART4 or UART5.
  * @param  USART_InitStruct: pointer to a USART_InitTypeDef structure
  *         that contains the configuration information for the specified USART 
  *         peripheral.
  * @retval None
  */
void USART_Init(USART_TypeDef* USARTx, USART_InitTypeDef* USART_InitStruct)
//Libraries\STM32F10x_StdPeriph_Driver_v3.5\src\stm32f10x_usart.c
```

#### USART_ITConfig()

USART_ITConfig 함수에 대한 설명은 아래와 같다.

```c
/**
  * @brief  Enables or disables the specified USART interrupts.
  * @param  USARTx: Select the USART or the UART peripheral. 
  *   This parameter can be one of the following values:
  *   USART1, USART2, USART3, UART4 or UART5.
  * @param  USART_IT: specifies the USART interrupt sources to be enabled or disabled.
  *   This parameter can be one of the following values:
  *     @arg USART_IT_CTS:  CTS change interrupt (not available for UART4 and UART5)
  *     @arg USART_IT_LBD:  LIN Break detection interrupt
  *     @arg USART_IT_TXE:  Transmit Data Register empty interrupt
  *     @arg USART_IT_TC:   Transmission complete interrupt
  *     @arg USART_IT_RXNE: Receive Data register not empty interrupt
  *     @arg USART_IT_IDLE: Idle line detection interrupt
  *     @arg USART_IT_PE:   Parity Error interrupt
  *     @arg USART_IT_ERR:  Error interrupt(Frame error, noise error, overrun error)
  * @param  NewState: new state of the specified USARTx interrupts.
  *   This parameter can be: ENABLE or DISABLE.
  * @retval None
  */
void USART_ITConfig(USART_TypeDef* USARTx, uint16_t USART_IT, FunctionalState NewState)
//Libraries\STM32F10x_StdPeriph_Driver_v3.5\src\stm32f10x_usart.c
```

## NVIC 설정
```c
void NVIC_Configure(void) {

    NVIC_InitTypeDef NVIC_InitStructure;
    
    // TODO: fill the arg you want
    NVIC_PriorityGroupConfig(NVIC_PriorityGroup_0);

   // TODO: Initialize the NVIC using the structure 'NVIC_InitTypeDef' and the function 'NVIC_Init'
   
    // Button S1
    NVIC_InitStructure.NVIC_IRQChannel = EXTI4_IRQn; // PC4
    NVIC_InitStructure.NVIC_IRQChannelPreemptionPriority = 0; 
    NVIC_InitStructure.NVIC_IRQChannelSubPriority = 0; //        
    NVIC_InitStructure.NVIC_IRQChannelCmd = ENABLE;
    NVIC_Init(&NVIC_InitStructure); //     

    // Button S2, S3
    NVIC_InitStructure.NVIC_IRQChannel = EXTI15_10_IRQn; // PB10, PC13
    NVIC_InitStructure.NVIC_IRQChannelPreemptionPriority = 0; 
    NVIC_InitStructure.NVIC_IRQChannelSubPriority = 0; //               
    NVIC_InitStructure.NVIC_IRQChannelCmd = ENABLE;
    NVIC_Init(&NVIC_InitStructure); //    

    // UART1
   // 'NVIC_EnableIRQ' is only required for USART setting
    NVIC_EnableIRQ(USART1_IRQn);
    NVIC_InitStructure.NVIC_IRQChannel = USART1_IRQn;
    NVIC_InitStructure.NVIC_IRQChannelPreemptionPriority = 0; // TODO
    NVIC_InitStructure.NVIC_IRQChannelSubPriority = 0; // TODO
    NVIC_InitStructure.NVIC_IRQChannelCmd = ENABLE;
    NVIC_Init(&NVIC_InitStructure);
}
```
먼저 NVIC_PriorityGroupConfig 함수를 활용하여 preemption priority와 subpriority의 bit의 개수를 설정한다. 여기서는 NVIC_PriorityGroup_0로 지정하여 preemption priority bit는 0, subpriority를 4bit로 설정한다.

 그리고 NVIC_InitTypeDef 구조체를 활용하여 NVIC를 설정한다. 구조체 내부 변수인 NVIC_IRQChannelPreemptionPriority와 NVIC_IRQChannelSubPriority는 0부터 15까지 들어갈 수 있는데, 여기서는 0으로 모두 설정한다. 

그리고 NVIC_IRQChannel 변수에는 인터럽트 번호가 들어간다. Button s1은 PC4 포트이므로, EXTI4_IRQn이 들어가고, button s2, s3는 각각 PB10, PC13이므로 EXTI15_10_IRQn이 들어간다. UART1은 USART1_IRQn으로 설정한다. 특히 UART1은 이전에 NVIC_EnableIRQ로 USART_IRQn 인터럽트를 활성화해야한다.

마지막으로 NVIC_IRQChannelCmd를 이용해 해당 IRQ Channel을 활성화(ENABLE)시킨다. 그리고 NVIC_Init()을 이용해 설정한 구조체를 이용하여 실제 레지스터를 초기화시킨다.

### 참고

#### NVIC_InitTypeDef

NVIC_InitTypeDef 구조체의 구조와 관련 변수는 아래와 같다.

```c
typedef struct
{
  uint8_t NVIC_IRQChannel;                    /*!< Specifies the IRQ channel to be enabled or disabled.
                                                   This parameter can be a value of @ref IRQn_Type 
                                                   (For the complete STM32 Devices IRQ Channels list, please
                                                    refer to stm32f10x.h file) */

  uint8_t NVIC_IRQChannelPreemptionPriority;  /*!< Specifies the pre-emption priority for the IRQ channel
                                                   specified in NVIC_IRQChannel. This parameter can be a value
                                                   between 0 and 15 as described in the table @ref NVIC_Priority_Table */

  uint8_t NVIC_IRQChannelSubPriority;         /*!< Specifies the subpriority level for the IRQ channel specified
                                                   in NVIC_IRQChannel. This parameter can be a value
                                                   between 0 and 15 as described in the table @ref NVIC_Priority_Table */

  FunctionalState NVIC_IRQChannelCmd;         /*!< Specifies whether the IRQ channel defined in NVIC_IRQChannel
                                                   will be enabled or disabled. 
                                                   This parameter can be set either to ENABLE or DISABLE */   
} NVIC_InitTypeDef;

/** @defgroup NVIC_Priority_Table 
  * @{
  */

/**
@code  
 The table below gives the allowed values of the pre-emption priority and subpriority according
 to the Priority Grouping configuration performed by NVIC_PriorityGroupConfig function
  ============================================================================================================================
    NVIC_PriorityGroup   | NVIC_IRQChannelPreemptionPriority | NVIC_IRQChannelSubPriority  | Description
  ============================================================================================================================
   NVIC_PriorityGroup_0  |                0                  |            0-15             |   0 bits for pre-emption priority
                         |                                   |                             |   4 bits for subpriority
  ----------------------------------------------------------------------------------------------------------------------------
   NVIC_PriorityGroup_1  |                0-1                |            0-7              |   1 bits for pre-emption priority
                         |                                   |                             |   3 bits for subpriority
  ----------------------------------------------------------------------------------------------------------------------------    
   NVIC_PriorityGroup_2  |                0-3                |            0-3              |   2 bits for pre-emption priority
                         |                                   |                             |   2 bits for subpriority
  ----------------------------------------------------------------------------------------------------------------------------    
   NVIC_PriorityGroup_3  |                0-7                |            0-1              |   3 bits for pre-emption priority
                         |                                   |                             |   1 bits for subpriority
  ----------------------------------------------------------------------------------------------------------------------------    
   NVIC_PriorityGroup_4  |                0-15               |            0                |   4 bits for pre-emption priority
                         |                                   |                             |   0 bits for subpriority                       
  ============================================================================================================================
@endcode
*/
//Libraries\STM32F10x_StdPeriph_Driver_v3.5\inc\misc.h
```
```c
/******  STM32 specific Interrupt Numbers *********************************************************/
  WWDG_IRQn                   = 0,      /*!< Window WatchDog Interrupt                            */
  PVD_IRQn                    = 1,      /*!< PVD through EXTI Line detection Interrupt            */
  TAMPER_IRQn                 = 2,      /*!< Tamper Interrupt                                     */
  RTC_IRQn                    = 3,      /*!< RTC global Interrupt                                 */
  FLASH_IRQn                  = 4,      /*!< FLASH global Interrupt                               */
  RCC_IRQn                    = 5,      /*!< RCC global Interrupt                                 */
  EXTI0_IRQn                  = 6,      /*!< EXTI Line0 Interrupt                                 */
  EXTI1_IRQn                  = 7,      /*!< EXTI Line1 Interrupt                                 */
  EXTI2_IRQn                  = 8,      /*!< EXTI Line2 Interrupt                                 */
  EXTI3_IRQn                  = 9,      /*!< EXTI Line3 Interrupt                                 */
  EXTI4_IRQn                  = 10,     /*!< EXTI Line4 Interrupt                                 */
  DMA1_Channel1_IRQn          = 11,     /*!< DMA1 Channel 1 global Interrupt                      */

  ...

  #ifdef STM32F10X_CL
  ...
  SPI2_IRQn                   = 36,     /*!< SPI2 global Interrupt                                */
  USART1_IRQn                 = 37,     /*!< USART1 global Interrupt                              */
  USART2_IRQn                 = 38,     /*!< USART2 global Interrupt                              */
  USART3_IRQn                 = 39,     /*!< USART3 global Interrupt                              */
  EXTI15_10_IRQn              = 40,     /*!< External Line[15:10] Interrupts                      */
  ...

  //Libraries\CMSIS\DeviceSupport\stm32f10x.h
```

#### NVIC_PriorityGroupConfig()

NVIC_PriorityGroupConfig() 함수의 설명은 아래와 같다.

```c
/**
  * @brief  Configures the priority grouping: pre-emption priority and subpriority.
  * @param  NVIC_PriorityGroup: specifies the priority grouping bits length. 
  *   This parameter can be one of the following values:
  *     @arg NVIC_PriorityGroup_0: 0 bits for pre-emption priority
  *                                4 bits for subpriority
  *     @arg NVIC_PriorityGroup_1: 1 bits for pre-emption priority
  *                                3 bits for subpriority
  *     @arg NVIC_PriorityGroup_2: 2 bits for pre-emption priority
  *                                2 bits for subpriority
  *     @arg NVIC_PriorityGroup_3: 3 bits for pre-emption priority
  *                                1 bits for subpriority
  *     @arg NVIC_PriorityGroup_4: 4 bits for pre-emption priority
  *                                0 bits for subpriority
  * @retval None
  */
void NVIC_PriorityGroupConfig(uint32_t NVIC_PriorityGroup)
//Libraries\STM32F10x_StdPeriph_Driver_v3.5\src\misc.c
```

#### NVIC_Init()

NVIC_Init() 함수의 설명은 아래와 같다.

```c
/**
  * @brief  Initializes the NVIC peripheral according to the specified
  *         parameters in the NVIC_InitStruct.
  * @param  NVIC_InitStruct: pointer to a NVIC_InitTypeDef structure that contains
  *         the configuration information for the specified NVIC peripheral.
  * @retval None
  */
void NVIC_Init(NVIC_InitTypeDef* NVIC_InitStruct)

//Libraries\STM32F10x_StdPeriph_Driver_v3.5\src\misc.c
```

#### NVIC_EnableIRQ()

NVIC_EnableIRQ() 함수에 대한 설명은 아래와 같다.

```c
/**
 * @brief  Enable Interrupt in NVIC Interrupt Controller
 *
 * @param  IRQn   The positive number of the external interrupt to enable
 *
 * Enable a device specific interupt in the NVIC interrupt controller.
 * The interrupt number cannot be a negative value.
 */
static __INLINE void NVIC_EnableIRQ(IRQn_Type IRQn)

//CoreSupport\core_cm3.h
```

## USART1_IRQHandler

```c
void USART1_IRQHandler() {
   uint16_t word;
    if(USART_GetITStatus(USART1,USART_IT_RXNE)!=RESET){
       // the most recent received data by the USART1 peripheral
        word = USART_ReceiveData(USART1);

        // TODO implement
        if(word == 'a') {
            direction = 1;
        }else if (word == 'b') {
            direction = -1;
        }

        // clear 'Read data register not empty' flag
       USART_ClearITPendingBit(USART1,USART_IT_RXNE);
    }
}
```
USART1에서 인터럽트가 발생하였을 때 처리할 핸들러를 만든다. USART1에서 발생한 인터럽트가 USART_IT_RXNE(USART를 통해 어떤 데이터를 받았을 때)이면, 데이터를 받고, 그 데이터가 a이면 direction을 1로, b이면 -1로 설정한다. 인터럽트를 처리한 후에는, 처리한 인터럽트로 인해 다시 인터럽트가 발생하지 않도록 pending bit를 클리어한다(USART_ClearITPendingBit)

### 참고

#### USART_GetITStatus()

USART_GetITStatus() 함수에 대한 설명은 아래와 같다.
```c
/**
  * @brief  Checks whether the specified USART interrupt has occurred or not.
  * @param  USARTx: Select the USART or the UART peripheral. 
  *   This parameter can be one of the following values:
  *   USART1, USART2, USART3, UART4 or UART5.
  * @param  USART_IT: specifies the USART interrupt source to check.
  *   This parameter can be one of the following values:
  *     @arg USART_IT_CTS:  CTS change interrupt (not available for UART4 and UART5)
  *     @arg USART_IT_LBD:  LIN Break detection interrupt
  *     @arg USART_IT_TXE:  Tansmit Data Register empty interrupt
  *     @arg USART_IT_TC:   Transmission complete interrupt
  *     @arg USART_IT_RXNE: Receive Data register not empty interrupt
  *     @arg USART_IT_IDLE: Idle line detection interrupt
  *     @arg USART_IT_ORE:  OverRun Error interrupt
  *     @arg USART_IT_NE:   Noise Error interrupt
  *     @arg USART_IT_FE:   Framing Error interrupt
  *     @arg USART_IT_PE:   Parity Error interrupt
  * @retval The new state of USART_IT (SET or RESET).
  */
ITStatus USART_GetITStatus(USART_TypeDef* USARTx, uint16_t USART_IT)

//Libraries\STM32F10x_StdPeriph_Driver_v3.5\src\stm32f10x_usart.c
```

#### USART_ReceiveData()

USART_ReceiveData() 함수에 대한 설명은 아래와 같다.
```c
/**
  * @brief  Returns the most recent received data by the USARTx peripheral.
  * @param  USARTx: Select the USART or the UART peripheral. 
  *   This parameter can be one of the following values:
  *   USART1, USART2, USART3, UART4 or UART5.
  * @retval The received data.
  */
uint16_t USART_ReceiveData(USART_TypeDef* USARTx)

//Libraries\STM32F10x_StdPeriph_Driver_v3.5\src\stm32f10x_usart.c
```

#### USART_ClearITPendingBit()

USART_ClearITPendingBit() 함수에 대한 설명은 아래와 같다.
```c
/**
  * @brief  Clears the USARTx's interrupt pending bits.
  * @param  USARTx: Select the USART or the UART peripheral. 
  *   This parameter can be one of the following values:
  *   USART1, USART2, USART3, UART4 or UART5.
  * @param  USART_IT: specifies the interrupt pending bit to clear.
  *   This parameter can be one of the following values:
  *     @arg USART_IT_CTS:  CTS change interrupt (not available for UART4 and UART5)
  *     @arg USART_IT_LBD:  LIN Break detection interrupt
  *     @arg USART_IT_TC:   Transmission complete interrupt. 
  *     @arg USART_IT_RXNE: Receive Data register not empty interrupt.
  *   
  * @note
  *   - PE (Parity error), FE (Framing error), NE (Noise error), ORE (OverRun 
  *     error) and IDLE (Idle line detected) pending bits are cleared by 
  *     software sequence: a read operation to USART_SR register 
  *     (USART_GetITStatus()) followed by a read operation to USART_DR register 
  *     (USART_ReceiveData()).
  *   - RXNE pending bit can be also cleared by a read to the USART_DR register 
  *     (USART_ReceiveData()).
  *   - TC pending bit can be also cleared by software sequence: a read 
  *     operation to USART_SR register (USART_GetITStatus()) followed by a write 
  *     operation to USART_DR register (USART_SendData()).
  *   - TXE pending bit is cleared only by a write to the USART_DR register 
  *     (USART_SendData()).
  * @retval None
  */
void USART_ClearITPendingBit(USART_TypeDef* USARTx, uint16_t USART_IT)

//Libraries\STM32F10x_StdPeriph_Driver_v3.5\src\stm32f10x_usart.c
```

## 버튼 인터럽트 핸들러

```c
void EXTI15_10_IRQHandler(void) { // when the button is pressed
   if (EXTI_GetITStatus(EXTI_Line10) != RESET) {
      if (GPIO_ReadInputDataBit(GPIOB, GPIO_Pin_10) == Bit_RESET) {
         // TODO implement
         direction = -1;
      }
        EXTI_ClearITPendingBit(EXTI_Line10);
   }

   if (EXTI_GetITStatus(EXTI_Line13) != RESET) {
      if (GPIO_ReadInputDataBit(GPIOC, GPIO_Pin_13) == Bit_RESET) {
         // TODO implement
         for(int i = 0; msg[i] != '\0'; ++i) {
            sendDataUART1(msg[i]);
         }
      }
        EXTI_ClearITPendingBit(EXTI_Line13);
   }
}

void EXTI4_IRQHandler(void) { // when the button is pressed
   if (EXTI_GetITStatus(EXTI_Line4) != RESET) {
      if (GPIO_ReadInputDataBit(GPIOC, GPIO_Pin_4) == Bit_RESET) {
         // TODO implement
        direction = 1;
      }
        EXTI_ClearITPendingBit(EXTI_Line4);
   }
}
```
EXTI15_10_IRQHandler()에는 s2와 s3 버튼 인터럽트 핸들러를 만든다. Line10에서 인터럽트가 발생하였고, PB10의 입력값이 0이면 direction을 -1로 설정한다. Line13에서 인터럽트가 발생하고 PC13의 입력값이 0이면 UART1을 통해 메시지를 보낸다.

EXTI4_IRQHandler()에서는 s1 버튼 인터럽트 핸들러를 만든다. Line4에서 인터럽트가 발생하였고, PC4의 입력값이 0이면, direction을 1로 설정한다.

EXTI15_10_IRQHandler()와 EXTI4_IRQHandler() 모두, 인터럽트를 처리 후에는 처리한 인터럽트의 Pendingbit를 클리어한다.

### 참고

#### EXTI_GetITStatus()

EXTI_GetITStatus() 함수의 설명은 아래와 같다.

```c
/**
  * @brief  Checks whether the specified EXTI line is asserted or not.
  * @param  EXTI_Line: specifies the EXTI line to check.
  *   This parameter can be:
  *     @arg EXTI_Linex: External interrupt line x where x(0..19)
  * @retval The new state of EXTI_Line (SET or RESET).
  */
ITStatus EXTI_GetITStatus(uint32_t EXTI_Line)

//Libraries\STM32F10x_StdPeriph_Driver_v3.5\src\stm32f10x_exti.c
```

#### GPIO_ReadInputDataBit()

GPIO_ReadInputDataBit() 함수에 대한 설명은 아래와 같다.
```c
/**
  * @brief  Reads the specified input port pin.
  * @param  GPIOx: where x can be (A..G) to select the GPIO peripheral.
  * @param  GPIO_Pin:  specifies the port bit to read.
  *   This parameter can be GPIO_Pin_x where x can be (0..15).
  * @retval The input port pin value.
  */
uint8_t GPIO_ReadInputDataBit(GPIO_TypeDef* GPIOx, uint16_t GPIO_Pin)

//Libraries\STM32F10x_StdPeriph_Driver_v3.5\src\stm32f10x_gpio.c
```

#### EXTI_ClearITPendingBit()

EXTI_ClearITPendingBit() 함수에 대한 설명은 아래와 같다.

```c
/**
  * @brief  Clears the EXTI's line pending bits.
  * @param  EXTI_Line: specifies the EXTI lines to clear.
  *   This parameter can be any combination of EXTI_Linex where x can be (0..19).
  * @retval None
  */
void EXTI_ClearITPendingBit(uint32_t EXTI_Line)

//Libraries\STM32F10x_StdPeriph_Driver_v3.5\src\stm32f10x_exti.c
```

#### sendDataUART1

sendDataUART1은 stm32f10x_usart.c에 정의된 USART_SendData()를 이용하여 만든 함수이다(미리 주어진 템플릿코드에서 주어져있었음).

```c
void sendDataUART1(uint16_t data) {
   /* Wait till TC is set */
   while ((USART1->SR & USART_SR_TC) == 0);
   USART_SendData(USART1, data);
}
```

USART_SendData()는 아래와 같이 정의되어있다.

```c
/**
  * @brief  Transmits single data through the USARTx peripheral.
  * @param  USARTx: Select the USART or the UART peripheral. 
  *   This parameter can be one of the following values:
  *   USART1, USART2, USART3, UART4 or UART5.
  * @param  Data: the data to transmit.
  * @retval None
  */
void USART_SendData(USART_TypeDef* USARTx, uint16_t Data)
```

## main()에서 수행할 무한루프문 작성

```c
while (1) {
       // TODO: implement 
       GPIOD->BSRR = GPIO_BSRR_BS2 | GPIO_BSRR_BS3 | GPIO_BSRR_BS4 | GPIO_BSRR_BS7;
        switch(index) {
            case 0: { GPIOD->BSRR = GPIO_BSRR_BR2; break;}
            case 1: { GPIOD->BSRR = GPIO_BSRR_BR3; break;}
            case 2: { GPIOD->BSRR = GPIO_BSRR_BR4; break;}
            case 3: { GPIOD->BSRR = GPIO_BSRR_BR7; break;}
        }
        if (direction == 1) {
            index ++;
            if (index == 4) {index = 0;}
        }
        else {
            index --;
            if (index == -1) {index = 3;}
        }

       Delay();
    }
```
먼저 GPIOD의 BSRR 레지스터를 사용하여 PD2, PD3, PD4, PD7의 레지스터를 set한다(회로 구조상, bit를 set하면 LED가 꺼지고, bit를 reset하면 LED가 켜진다).

그리고 index에 따라 PD2, PD3, PD4, PD7 LED를 켠다. direction이 1일 때는 index가 증가하여 PD2 -PD3 -PD4 -PD7 순으로 LED를 켜고, direction이 -1일 때는 index가 감소하여 PD7-PD4-PD3-PD2 순으로 LED를 켠다.

### 참고

#### GPIOD and GPIO_TypeDef

```c
#define GPIOD               ((GPIO_TypeDef *) GPIOD_BASE)

//Libraries\CMSIS\DeviceSupport\stm32f10x.h
```

```c
typedef struct
{
  __IO uint32_t CRL;
  __IO uint32_t CRH;
  __IO uint32_t IDR;
  __IO uint32_t ODR;
  __IO uint32_t BSRR;
  __IO uint32_t BRR;
  __IO uint32_t LCKR;
} GPIO_TypeDef;
//Libraries\CMSIS\DeviceSupport\stm32f10x.h
```

```c
/******************  Bit definition for GPIO_BSRR register  *******************/
#define GPIO_BSRR_BS0                        ((uint32_t)0x00000001)        /*!< Port x Set bit 0 */
#define GPIO_BSRR_BS1                        ((uint32_t)0x00000002)        /*!< Port x Set bit 1 */
#define GPIO_BSRR_BS2                        ((uint32_t)0x00000004)        /*!< Port x Set bit 2 */
#define GPIO_BSRR_BS3                        ((uint32_t)0x00000008)        /*!< Port x Set bit 3 */
#define GPIO_BSRR_BS4                        ((uint32_t)0x00000010)        /*!< Port x Set bit 4 */
#define GPIO_BSRR_BS5                        ((uint32_t)0x00000020)        /*!< Port x Set bit 5 */
#define GPIO_BSRR_BS6                        ((uint32_t)0x00000040)        /*!< Port x Set bit 6 */
#define GPIO_BSRR_BS7                        ((uint32_t)0x00000080)        /*!< Port x Set bit 7 */
#define GPIO_BSRR_BS8                        ((uint32_t)0x00000100)        /*!< Port x Set bit 8 */
#define GPIO_BSRR_BS9                        ((uint32_t)0x00000200)        /*!< Port x Set bit 9 */
#define GPIO_BSRR_BS10                       ((uint32_t)0x00000400)        /*!< Port x Set bit 10 */
#define GPIO_BSRR_BS11                       ((uint32_t)0x00000800)        /*!< Port x Set bit 11 */
#define GPIO_BSRR_BS12                       ((uint32_t)0x00001000)        /*!< Port x Set bit 12 */
#define GPIO_BSRR_BS13                       ((uint32_t)0x00002000)        /*!< Port x Set bit 13 */
#define GPIO_BSRR_BS14                       ((uint32_t)0x00004000)        /*!< Port x Set bit 14 */
#define GPIO_BSRR_BS15                       ((uint32_t)0x00008000)        /*!< Port x Set bit 15 */

#define GPIO_BSRR_BR0                        ((uint32_t)0x00010000)        /*!< Port x Reset bit 0 */
#define GPIO_BSRR_BR1                        ((uint32_t)0x00020000)        /*!< Port x Reset bit 1 */
#define GPIO_BSRR_BR2                        ((uint32_t)0x00040000)        /*!< Port x Reset bit 2 */
#define GPIO_BSRR_BR3                        ((uint32_t)0x00080000)        /*!< Port x Reset bit 3 */
#define GPIO_BSRR_BR4                        ((uint32_t)0x00100000)        /*!< Port x Reset bit 4 */
#define GPIO_BSRR_BR5                        ((uint32_t)0x00200000)        /*!< Port x Reset bit 5 */
#define GPIO_BSRR_BR6                        ((uint32_t)0x00400000)        /*!< Port x Reset bit 6 */
#define GPIO_BSRR_BR7                        ((uint32_t)0x00800000)        /*!< Port x Reset bit 7 */
#define GPIO_BSRR_BR8                        ((uint32_t)0x01000000)        /*!< Port x Reset bit 8 */
#define GPIO_BSRR_BR9                        ((uint32_t)0x02000000)        /*!< Port x Reset bit 9 */
#define GPIO_BSRR_BR10                       ((uint32_t)0x04000000)        /*!< Port x Reset bit 10 */
#define GPIO_BSRR_BR11                       ((uint32_t)0x08000000)        /*!< Port x Reset bit 11 */
#define GPIO_BSRR_BR12                       ((uint32_t)0x10000000)        /*!< Port x Reset bit 12 */
#define GPIO_BSRR_BR13                       ((uint32_t)0x20000000)        /*!< Port x Reset bit 13 */
#define GPIO_BSRR_BR14                       ((uint32_t)0x40000000)        /*!< Port x Reset bit 14 */
#define GPIO_BSRR_BR15                       ((uint32_t)0x80000000)        /*!< Port x Reset bit 15 */

//Libraries\CMSIS\DeviceSupport\stm32f10x.h
```

#### Delay()

Delay() 함수는 주어진 템플릿코드에 아래와 같이 정의되어있다.

```c
void Delay(void) {
   int i;

   for (i = 0; i < 2000000; i++) {}
}

```

# 실험 결과

{% include video id="-UPOpsq6Big" provider="youtube" %}

- 시작 시 LED가 정방향으로 점등을 반복 (1 -> 2 -> 3 -> 4 -> 1)
- KEY2를 누르면 LED가 역방향으로 변경 (1 -> 4 -> 3 -> 2 -> 1)
- KEY1을 누르면 LED가 정방향으로 변경
- KEY3을 누르면 PUTTY에 “Hello Team05”가 한 줄씩 출력
- PUTTY에 'b' 입력 시 LED가 역방향으로 변경

# 알게 된 점
구조체를 사용하면 레지스터 설정을 더 편리하게 할 수 있음을 알게 되었으며, STM32F107 보드에서 인터럽트를 활용하여 외부 입력을 처리하는 방법을 알게 되었다.

# 참고 문헌
- STM32F107 Datasheet 
[ https://www.st.com/resource/en/datasheet/stm32f107vc.pdf ]( https://www.st.com/resource/en/datasheet/stm32f107vc.pdf )
- STM32F107 Reference Manual 
[https://www.st.com/resource/en/reference_manual/rm0008-stm32f101xxstm32f102xxstm32f103xx-stm32f105xx-and-stm32f107xx-advanced-armbased-32bitmcusstmicroelectronics.pdf ](https://www.st.com/resource/en/reference_manual/rm0008-stm32f101xxstm32f102xxstm32f103xx-stm32f105xx-and-stm32f107xx-advanced-armbased-32bitmcusstmicroelectronics.pdf )
- STM32F107VCT6 schematic
