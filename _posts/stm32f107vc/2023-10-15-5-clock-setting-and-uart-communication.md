---
title: "clock setting and UART communication on stm32f107vc"
last_modified_at: 2023-10-15T23:53:12+09:00
categories:
    - stm32f107vc
tags:
    - stm32f107vc
    - embedded-system

toc: true
toc_label: "My Table of Contents"
author_profile: true

---
# 목표
-	라이브러리를 활용하여 코드 작성
-	Clock Tree의 이해 및 사용자 Clock 설정
-	오실로스코프를 이용한 clock 확인
-	UART 통신의 원리를 배우고 실제 설정 방법 파악

# 실험 원리
## 라이브러리를 활용한 코드 작성
주소와 값을 일일이 입력하지 않고, stm32f10x.h 등의 라이브러리 파일에서 정의된 상수와 구조체를 이용하여 코드를 작성할 수 있다. 예를 들어 아래 코드는

```c
(*(volatile unsigned int*)0x40021018) |= 0x04;
```
아래와 같이 라이브러리에 정의된 상수와 구조체를 이용하여 아래와 같이 변경할 수 있다.

```c
RCC->APB2ENR |=RCC_APB2ENR_IOPAEN;
```
![그림 1 RCC_TypeDef 구조체 일부. RCC_TypeDef 구조체는 RCC의 구조를 정의한 구조체이다.](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/7101fdb1-b195-4843-a6a1-44d357f0ecff)

(그림 1 RCC_TypeDef 구조체 일부. RCC_TypeDef 구조체는 RCC의 구조를 정의한 구조체이다.)

![그림 2 RCC_APB2ENR(APB2 peripheral clock enable register)를 설정하는 상수 정의의 일부](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/cf7f7eca-1080-4bc9-a377-278ca0c40d7c)

(그림 2 RCC_APB2ENR(APB2 peripheral clock enable register)를 설정하는 상수 정의의 일부)

IAR Embedded Workbench에서는, 주어진 매크로 상수에 오른쪽 마우스를 클릭하고 “go to Definition …”을 클릭하면 해당 상수가 정의된 곳으로 이동할 수 있다.

코드에서 정의된 매크로 상수를 어느정도 입력 후 ctrl+spacebar 또는 ctrl+alt+spacebar를 누르면 정의된 이름들로 바로 바뀐다.

## Clock

### HSI(High-Speed Internal) clock

![그림 3 Clock tree에서 HSI clock의 위치와 주변 구조](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/b260fefe-a5a2-4a72-aba9-b5747d77cd01)

(그림 3 Clock tree에서 HSI clock의 위치와 주변 구조)

HSI Clock은 8Mhz RC 오실레이터에서 생성된다. 그림3에서 알 수 있듯, HSI에서생성된 클럭은 바로 시스템 클럭으로 사용하거나 여러 계산을 거쳐 PLLCLK로 쓸 수 있다. 이때 그림3에서, PLLCLK = HSI RC / 2 * PLLMUL (PLLSRC MUX에서 HSI RC/2가 선택되었을 때)이다.

### HSE(High-Speed External) clock

![그림 4 HSE Clock의 위치와 주변 구조](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/700181fb-d144-4e60-9803-0ae1998c11f7)

(그림 4 HSE Clock의 위치와 주변 구조)

![그림 5 STM32F107 개발보드에서 OSC_IN, OSC_OUT으로 25MHZ 클럭이 들어가는 것을 알 수 있다.](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/80dfd72e-6b10-466f-a502-fd7707280b46)

(그림 5 STM32F107 개발보드에서 OSC_IN, OSC_OUT으로 25MHZ 클럭이 들어가는 것을 알 수 있다.)

HSE OSC에서 25MHz 클럭이 생성되며, 생성된 클럭은 바로 시스템 클럭으로 사용하거나 여러 계산과 mux를 거쳐 PLLCLK로 사용할 수 있다.

PLLCLK = HSE OSC / PREDIV1 * PLLMUL (PREDIV1SRC가 HSE로 선택되고, PLLSRC가 PREDIV1으로 선택되었을 때)

PLLCLK = HSE OSC/ PREDIV2 * PLL2MUL / PREDIV1 * PLLMUL (PREDIV1SRC가 PLL2MUL로 선택되고, PLLSRC가 PREDIV1으로 선택되었을 때)

