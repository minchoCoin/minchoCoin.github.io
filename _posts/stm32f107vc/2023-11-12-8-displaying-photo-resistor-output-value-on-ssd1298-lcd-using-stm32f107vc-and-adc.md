---
title: "Displaying photo resistor output value on SSD1298 LCD using stm32f107vc and adc"
last_modified_at: 2023-11-12T00:28:12+09:00
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
-	TFT-LCD의 원리와 동작 방법에 대한 이해
-	TFT-LCD 라이브러리 작성과 이해
-	TFT-LCD Touch 동작 제어
-	ADC 개념 이해
-	조도 센서 사용 방법 학습

# 실험 원리
## LCD
액정이란 액체와 결정의 중간 성질을 가진 화합물로 전압이나 온도에 의해 색이나 투명도가 달라지는 특성을 가지고 있다. 광원에서 나온 빛이 액정을 통과한 후 색 필터를 통과하게 하여 RGB 빛의 점을 만든다. 액정에 전압이 걸리지 않으면 광원에서 나오는 빛이 통과되고, 액정에 전압이 걸리면 빛이 차단된다.

## TFT-LCD
TFT-LCD는 초 박막 액정표시장치로, 액체와 고체의 중간 특성을 가진 액정의 상태 변화와 편광판의 편광 성질을 이용하여 통과하는 빛의 양을 조절함으로써 정보를 표시한다. RGB 픽셀이 유리판에 코딩 되어 컬러 영상을 구현하는 Color Filter와 액정을 제어하기 위해 조박형 유리 기판 위에 반도체 막을 형성한 회로인 TFT 기판, Filter와 기판 사이에 주입된 액정과 광원인 Black light unit으로 구성되어있다.

## Timing Diagram
Timing Diagram은 신호의 동작을 시간에 따라 시각적으로 표현하는 도구로 신호 간의 상대적인 타이밍 관계를 파악하고 시스템의 동작을 분석할 수 있다. Timing Diagram에서 Low에서 High로 올라가는 구간을 Rising Edge, High에서 Low로 떨어지는 구간을 Falling Edge라고 한다. 교차 형태를 취하고 있으면 High/Low 둘 중 하나의 값을 가질 수 있다는 의미이다.

## 실험에서 사용할 SSD1298의 Timing Diagram
![그림 1 LCD에 Write/Read 하기 위해 필요한 Timing Diagram](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/12cc9619-b044-4562-b338-8c6c250ff3be)

(그림 1 LCD에 Write/Read 하기 위해 필요한 Timing Diagram)

CS는 Chip Select로, High에서 Low로 Falling Edge일 때 LCD Chip을 사용한다는 의미이다. D/C는 Data/Command(핀맵에서 RS)로, High로 두고 Data를 전송하고, Low로 두고 Command를 전송한다. WR, RD는 각각 Write 와 Read로, High에서 Low로 Falling Edge일 때, display에 Write/Read한다.

![표 1 SSD1298의 Timing Characteristic](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/e9d401a4-c81d-48c4-b0e1-2bfc7c57e0c0)

(표 1 SSD1298의 Timing Characteristic)

또한 표1에서, 해당 Symbol의 Min Time 내에 Falling/Rising을 해야한다.

따라서 COMMAND를 쓰기 위해서는 D/C를 Low, CS를 LOW, WR를 LOW로 두고 Command를 전송한 후, CS를 High, WR를 High로 다시 되돌려놓는다. DATA를 쓰기 위해서는 D/C를 High, CS를 Low, WR를 Low로 두고 Data를 전송한 후, CS를 High, WR를 High로 다시 되돌려 놓는다.

데이터를 읽기 위해서는 D/C가 High, CS가 LOW, RD 신호가 Low일 때 D0~D17의 데이터를 읽는다.

![그림 2 TFT LCD와 보드 연결 핀맵](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/2ff37638-be81-4fac-8197-528efb93e08c)

(그림 2 TFT LCD와 보드 연결 핀맵)

## ADC(Analog-to-Digital Converter)

아날로그 전기 신호를 디지털 전기신호로 변환하는 전자회로로, 디지털 신호가 아날로그 신호에 비해 데이터의 저장, 조작이 쉽고 신호전송시 잡음에 강하기 때문에 디지털 신호로 변환한다. ADC는 표본화, 양자화, 부호화 총 3단계를 거친다.