### Clock Tree

![그림 6 시스템클럭이 설정되는 방법과 시스템클럭이 사용되는 곳](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/5a047cb2-0d67-460b-9dd5-0ddd9416ddc0)

(그림 6 시스템클럭이 설정되는 방법과 시스템클럭이 사용되는 곳)

HSI, HSE, PLL 중 SW MUX에 의해 시스템 클럭(최대 72MHz)로 설정될 수 있다. 이 시스템 클럭은 AHB prescaler와 APB1 prescaler, APB2 prescaler 를 통해 APB1, APB2로 전달된다.

또한 시스템 클럭은 MCO MUX를 통해 MCO에 출력해 오실로스코프로 확인할 수 있다.

![그림7 시스템 클럭이 AHB prescaler, APB1 prescaler, APB2 prescaler를 통해 APB1, APB2로 전달됨](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/99ed13e0-7aa0-4794-a037-a7ea0cbafd21)

(그림7 시스템 클럭이 AHB prescaler, APB1 prescaler, APB2 prescaler를 통해 APB1, APB2로 전달됨)

시스템 클럭을 APB1 prescaler를 이용해 최대 36MHz 클럭을 PCLK1에 제공 가능하다. APB2 prescaler를 이용해 최대 72MHz 클럭을 PCLK2에 제공 가능하다(공급되는 클럭이 최대 클럭을 넘지 않도록 해야한다).

![그림 8 HCLK, FCLK, PCLK](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/50b206e4-eabf-4a5c-850d-06e1baa168a3)

(그림 8 HCLK, FCLK, PCLK)

HCLK는 DMA, Core memory, AHB bus에서 사용되는 클럭이고 FCLK는 CPU에서 사용되는 클럭이다. PCLK는 APB Bus에서 사용되는 Clock이다.

### PLL(Phase-Locked Loop)

![그림 9 PLL](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/47697df7-3602-4525-9ea4-0180fc3e049f)

(그림 9 PLL)

PLL은 위상 동기 회로이며, 입력 신호와 (입력되는)출력 신호를 이용해 출력 신호를 제어하는 시스템을 말한다. 즉 입력된 신호에 맞추어 출력 신호의 주파수 조절이 목적이다. 앞서 살펴보았듯이, MUX와 PREDIVx와 PLLxMUL을 이용해 HSE 오실레이터 Clock(25MHz)를 원하는 시스템 클럭 주파수로 만든다.

### Clock 제어에 사용되는 여러가지 레지스터

Clock 제어에는 RCC_CFGR(Clock configuration register), RCC_CFGR2(Clock configuration register2), RCC_CR(Clock control regsiter) 레지스터가 쓰인다.

RCC_CFGR에서는 MCO source, PLLMUL, APB2 prescaler, APB1 prescaler, AHB prescaler, System clock source등을 정할 수 있다.

RCC_CFGR2에서는 PLL3MUL, PLL2MUL, PREDIV2, PREDIV1을 설정할 수 있다.

RCC_CR에서는 PLL 활성화, HSE 활성화, HSI 활성화 등을 설정할 수 있다.

예를 들어, MCO source를 시스템 클럭으로 사용하고 싶으면 라이브러리를 이용해 
```c
RCC->CFGR &= ~(uint32_t)RCC_CFGR_MCO
```
를 통해 CFGR의 MCO 설정 부분을 초기화한 후, 
```c
RCC->CFGR |=(uint32_t)RCC_CFGR_MCO_SYSCLK
```
으로 설정하여 MCO source를 시스템 클럭으로 사용할 수 있다.

## 시리얼 통신(Serial Communication)
병렬 통신(Parallel Communication)은 여러 개의 데이터 선을 이용해, 각각의 선에 비트를 하나씩 보내는 방법이다. 그러나 직렬 통신(Serial Communication)은 하나의 데이터 선을 이용해 비트를 차례로 보내는 방법으로, 속도는 상대적으로 느리지만, 데이터선을 연장하기 위한 비용이 적다.

### Universal Sync/Async Receiver/Transmitter (USART)

동기식 통신을 지원하는 UART로, 데이터 동기화를 위해 같은 클럭을 사용한다. 클럭 전송을 위한 별도의 클럭 선이 필요하고, 동기식이기 때문에 start bit/stop bit가 없고, data를 동기화하여 전송한다.

### Universal Asynchronous Recevier/Transmitter (UART)

비동기 통신 프로토콜로, Rx와 Tx가 교차 연결된다. 비동기 통신이기 때문에 Baud rate를 일치시켜야한다. 

UART의 데이터 패킷은 start bit, data bit, parity bit, stop bit로 구성되어있다. Start bit는 통신의 시작을 나타내며, data bit는 실제 데이터가 들어있으며, parity bit는 에러 검사를 위한 값이고, stop bit는 통신의 종료를 알려주는 bit이다.

Parity bit는 오류로 인해 비트가 1에서 0으로 또는 0에서 1로 바뀌었을 때 확인할 수 있는 보호장치이다. 에러 발생 여부는 확인 가능하지만, 어느 위치에서 에러가 발생하였는지는 확인이 불가능하므로, 수정은 불가능하다. 따라서 수신측에서 에러 발생을 확인하면 데이터를 재요청한다.

Even parity는 parity bit를 데이터 비트 + parity bit에서 1의 개수를 짝수 개가 되게 맞추며, Odd parity는 parity bit를 데이터 비트 + parity bit에서 1의 개수를 홀수 개가 되게 맞춘다.

# 실험 방법
SYSCLK은 28MHz, PCLK2는 14MHz, Baud Rate는 28800으로 설정한다.

(1)	Clock configuration register(RCC_CFGR) 레지스터 및 Clock configuration register2(RCC_CFGR2)를 사용하여, AHB prescaler, APB1 prescaler, APB2 prescaler를 설정하고, PLLSRC, PLLMULL, PREDIV2, PLL2MUL, PREDIV1SRC, PREDIV1을 설정한다.

(2)	MCO의 source를 설정한다.

(3)	APB2 peripheral clock enable register(RCC_APB2ENR)을 이용해 필요한 포트(MCO, 버튼 및 UART 관련 포트)를 활성화한다.

(4)	Port configuration register low/high(GPIOx_CRL/CRH)를 이용해 각 포트를 설정(input/output 등)

(5)	Control register1(USART_CR1)을 사용하여 UART word length를 8bit로 설정한다.

(6)	Control register1(USART_CR1)을 사용하여 UART 패리티 비트는 없음으로 설정한다.

(7)	Control register1(USART_CR1)을 사용하여 UART Tx Rx를 활성화한다.

(8)	Control register2(USART_CR2)을 사용하여 Stop bit를 1bit로 설정한다.

(9)	Control register3(USART_CR3)을 사용하여 USART CTS와 RTS를 비활성화한다
(CTS, RTS가 활성화되어있으면, UARTx_CTS, UARTx_RTS 핀이 입력될 때만 데이터를 송수신한다)

(10) Baud rate register(USART_BRR)를 사용하여 설정해야할 Baud rate에 맞는 DIV_Mantissa와 DIV_Fraction을 설정한다.

![그림 10 Baud rate를 설정하는 방법](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/6ddcbcae-f0ac-4dbc-98ad-a89c16a3a39d)

(그림 10 Baud rate를 설정하는 방법)

이때 Baud rate를 설정하는 방법은 다음과 같다. PCLK1 또는 PCLK2인 f_ck를 16*USARTDIV로 나눈 값이 Baud Rate이다. 이때 USARTDIV는 USART_BRR 레지스터에서 설정하는 값인 DIV_Mantissa, DIV_Fraction을 이용하여 다음과 같이 정의된다.

$$ USARTDIV = Mantissa + \frac{Fraction}{16}$$

# 실험 과정

## PCLK2를 시스템 클럭의 1/2로 설정(14MHz)하고, 시스템 클럭을 HSE 클럭(25MHz)과 Mux, PREDIVx, PLLxMUL을 이용하여 28MHz로 설정하기

```c
//@TODO - 1 Set the clock
        /* HCLK = SYSCLK */
        RCC->CFGR |= (uint32_t)RCC_CFGR_HPRE_DIV1; // set AHB prescaler to SYSCLK/1
        /* PCLK2 = HCLK / ?, use PPRE2 */
         RCC->CFGR |= (uint32_t)RCC_CFGR_PPRE2_DIV2;// set APB2 prescaler to (SYSCLK via AHB prescaler) /2
        /* PCLK1 = HCLK */
        RCC->CFGR |= (uint32_t)RCC_CFGR_PPRE1_DIV1; // set APB1 prescaler to SYSCLK / 1
```