### 표본화
시간축 방향에서 연속된 아날로그 신호의 진폭을 일정시간 간격으로 추출한다. 샘플링 주기가 짧을수록 원래의 아날로그 파형에 가까운 데이터를 얻을 수 있으나, 처리할 데이터의 양이 많아진다.

Nyquist-Shannon sampling theorem 은 신호가 대역제한 신호이고, 표본화 주파수가 신호 대역폭의 두 배 초과라면 표본으로부터 연속 시간 기저 대역 신호를 완전히 재구성할 수 있다는 정리이다.

### 양자화
샘플링에서 추출한 연속적인 진폭의 신호를 이산적인 진폭값으로 바꾸는 과정이다. 즉 신호의 크기를 가장 가까운 이산적인 값으로 근사시키는 것으로, 손실이 무조건 발생하는 비가역적 과정이다. 표현할 수 있는 값의 범위를 n bit 형태로 표현하는데, 8bit면 256개, 10bit면 1024개로 신호값을 표시할 수 있다.

### 부호화
양자화된 값들을 2진수 형태로 변환하는 과정으로 0과 1의 비트값으로 인코딩된다.

## DAC(Digital-to-Analog Converter)
디지털 전기 신호를 아날로그 전기신호로 변환하는 전자회로이다. 디지털 형태의 정보를 인간의 눈과 귀로 직접 보거나 들으려면 아날로그 형태로 변환해야한다. 해상도가 높을수록 정밀도가 높아지지만, 처리회로가 복잡해진다.

## 조도 센서
주위의 빛의 밝기를 측정하여 전기신호로 변환해주는 소자로, 황화 카드뮴으로 만들어졌다. 빛이 많이 들어올수록 저항이 작아지고, 적게 들어오면 저항이 증가한다.

## 조도 센서 연결
조도 센서 한쪽 단자는 그라운드에 연결하고, 다른쪽 단자는 저항을 지나 VCC에 연결한다. 조도 센서와 저항 사이를 보드의 핀과 연결하여 전압을 측정한다. 즉 빛이 많이 들어올수록 측정되는 전압이 낮아지고, 빛이 적게 들어올 수록 측정되는 전압이 증가한다.

# 실험 과정
## 조도 센서와 보드 연결
조도 센서 한쪽 단자는 그라운드에 연결하고, 다른쪽 단자는 저항을 지나 VCC에 연결한다. 조도 센서와 저항 사이를 보드의 핀(PC2)과 연결하여 전압을 측정한다.

## LCD 라이브러리 작성
LCD에 Command와 Data를 쓰는 라이브러리를 작성한다. 

Command를 쓰기 위해 그림2의 핀맵을 참조하여, RD를 HIGH, D/C, CS, WR를 LOW로 두고 command를 쓴 다음, WR와 CS를 다시 HIGH로 바꾼다.

Data를 쓰기 위해 그림2의 핀맵을 참조하여, RD와 D/C를 HIGH, CS, WR를 LOW로 두고 data를 쓴 다음, WR과 CS를 다시 HIGH로 바꾼다.

## 코드 작성
1.	GPIOC와 ADC1에 클럭을 인가한다.
2.	조도센서가 연결된 PC2의 핀을 아날로그 입력으로 설정한다.
3.	ADC를 설정한다.
4.	NVIC를 설정하여 ADC 인터럽트를 활성화한다.
5.	ADC 인터럽트가 발생했을 때 처리할 인터럽트 핸들러를 작성한다.
6.	실행하면 LCD에 “THUR_Team05”가 나타나도록 하고, 터치하면 터치된 곳에 작은 원을 그리고 좌표값과 조도 센서 값을 출력하게 한다.

## LCD 터치 조절
코드를 업로드 후, 실행하여 LCD에 십자모양이 나타나면 십자모양 가운데를 눌러 터치 감도를 조절한다.