먼저 PCLK2를 시스템 클럭의 1/2로 설정한다. SYSCLK이 PCLK2로 계산되는 과정에서 AHB prescaler와 APB2 prescaler를 둘다 거쳐가므로, RCC_CFGR_HPRE_DIV1을 통해 AHB prescaler는 /1로 설정하고, RCC_CFGR_PPRE2_DIV2를 통해 APB2 prescaler는 /2로 설정한다. 그러면 최종적으로 PCLK2 = SYSCLK/1/2 = SYSCLK/2이다. PCLK1은 기본 SYSCLK으로 인가한다.

![SYSCLK은 AHB prescaler와 APB2 prescaler를 통해 PCLK2 주파수로 인가된다.](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/5adbab24-30d3-4619-968f-20b8a67e5634)

(그림 11 SYSCLK은 AHB prescaler와 APB2 prescaler를 통해 PCLK2 주파수로 인가된다.)

그리고, HSE clock은 25MHz이고 시스템 클럭을 28MHz로 설정해야하므로 시스템 클럭을 다음과 같이 계산하여 HSE 클럭으로부터 추출할 수 있다.

$$ 28=25/5 \times 8 / 10 \times 7 $$

이를 clock tree에서 찾으면 다음과 같다.

![Clock tree에서 HSE 클럭을 28MHz 시스템 클럭으로 만들기 위해 필요한 루트](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/b3a39442-b546-47cd-935f-88b13ed3cc55)

(그림 12 Clock tree에서 HSE 클럭을 28MHz 시스템 클럭으로 만들기 위해 필요한 루트)

그림 12에서 알 수 있듯, 25를 28로 만들기 위해 PREDIV2를 /5로 선택해 5로 나누고, PLL2MUL에서 x8을 선택해 8을 곱하고, PLL2MUL을 PREDIV1의 소스로 선택한다. 그런다음 PREDIV1을 /10을 선택해 10으로 나누고, PLLSRC를 PREDIV1으로 선택하고, 마지막으로 PLLMUL을 x7로 하여 7을 곱한다.

이를 코드로 구현하면 다음과 같다

```c
/* Configure PLLs ------------------------------------------------------*/
        //Default Hz : 25MHz
        RCC->CFGR &= (uint32_t)~(RCC_CFGR_PLLSRC | RCC_CFGR_PLLMULL);
        RCC->CFGR |= (uint32_t)(RCC_CFGR_PLLSRC_PREDIV1 | RCC_CFGR_PLLMULL7);
        //PREDIV1 selected as PLLSRC, and x7 selected for PLLMUL

        RCC->CFGR2 &= (uint32_t)~(RCC_CFGR2_PREDIV2 | RCC_CFGR2_PLL2MUL | RCC_CFGR2_PREDIV1 | RCC_CFGR2_PREDIV1SRC);
        RCC->CFGR2 |= (uint32_t)(RCC_CFGR2_PREDIV2_DIV5 | RCC_CFGR2_PLL2MUL8 | RCC_CFGR2_PREDIV1SRC_PLL2 | RCC_CFGR2_PREDIV1_DIV10);
        // /5 selected for PREDIV2, x8 selected for PLL2MUL, PLL2MUL selected as PREDIV1SRC(PREDIV1 source), /10 selected for PREDIV1
//@End of TODO - 1
```

Clock configuration register를 이용해 먼저 PLLSRC MUX를 PREDIV1으로 선택(RCC_CFGR_PLLSRC_PREDIV1)하고, PLLMUL을 x7로 설정(RCC_CFGR_PLLMULL7)한다.

Clock Configuration register2를 이용해 먼저 PREDIV2를 /5로 설정(RCC_CFGR_PREDIV2_DIV5)하고 PLL2MUL을 x8로 설정(RCC_CFGR_PLL2MUL8)한다. 그리고 PREDIV1SRC MUX를 PLL2MUL로 설정(RCC_CFGR2_PREDIV1SRC_PLL2)하고, PREDIV1을 /10으로 설정(RCC_CFGR2_PREDIV1_DIV10)으로 설정한다.

## MCO의 소스를 SYSCLK(시스템 클럭)으로 지정하기

```c
//@TODO - 2 Set the MCO port for system clock output
        RCC->CFGR &= ~(uint32_t)RCC_CFGR_MCO;
        RCC->CFGR |= (uint32_t)RCC_CFGR_MCO_SYSCLK;  
        //using clock configuration register, system clock selected as MCO source        
//@End of TODO - 2
```

Clock configuration register를 이용하여, 시스템 클럭을 MCO 소스로 지정(RCC_CFGR_MCO_SYSCLK)하여, MCO(PA8)핀을 통해 주파수를 관찰할 수 있도록 한다.

## (3)	APB2 peripheral clock enable register(RCC_APB2ENR)을 이용해 필요한 포트(MCO, 버튼 및 UART 관련 포트, 버튼 포트)를 활성화.

```c
void RCC_Enable(void) {
//@TODO - 3 RCC Setting
    /*---------------------------- RCC Configuration -----------------------------*/
    /* GPIO RCC Enable  */
    /* UART Tx, Rx, MCO port */
    // RCC->APB2ENR |= ??
    RCC->APB2ENR |= ( RCC_APB2ENR_IOPAEN | RCC_APB2ENR_AFIOEN );
    //for MCO(PA8), USART_TX(PA9), USART_RX(PA10), enable Pin A
    //for alternative Fuction I/O, alternative function I/O clock enable
    /* USART RCC Enable */
    // RCC->APB2ENR |= ??
    RCC->APB2ENR |= RCC_APB2ENR_USART1EN;
    //for USART, USART1 clock enable
    /* User S1 Button RCC Enable */
    //Use Key1 (PC4)
    // RCC->APB2ENR |= ??
    RCC->APB2ENR |= RCC_APB2ENR_IOPCEN;
    //for S1 button(PC4), PIN C clock enable
}
```
MCO(PA8), USART_TX(PA9), USART_RX(PA10)을 사용하기 위해, Pin A를 활성화한다(RCC_APB2ENR_IOPAEN). 그리고 alternative function I/O를 사용하기 위해 alternative function I/O에 clock을 활성화(RCC_APB2ENR_AFIOEN)한다.

그리고 USART1에 클럭을 인가한다(RCC_APB2ENR_USART1EN). 마지막으로, S1 버튼이 있는 포트C를 활성화시킨다(RCC_APB2ENR_IOPCEN).

![MCO, USART1_TX, USART1_RX](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/cad8c79f-7245-4d16-a312-816ad333febb)

![S1버튼 핀맵](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/5efd4b4a-8033-4cbd-a602-c01c05e3b6eb)

(그림 13 MCO, USART1_TX, USART1_RX 및 S1버튼 핀맵)

## Port configuration register를 이용하여 각 포트 설정

![Port configuration register](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/41eb208a-9b5a-4fe1-a240-1fab8c79805f)

(그림 14 Port configuration register)

MCO(PA8)와 USART TX(PA9)는 alternative function output push-pull max 50MHz로 설정하고, USART RX(PA10)은 input with pull-up/pull down으로 설정해야한다. S1버튼(PC4)은 General purose output push-pull, max speed 2MHz로 설정한다. 즉 다음과 같이 설정해야한다.