# LCD 라이브러리 작성
```c
static void LCD_WR_REG(uint16_t LCD_Reg)
{
	// TODO implement using GPIO_ResetBits/GPIO_SetBits
    GPIO_SetBits(GPIOD, GPIO_Pin_15); //RD HIGH
	GPIO_ResetBits(GPIOD, GPIO_Pin_13); // D/C LOW
	GPIO_ResetBits(GPIOC, GPIO_Pin_8); // CS LOW
	GPIO_ResetBits(GPIOB, GPIO_Pin_14); // WR LOW
	
	GPIO_Write(GPIOE, LCD_Reg);
	// TODO implement using GPIO_ResetBits/GPIO_SetBits
	
	GPIO_SetBits(GPIOB, GPIO_Pin_14); // WR HIGH -> Setbits
	GPIO_SetBits(GPIOC, GPIO_Pin_8); // CS HIGH -> Setbits
}

static void LCD_WR_DATA(uint16_t LCD_Data)
{
	// TODO implement using GPIO_ResetBits/GPIO_SetBits
    GPIO_SetBits(GPIOD, GPIO_Pin_15); //RD HIGH
	GPIO_SetBits(GPIOD, GPIO_Pin_13); // D/C HIGH
	GPIO_ResetBits(GPIOC, GPIO_Pin_8); // CS LOW
	GPIO_ResetBits(GPIOB, GPIO_Pin_14); // WR LOW
	
	GPIO_Write(GPIOE, LCD_Data);

	// TODO implement using GPIO_ResetBits/GPIO_SetBits
	GPIO_SetBits(GPIOB, GPIO_Pin_14); // WR HIGH -> Setbits
	GPIO_SetBits(GPIOC, GPIO_Pin_8); // CS HIGH -> Setbits
}
```
그림1과 그림2를 참고하여 작성하였다.

LCD에 Command를 입력하기 위해 RD(PD15)를 HIGH, D/C(PD13)을 LOW, CS(PC8)을 LOW, WR(PB14)를 LOW로 한 다음, Command를 전송하고, 다시 WR(PB14)와 CS(PC8)을 원래대로, 즉 HIGH로 되돌려 놓는다.

LCD에 Data를 입력하기 위해 RD(PD15)를 HIGH, D/C(PD13)을 HIGH, CS(PC8)을 LOW, WR(PB14)를 LOW로 한 다음, Command를 전송하고, 다시 WR(PB14)와 CS(PC8)을 원래대로, 즉 HIGH로 되돌려 놓는다.

## 참고
GPIO_SetBits는 아래와 같이 stm32f10x_gpio.c에 정의되어 있다.

```c
/**
  * @brief  Sets the selected data port bits.
  * @param  GPIOx: where x can be (A..G) to select the GPIO peripheral.
  * @param  GPIO_Pin: specifies the port bits to be written.
  *   This parameter can be any combination of GPIO_Pin_x where x can be (0..15).
  * @retval None
  */
void GPIO_SetBits(GPIO_TypeDef* GPIOx, uint16_t GPIO_Pin)
{
  /* Check the parameters */
  //assert_param(IS_GPIO_ALL_PERIPH(GPIOx));
  //assert_param(IS_GPIO_PIN(GPIO_Pin));
  
  GPIOx->BSRR = GPIO_Pin;
}
//\Libraries\STM32F10x_StdPeriph_Driver_v3.5\src\stm32f10x_gpio.c
```

GPIO_ResetBits는 아래와 같이 stm32f10x_gpio.c에 정의되어 있다.

```c
/**
  * @brief  Clears the selected data port bits.
  * @param  GPIOx: where x can be (A..G) to select the GPIO peripheral.
  * @param  GPIO_Pin: specifies the port bits to be written.
  *   This parameter can be any combination of GPIO_Pin_x where x can be (0..15).
  * @retval None
  */
void GPIO_ResetBits(GPIO_TypeDef* GPIOx, uint16_t GPIO_Pin)
{
  /* Check the parameters */
  //assert_param(IS_GPIO_ALL_PERIPH(GPIOx));
  //assert_param(IS_GPIO_PIN(GPIO_Pin));
  
  GPIOx->BRR = GPIO_Pin;
}
//\Libraries\STM32F10x_StdPeriph_Driver_v3.5\src\stm32f10x_gpio.c
```

GPIO_Write는 아래와 같이 stm32f10x_gpio.c에 정의되어 있다.

```c
/**
  * @brief  Writes data to the specified GPIO data port.
  * @param  GPIOx: where x can be (A..G) to select the GPIO peripheral.
  * @param  PortVal: specifies the value to be written to the port output data register.
  * @retval None
  */
void GPIO_Write(GPIO_TypeDef* GPIOx, uint16_t PortVal)
{
  /* Check the parameters */
  //assert_param(IS_GPIO_ALL_PERIPH(GPIOx));
  
  GPIOx->ODR = PortVal;
}
//\Libraries\STM32F10x_StdPeriph_Driver_v3.5\src\stm32f10x_gpio.c
```

# 코드 작성
앞으로 나올 코드에서 value, x, y는 uint16_t 타입으로 선언된 전역 변수이다.