```c
void PortConfiguration(void) {
//@TODO - 4 GPIO Configuration
    /* Reset(Clear) Port A CRH - MCO, USART1 TX,RX*/
    GPIOA->CRH &= ~(
        (GPIO_CRH_CNF8 | GPIO_CRH_MODE8) |
        (GPIO_CRH_CNF9 | GPIO_CRH_MODE9) |
        (GPIO_CRH_CNF10 | GPIO_CRH_MODE10)
    );
    /* MCO Pin Configuration */
    //TX : PA09, //RX : PA10
    GPIOA->CRH |= (GPIO_CRH_CNF8_1 | GPIO_CRH_MODE8_1 | GPIO_CRH_MODE8_0);
    // set MCO Pin(PA8) to output mode max 50MHz(using MODE 11 ), and alternative input/output mode(using CNF 10)
    /* USART Pin Configuration */
    GPIOA->CRH |= ( GPIO_CRH_MODE9_1 |GPIO_CRH_MODE9_0| GPIO_CRH_CNF9_1 ) | GPIO_CRH_CNF10_1;
    //set USART TX(PA9) to output mode max 50MHz(using MODE 11), and alternative input/output mode(using CNF 10)
    //set USART RX(PA10) to input mode(using MODE 00), and input with pull-up/pull-down(using CNF 10)
    /* Reset(Clear) Port C CRH - User S1 Button */
    //Key 4
    GPIOC->CRL &= ~( GPIO_CRL_MODE4 | GPIO_CRL_CNF4 );
    // GPIOC->CRH &= ??
    /* User S1 Button Configuration */
    GPIOC->CRL |= GPIO_CRL_CNF4_1;
    //set S1 button(PC4) to input mode(using MODE 00), and input with pull-up/pull-down(using CNF 10)
}
```

MCO(PA8)와 USART TX(PA9)은 MODE 11, CNF 10으로 설정하고, USART TX(PA10)은 MODE 00, CNF 10으로 설정한다. 마지막으로, S1 button(PC4)는 MODE 00, CNF 10으로 설정한다. 이때 설정할 비트만 GPIO_CRy_MODEx_z와 GPIO_CRy_CNFx_z를 이용하여 설정한다(y=L or H, y=0..7(L) 8..15(H), z=0 or 1. 예를 들어, CRH 8번핀의 MODE 0번째 비트를 set하고싶으면, GPIO_CRH_MODE8_0 매크로 상수를 통해 설정할 수 있다).

## Word length와 parity bit 설정

미리 주어진 코드에 의해 word length 설정 비트 및 parity enable 설정 비트가 0으로 초기화되어있다. 구체적으로, 

```c
USART->CR1 &= ~(uint32_t)(USART_CR1_M|USART_CR1_PCE, USART_CR1_PS|USART-CR1_TE|USART_CR1_RE)
```
에 의해 word length설정, 패리티 활성화 설정, even/odd 패리티 설정, TX 활성화 설정, RX 활성화 설정이 모두 0으로 초기화 되어있다.

![Word length configuration in Control register(USART_CR1)](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/87becd26-46f7-4f2e-9efe-c95a40c791be)

(그림 15 Word length configuration in Control register(USART_CR1))

Word length 8bit를 설정하기 위해서는 해당 비트를 0으로 설정하면 된다. 그런데 주어진 코드에서 이미 0으로 초기화되어있으므로, 따로 설정하지 않는다.

![Parity control enable/disable bit in register(USART_CR1)](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/ac38a094-cab8-498a-8a4c-828aa728080a)

(그림 17 Parity control enable/disable bit in register(USART_CR1))

Parity bit를 비활성화하기 위해 해당 비트를 0으로 설정하면 된다. 그런데 주어진 코드에서 이미 0으로 초기화되어 있으므로, 따로 설정하지 않는다.

## TX RX 활성화

```c
//@TODO - 8: Enable Tx and Rx
    USART1->CR1 |= ( USART_CR1_TE | USART_CR1_RE );
```

USART_CR1_TE, USART_CR1_RE 매크로 상수를 이용해 TX와 RX를 활성화시킨다.

## Stop bit를 1bit로 설정 및 CTS, RTS 비활성화

![STOP bit configuration bits in register(USART_CR2)](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/c7a95d11-72a8-421f-9782-75a422158917)

(그림 18 STOP bit configuration bits in register(USART_CR2))

stop bit 설정 비트가 0으로 초기화되어있다. 구체적으로,
```c
USART1->CR2 &=~(uint32_t)(USART_CR2_STOP)
```
에 의해 0으로 초기화 되어있다. Stop bit를 1bit로 설정하기 위해서는 00을 넣으면 되는데 이밎 초기화되어있으므로, 따로 설정하지 않는다

![CTS enable/disable bit in register(USART_CR3)](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/0edc8144-6eaf-43c7-9680-664388eaf5d5)

![RTS enable/disable bit in register(USART_CR3)](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/13851264-8d10-46a5-9661-82783e04bfc5)

(그림 19 CTS, RTS enable/disable bit in register(USART_CR3))