## 여러 헤더 파일 include
```c
#include <stdbool.h>
#include "stm32f10x.h"
#include "core_cm3.h"
#include "misc.h"
#include "stm32f10x_gpio.h"
#include "stm32f10x_usart.h"
#include "stm32f10x_adc.h"
#include "lcd.h"
#include "touch.h"

#include "stm32f10x_exti.h"
#include "stm32f10x_rcc.h"

```

## RCC 설정

```c
void RCC_Configure(void)
{
        RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOC,ENABLE); 
        RCC_APB2PeriphClockCmd(RCC_APB2Periph_ADC1,ENABLE); 
}
```
조도센서가 연결된 PC2를 활성화하기 위해 RCC_APB2PeriphClockCmd()를 사용하여 GPIOC에 클럭을 인가한다. 그리고 ADC를 사용하기 위해 RCC_APB2PeriphClockCmd()를 사용하여 ADC1에도 클럭을 인가한다.

## GPIO 설정
```c
void GPIO_Configure(void)
{
    GPIO_InitTypeDef GPIO_InitStructure;

    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_2; 
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AIN; 
    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz; 
    GPIO_Init(GPIOC, &GPIO_InitStructure);
}
```
GPIO_InitTypeDef 구조체를 사용하여조도센서가 연결된 PC2를 아날로그 입력으로 설정하고, GPIO_Init()을 사용하여 설정한 구조체를 바탕으로 실제 레지스터를 설정한다.

## ADC 설정
ADC에는 Regular Group, Injected Group이 있다. Regular Group은 최대 16개의 채널 할당이 가능하며 Injected Group은 최대 4개의 채널 할당이 가능하다. ADC regular group은 ADC_DR 레지스터를 통해 변환된 값을 받고(EOC Flag set), 그리고 인터럽트 발생(EOCIE가 set되어있으면), Injected Group은 ADC_JDRx(x=1..4)를 통해 변환된 값을 받는다. 그리고 JEOC flag가 set되고 인터럽트가 발생한다(JEOCIE bit가 1로 설정되어있으면).

```c
void ADC_Configure(void) {
	
    ADC_InitTypeDef ADC_12; 
	
	ADC_12.ADC_ContinuousConvMode = ENABLE; 
    ADC_12.ADC_DataAlign = ADC_DataAlign_Right; 
    ADC_12.ADC_ExternalTrigConv = ADC_ExternalTrigConv_None;
    ADC_12.ADC_Mode = ADC_Mode_Independent; 
    ADC_12.ADC_NbrOfChannel = 1; 
    ADC_12.ADC_ScanConvMode = DISABLE;
    
	ADC_Init(ADC1, &ADC_12); 
    ADC_RegularChannelConfig(ADC1, ADC_Channel_12, 1, ADC_SampleTime_239Cycles5);
    ADC_ITConfig(ADC1,ADC_IT_EOC, ENABLE);
    ADC_Cmd(ADC1, ENABLE);
    ADC_ResetCalibration(ADC1);
    while(ADC_GetResetCalibrationStatus(ADC1));
    ADC_StartCalibration(ADC1);
    while(ADC_GetCalibrationStatus(ADC1));
    ADC_SoftwareStartConvCmd(ADC1, ENABLE);
}
```
ADC_InitTypeDef 구조체를 사용하여 사용할 ADC를 설정한다.

ADC_ContinuousConvMode를 ENABLE로 설정하여 연속적으로 변환하도록 설정한다. DISABLE하여 single conversion mode로 설정하면 ADC_CR2 레지스터의 ADON 비트가 1일 때만 변환한다.

ADC_DataAlign은 ADC_DataAlign_Right로 설정하여 변환된 값 왼쪽에 4비트의 0을 붙여 ADC_DR 레지스터를 통해 4비트의 0이 붙은 변환된 값을 반환한다.

![그림 3 Data alignment](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/8fc27cb6-cc36-4544-9946-05ce8b10def3)

(그림 3 Data alignment)

![그림 4 ADC regular data register](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/df60ca33-000b-4757-b49a-bd017aa0b507)

(그림 4 ADC regular data register)

ADC_ExternalTrigConv는 ADC_ExternalTrigConv_None으로 설정하여, 외부 트리거를 받지 않는 것으로 설정한다.

그리고 ADC Mode는 independent 모드로 설정하고, NbrOfChannel은 1로 설정하여 채널의 개수는 1로 설정하고, ScanCovMode는 채널 그룹 내의 모든 채널을 순서대로 스캔하여 변환하는 모드로, 지금은 한 개의 채널만 사용하면 되므로 DISABLE로 설정한다. 그리고 ADC_Init()으로 설정한 구조체를 실제 레지스터에 반영한다.

ADC_RegularChannelConfig를 사용하여 ADC regular channel을 설정한다. ADC1을 설정한다고 명시하고, 채널은 ADC_Channel_12로 설정한다. 그리고 채널 rank는 1, 샘플링 타임은 239.5Cycle로 지정한다.

![그림 5 PC2는 ADC1에서 channel 12와 연결되어있다.](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/4f880aba-785f-46b8-a592-8d512e898613)

(그림 5 PC2는 ADC1에서 channel 12와 연결되어있다.)

ADC_ITConfig를 사용하여 ADT_IT_EOC, 즉 변환이 끝났을 때 인터럽트가 발생하도록 설정한다.

그리고 ADC_Cmd를 사용하여 ADC1을 활성화시킨다.

ADC_ResetCalibration을 사용하여 calibration을 초기화하고, 초기화될때까지 기다린 후, calibration과정을 시작한다. 시작될 때까지 기다린 후 변환을 시작한다.

## NVIC 설정

```c
void NVIC_Configure(void) {

    NVIC_InitTypeDef NVIC_InitStructure;
    
    NVIC_PriorityGroupConfig(NVIC_PriorityGroup_1);
	
    NVIC_InitStructure.NVIC_IRQChannel = ADC1_2_IRQn;
    NVIC_InitStructure.NVIC_IRQChannelCmd = ENABLE;
    NVIC_InitStructure.NVIC_IRQChannelSubPriority = 0x00;
    NVIC_InitStructure.NVIC_IRQChannelPreemptionPriority = 0x00;
    NVIC_Init(&NVIC_InitStructure);
    
}

```
NVIC_InitTypeDef 구조체를 사용하여 ADC1_2 인터럽트를 활성화시킨다.

## ADC 인터럽트 핸들러

```c
void ADC1_2_IRQHandler() {
	
  if(ADC_GetITStatus(ADC1,ADC_IT_EOC) != RESET) { 
	  
    value = ADC_GetConversionValue(ADC1); 
	
  }
  
  ADC_ClearITPendingBit(ADC1, ADC_IT_EOC);
  
}

```
ADC로부터 인터럽트가 발생하였다면 발생한 인터럽트가 변환이 끝나서 발생한 인터럽트인지 확인하고, 해당 인터럽트라면, 변환한 값을 받은 후 pendingbit를 초기화한다.

## main() 작성
```c
int main(void)
{

    SystemInit();
    RCC_Configure();
    GPIO_Configure();
    ADC_Configure();
    NVIC_Configure();
    
    LCD_Init();
    Touch_Configuration();
    Touch_Adjust();
    LCD_Clear(WHITE);

    
    LCD_ShowString(80,80,"THUR_Team05",BLACK,WHITE); 
    
    while (1) {
      Touch_GetXY(&x,&y,1); 
	  
      Convert_Pos(x,y,&x,&y); 
      
	  if(x !=0 || y !=0) {
          LCD_DrawCircle(x,y,3);							      
		  
          LCD_ShowNum(80,120,value,4,BLACK,WHITE); 
          LCD_ShowNum(80,140,x,3,BLACK,WHITE); 	  
          LCD_ShowNum(80,160,y,3,BLACK,WHITE);      
        }
      
    }

}

```

LCD와 touch 관련 초기화 및 설정을 하고(LCD 라이브러리에 정의되어 있음) LCD_ShowString을 사용하여 LCD에 “THUR_Team05”를 표시하고, Touch_GetXY를 사용하여 터치된 좌표를 얻고, 좌표를 조정한 다음, 해당 좌표에 원을 (LCD_DrawCircle)그리고, LCD_ShowNum을 사용하여 조도센서값과 좌표를 표시한다.

# 실험 결과

{% include video id="xdbkA55rpo4" provider="youtube" %}

코드를 보드에 다운로드하고 실행하면 먼저 십자 모양이 뜨는데, 십자모양을 클릭하여 터치 감도를 조절한다. 터치 감도조절이 끝나면 그림6과 같이 터치된 곳에 작은 원이 표시되고, 조도 센서 값, 터치된 곳의 좌표, 그리고 “THUR_Team05”가 표시된다.

# 알게 된 점
아날로그 값은 ADC를 통해 디지털 값으로 변환하여 사용할 수 있다는 점을 알게 되었으며 LCD 조작 방법을 알게 되었다.