마찬가지로 주어진 코드에서 CTSE와 RTSE 비트가 0으로 초기화되어 있고 CTS와 RTS를 disable하기 위해서는 해당 비트를 0으로 하면 되므로, 따로 설정할 필요는 없다.

## USARTDIV, Mantissa, Fraction 계산 및 USART_BRR 설정
PCLK2는 14MHz(=14000000Hz)이다. Baud Rate를 28800으로 맞추기 위해서는 USARTDIV가 다음과 같이 되어야 한다.

$$ USARTDIV = \frac{14000000}{16\times 28800} \approx 30 \frac{6}{16}$$

따라서 mantissa는 30, fraction은 6이 되어야 한다. 즉  

DIV_Mantissa[11:0]=0b00000011110, DIV_Fraction[3:0]=0b0110이 되어야하고 둘을 합치면 0b000000111100110 = 0b0001 1110 0110 = 0x1E6 이 된다. 따라서 아래와 같이 코드를 작성한다.

```c
//@TODO - 11: Calculate & configure BRR
    // USART1->BRR |= ??
    // PCLK2 / ( 16 * Baud Rate) = 14MHz / ( 16 * 28800 ) = 30.382

    // 0d30 -> 0x1E -> 16 * 1 + 14
    // 16 * 0d0.382 -> 0d6.112 -> ???? -> 0d6 -> 0x6
    //USARTDIV = 30 + 6/16
    // USART_BRR: 0x1E6
    USART1->BRR |= 0x1E6;
```

![Baud rate register](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/f3ababdc-c39f-425e-b9e8-2482bbd09232)

(그림 20 Baud rate register)

## UART 활성화
USART1_CR1 레지스터에 있는 13번 bit를 활용하여 USART를 활성화한다.

```c
//@TODO - 12: Enable UART (UE)
    USART1->CR1 |= USART_CR1_UE;
```

코드에서는 13번째 bit가 1인 매크로 상수 USART_CR1_UE를 활용하였다. 

## 버튼이 눌렸을 때 메시지 보내기

```c
 // if you need, init pin values here
    SendData('\0');
    // first msg denined therefore, insert null character to show full text of msg
    while (1) {
        //@TODO - 13: Send the message when button is pressed
        if(!(GPIOC->IDR & GPIO_IDR_IDR4)) { // if button is pressed
            printf("p\n");
            for(i = 0; msg[i] != '\0'; i++){
                SendData(msg[i]);//send data
            }
            delay();
        }
    }
```

GPIOC_IDR 레지스터에서, PC4의 입력이 0이면, GPIO_IDR_IDR4와 and 했을 때 0이 된다. 이것을 계속확인하여 0이 나오면 SendData함수를 통해 msg에 null문자가 나올 때까지 USART1_DR 레지스터(USART data register)를 통해 보내게 된다.

# 실험 결과
![버튼을 눌렀을 때 출력. Hello Team05가 출력된다.](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/ab97d048-37d6-4080-a93a-8cee0114c717)

(그림 21 버튼을 눌렀을 때 출력. Hello Team05가 출력된다.)

![MCO로 출력되는 시스템클럭. 설정한 28MHz로 출력되는 것을 알 수 있다.](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/05165b10-ebcf-485b-a77a-7cac5b1ac3c5)

(그림 22 MCO로 출력되는 시스템클럭. 설정한 28MHz로 출력되는 것을 알 수 있다.)

# 알게 된 점
미리 정의된 매크로 상수를 이용해 좀더 편리하게 코드를 작성하고 디버깅할 수 있다는 것을 알게 되었으며, clock tree와 HSE를 활용하여 원하는 클럭을 만들어내고, 이를 이용해 UART 통신을 하는 방법을 알게 되었다.

# 참고 문헌
-	STM32F107 Datasheet 
https://www.st.com/resource/en/datasheet/stm32f107vc.pdf 
-	STM32F107 Reference Manual 
https://www.st.com/resource/en/reference_manual/rm0008-stm32f101xxstm32f102xx-stm32f103xx-stm32f105xx-and-stm32f107xx-advanced-armbased-32bitmcus-stmicroelectronics.pdf 
-	STM32F107VCT6 schematic
-	https://community.st.com/t5/stm32-mcus-products/what-is-the-function-of-afio-en-bit-in-rcc-apb2enr/td-p/509174