# 참고 문헌
- STM32F107 Datasheet https://www.st.com/resource/en/datasheet/stm32f107vc.pdf 
- STM32F107 Reference Manual
 https://www.st.com/resource/en/reference_manual/rm0008- stm32f101xxstm32f102xxstm32f103xx-stm32f105xx-and-stm32f107xxadvancedarmbased-32bitmcusstmicroelectronics.pdf 
- STM32F107VCT6 schematic
- Solomon Systech SSD1298 LCD Controller Datasheet 
https://www.crystalfontz.com/controllers/SolomonSystech/SSD1298/
-	STM32 ADC 펌웨어 가이드 (모드와 특성)
https://www.st.com/content/dam/kms/Contents/Reflibrary/ADC_Firmware_guide_Mode_and_Feature.pdf
-	https://blog.naver.com/ceg_tiny/221067538968
-	https://boardmix.com/kr/skills/timing-diagram/
-	https://en.wikipedia.org/wiki/Nyquist%E2%80%93Shannon_sampling_theorem

# 참고 - ADC 관련 라이브러리 구조체 및 함수

## ADC_InitTypeDef
ADC_InitTypeDef 구조체와 멤버는 아래와 같다.

```c
typedef struct
{
  uint32_t ADC_Mode;                      /*!< Configures the ADC to operate in independent or
                                               dual mode. 
                                               This parameter can be a value of @ref ADC_mode */

  FunctionalState ADC_ScanConvMode;       /*!< Specifies whether the conversion is performed in
                                               Scan (multichannels) or Single (one channel) mode.
                                               This parameter can be set to ENABLE or DISABLE */

  FunctionalState ADC_ContinuousConvMode; /*!< Specifies whether the conversion is performed in
                                               Continuous or Single mode.
                                               This parameter can be set to ENABLE or DISABLE. */

  uint32_t ADC_ExternalTrigConv;          /*!< Defines the external trigger used to start the analog
                                               to digital conversion of regular channels. This parameter
                                               can be a value of @ref ADC_external_trigger_sources_for_regular_channels_conversion */

  uint32_t ADC_DataAlign;                 /*!< Specifies whether the ADC data alignment is left or right.
                                               This parameter can be a value of @ref ADC_data_align */

  uint8_t ADC_NbrOfChannel;               /*!< Specifies the number of ADC channels that will be converted
                                               using the sequencer for regular channel group.
                                               This parameter must range from 1 to 16. */
}ADC_InitTypeDef;

...

/** @defgroup ADC_mode 
  * @{
  */

#define ADC_Mode_Independent                       ((uint32_t)0x00000000)
#define ADC_Mode_RegInjecSimult                    ((uint32_t)0x00010000)
#define ADC_Mode_RegSimult_AlterTrig               ((uint32_t)0x00020000)
#define ADC_Mode_InjecSimult_FastInterl            ((uint32_t)0x00030000)
#define ADC_Mode_InjecSimult_SlowInterl            ((uint32_t)0x00040000)
#define ADC_Mode_InjecSimult                       ((uint32_t)0x00050000)
#define ADC_Mode_RegSimult                         ((uint32_t)0x00060000)
#define ADC_Mode_FastInterl                        ((uint32_t)0x00070000)
#define ADC_Mode_SlowInterl                        ((uint32_t)0x00080000)
#define ADC_Mode_AlterTrig                         ((uint32_t)0x00090000)

...
/**
  * @}
  */

/** @defgroup ADC_external_trigger_sources_for_regular_channels_conversion 
  * @{
  */

#define ADC_ExternalTrigConv_T1_CC1                ((uint32_t)0x00000000) /*!< For ADC1 and ADC2 */
#define ADC_ExternalTrigConv_T1_CC2                ((uint32_t)0x00020000) /*!< For ADC1 and ADC2 */
#define ADC_ExternalTrigConv_T2_CC2                ((uint32_t)0x00060000) /*!< For ADC1 and ADC2 */
#define ADC_ExternalTrigConv_T3_TRGO               ((uint32_t)0x00080000) /*!< For ADC1 and ADC2 */
#define ADC_ExternalTrigConv_T4_CC4                ((uint32_t)0x000A0000) /*!< For ADC1 and ADC2 */
#define ADC_ExternalTrigConv_Ext_IT11_TIM8_TRGO    ((uint32_t)0x000C0000) /*!< For ADC1 and ADC2 */

#define ADC_ExternalTrigConv_T1_CC3                ((uint32_t)0x00040000) /*!< For ADC1, ADC2 and ADC3 */
#define ADC_ExternalTrigConv_None                  ((uint32_t)0x000E0000) /*!< For ADC1, ADC2 and ADC3 */

#define ADC_ExternalTrigConv_T3_CC1                ((uint32_t)0x00000000) /*!< For ADC3 only */
#define ADC_ExternalTrigConv_T2_CC3                ((uint32_t)0x00020000) /*!< For ADC3 only */
#define ADC_ExternalTrigConv_T8_CC1                ((uint32_t)0x00060000) /*!< For ADC3 only */
#define ADC_ExternalTrigConv_T8_TRGO               ((uint32_t)0x00080000) /*!< For ADC3 only */
#define ADC_ExternalTrigConv_T5_CC1                ((uint32_t)0x000A0000) /*!< For ADC3 only */
#define ADC_ExternalTrigConv_T5_CC3                ((uint32_t)0x000C0000) /*!< For ADC3 only */

...

/** @defgroup ADC_data_align 
  * @{
  */

#define ADC_DataAlign_Right                        ((uint32_t)0x00000000)
#define ADC_DataAlign_Left                         ((uint32_t)0x00000800)
#define IS_ADC_DATA_ALIGN(ALIGN) (((ALIGN) == ADC_DataAlign_Right) || \
                                  ((ALIGN) == ADC_DataAlign_Left))
/**
  * @}
  */
  ...

//\Libraries\STM32F10x_StdPeriph_Driver_v3.5\inc\stm32f10x_adc.h
```

## ADC_Init()
ADC_Init 함수에 대한 설명은 아래와 같다.

```c
/**
  * @brief  Initializes the ADCx peripheral according to the specified parameters
  *         in the ADC_InitStruct.
  * @param  ADCx: where x can be 1, 2 or 3 to select the ADC peripheral.
  * @param  ADC_InitStruct: pointer to an ADC_InitTypeDef structure that contains
  *         the configuration information for the specified ADC peripheral.
  * @retval None
  */
void ADC_Init(ADC_TypeDef* ADCx, ADC_InitTypeDef* ADC_InitStruct)
//\Libraries\STM32F10x_StdPeriph_Driver_v3.5\inc\stm32f10x_adc.h
```

## ADC_RegularChannelConfig()
ADC_RegularChannelConfig()에 대한 설명은 아래와 같다.

```c
/**
  * @brief  Configures for the selected ADC regular channel its corresponding
  *         rank in the sequencer and its sample time.
  * @param  ADCx: where x can be 1, 2 or 3 to select the ADC peripheral.
  * @param  ADC_Channel: the ADC channel to configure. 
  *   This parameter can be one of the following values:
  *     @arg ADC_Channel_0: ADC Channel0 selected
  *     @arg ADC_Channel_1: ADC Channel1 selected
  *     @arg ADC_Channel_2: ADC Channel2 selected
  *     @arg ADC_Channel_3: ADC Channel3 selected
  *     @arg ADC_Channel_4: ADC Channel4 selected
  *     @arg ADC_Channel_5: ADC Channel5 selected
  *     @arg ADC_Channel_6: ADC Channel6 selected
  *     @arg ADC_Channel_7: ADC Channel7 selected
  *     @arg ADC_Channel_8: ADC Channel8 selected
  *     @arg ADC_Channel_9: ADC Channel9 selected
  *     @arg ADC_Channel_10: ADC Channel10 selected
  *     @arg ADC_Channel_11: ADC Channel11 selected
  *     @arg ADC_Channel_12: ADC Channel12 selected
  *     @arg ADC_Channel_13: ADC Channel13 selected
  *     @arg ADC_Channel_14: ADC Channel14 selected
  *     @arg ADC_Channel_15: ADC Channel15 selected
  *     @arg ADC_Channel_16: ADC Channel16 selected
  *     @arg ADC_Channel_17: ADC Channel17 selected
  * @param  Rank: The rank in the regular group sequencer. This parameter must be between 1 to 16.
  * @param  ADC_SampleTime: The sample time value to be set for the selected channel. 
  *   This parameter can be one of the following values:
  *     @arg ADC_SampleTime_1Cycles5: Sample time equal to 1.5 cycles
  *     @arg ADC_SampleTime_7Cycles5: Sample time equal to 7.5 cycles
  *     @arg ADC_SampleTime_13Cycles5: Sample time equal to 13.5 cycles
  *     @arg ADC_SampleTime_28Cycles5: Sample time equal to 28.5 cycles	
  *     @arg ADC_SampleTime_41Cycles5: Sample time equal to 41.5 cycles	
  *     @arg ADC_SampleTime_55Cycles5: Sample time equal to 55.5 cycles	
  *     @arg ADC_SampleTime_71Cycles5: Sample time equal to 71.5 cycles	
  *     @arg ADC_SampleTime_239Cycles5: Sample time equal to 239.5 cycles	
  * @retval None
  */
void ADC_RegularChannelConfig(ADC_TypeDef* ADCx, uint8_t ADC_Channel, uint8_t Rank, uint8_t ADC_SampleTime)

//\Libraries\STM32F10x_StdPeriph_Driver_v3.5\src\stm32f10x_adc.c
```

## ADC_ITConfig()
ADC_ITConfig()에 대한 설명은 아래와 같다.
```c
/**
  * @brief  Enables or disables the specified ADC interrupts.
  * @param  ADCx: where x can be 1, 2 or 3 to select the ADC peripheral.
  * @param  ADC_IT: specifies the ADC interrupt sources to be enabled or disabled. 
  *   This parameter can be any combination of the following values:
  *     @arg ADC_IT_EOC: End of conversion interrupt mask
  *     @arg ADC_IT_AWD: Analog watchdog interrupt mask
  *     @arg ADC_IT_JEOC: End of injected conversion interrupt mask
  * @param  NewState: new state of the specified ADC interrupts.
  *   This parameter can be: ENABLE or DISABLE.
  * @retval None
  */
void ADC_ITConfig(ADC_TypeDef* ADCx, uint16_t ADC_IT, FunctionalState NewState)

//\Libraries\STM32F10x_StdPeriph_Driver_v3.5\src\stm32f10x_adc.c
```
## ADC_Cmd()
ADC_Cmd()에 대한 설명은 아래와 같다.

```c
/**
  * @brief  Enables or disables the specified ADC peripheral.
  * @param  ADCx: where x can be 1, 2 or 3 to select the ADC peripheral.
  * @param  NewState: new state of the ADCx peripheral.
  *   This parameter can be: ENABLE or DISABLE.
  * @retval None
  */
void ADC_Cmd(ADC_TypeDef* ADCx, FunctionalState NewState)
```

## ADC_ResetCalibration()
ADC_ResetCalibration()에 대한 설명은 아래와 같다.
```c
/**
  * @brief  Resets the selected ADC calibration registers.
  * @param  ADCx: where x can be 1, 2 or 3 to select the ADC peripheral.
  * @retval None
  */
void ADC_ResetCalibration(ADC_TypeDef* ADCx)
```

## ADC_GetResetCalibrationStatus()
ADC_GetResetCalibrationStatus()에 대한 설명은 아래와 같다.
```c
/**
  * @brief  Gets the selected ADC reset calibration registers status.
  * @param  ADCx: where x can be 1, 2 or 3 to select the ADC peripheral.
  * @retval The new state of ADC reset calibration registers (SET or RESET).
  */
FlagStatus ADC_GetResetCalibrationStatus(ADC_TypeDef* ADCx)
```

## ADC_StartCalibration()
ADC_StartCalibration()에 대한 설명은 아래와 같다.
```c
/**
  * @brief  Starts the selected ADC calibration process.
  * @param  ADCx: where x can be 1, 2 or 3 to select the ADC peripheral.
  * @retval None
  */
void ADC_StartCalibration(ADC_TypeDef* ADCx)
```

## ADC_GetCalibrationStatus()
ADC_GetCalibrationStatus()에 대한 설명은 아래와 같다.
```c
/**
  * @brief  Gets the selected ADC calibration status.
  * @param  ADCx: where x can be 1, 2 or 3 to select the ADC peripheral.
  * @retval The new state of ADC calibration (SET or RESET).
  */
FlagStatus ADC_GetCalibrationStatus(ADC_TypeDef* ADCx)
```

## ADC_SoftwareStartConvCmd()
ADC_SoftwareStartConvCmd()에 대한 설명은 아래와 같다.
```c
/**
  * @brief  Enables or disables the selected ADC software start conversion .
  * @param  ADCx: where x can be 1, 2 or 3 to select the ADC peripheral.
  * @param  NewState: new state of the selected ADC software start conversion.
  *   This parameter can be: ENABLE or DISABLE.
  * @retval None
  */
void ADC_SoftwareStartConvCmd(ADC_TypeDef* ADCx, FunctionalState NewState)
```

