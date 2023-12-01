---
title: "measuring photoresistor value using dma and adc on stm32f107vc"
last_modified_at: 2023-11-24T16:28:12+09:00
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
- DMA 동작 방법의 이해
- DMA 구현

# 실험 원리

## Direct Memory Access (DMA)
주변장치들이 CPU의 개입없이 메모리에 직접 접근하여 읽거나 쓸 수 있도록 하는 기능이다. 즉 CPU의 개입없이 I/O 장치와 기억장치 데이터를 전송하는 방식이다. 인터럽트와 달리 중앙제어장치는 명령을 실행할 필요가 없다. 메모리 처리 interrupt의 사이클만큼 성능의 향상이 있다.

## 일반적인 메모리 접근 방식와 DMA 차이
일반적인 메모리 접근 방식은 모든 I/O로의 접근은 CPU를 통해서 수행되고, 데이터를 전달할 때마다 CPU가 관여한다.

DMA 방식은 RAM이 I/O 장치로부터 데이터가 필요해지면, CPU는 DMA 컨트롤러에게 신호(전송 크기, 주소 등)를 보낸다. 그리고 DMA 컨트롤러가 RAM 주소로 데이터를 bus를 통해 주고받는다. 모든 데이터 전송이 끝나면, DMA controller가 CPU에게 인터럽트 신호를 보낸다.

## DMA Channel
모듈은 DMA Controller의 DMA 채널을 통해 메모리에 R/W한다. STM32 보드의 DMA 채널은 총 12개(DMA1 7개, DMA2 5개)가 있으며, 한 DMA의 여러 채널 사이 요청은 priority(very high, high, medium, low)에 따라 동작한다. Peripheral-to-memory, memory-to-peripheral, peripheral-to-peripheral 전송이 있다.

## DMA Mode

### Normal Mode
DMA Controller는 데이터를 전송할 때 마다 NDT 값을 감소시킨다. NDT는 DMA를 통해 전송할 데이터의 총 용량을 의미하며 레지스터의 값이 0이 되면 데이터 전송을 중단한다. 데이터 전송을 받고 싶을 때 마다 새롭게 요청이 필요하다.

### Circular Mode
주기적인 값의 전송(업데이트)이 필요할 떄 사용하는 모드로, NDT 값이 0이 될 경우 설정한 데이터 최대 크기로 재설정된다.

## DMA Controller
주변 장치에서 Request Signal이 발생하면, DMA Controller에서 우선순위에 따라 요청에 대한 서비스를 제공한다. Request/ACK 방식을 통해 주변 장치와 DMA Controller 간에 통신을 한다.

![그림1: DMA block diagram](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/f35bd012-f879-427e-bc14-bfc2a9ba2f47)

(그림1: DMA block diagram)

![그림2: stm32 Memory Map. DMA1은 0x40020000부터 0x400203FF까지이다.](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/ef68c3ab-ac57-43a8-9856-873b9fd85e7e)

(그림2: stm32 Memory Map. DMA1은 0x40020000부터 0x400203FF까지이다.)

## DMA Channel

![표1: DMA1 requests for each channel](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/50b52a43-c475-4413-b571-186388e19197)

(표1 : DMA1 requests for each channel)

![표2: DMA2 requests for each channel](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/f68f8520-7620-4e71-8703-5c49b64e061d)

(표2: DMA2 requests for each channel)

## DMA configuration
DMA_Init()의 코드는 다음과 같다.

```c
/**
  * @brief  Initializes the DMAy Channelx according to the specified
  *         parameters in the DMA_InitStruct.
  * @param  DMAy_Channelx: where y can be 1 or 2 to select the DMA and 
  *   x can be 1 to 7 for DMA1 and 1 to 5 for DMA2 to select the DMA Channel.
  * @param  DMA_InitStruct: pointer to a DMA_InitTypeDef structure that
  *         contains the configuration information for the specified DMA Channel.
  * @retval None
  */
void DMA_Init(DMA_Channel_TypeDef* DMAy_Channelx, DMA_InitTypeDef* DMA_InitStruct)
{
  uint32_t tmpreg = 0;

  /* Check the parameters */
  //assert_param(IS_DMA_ALL_PERIPH(DMAy_Channelx));
  //assert_param(IS_DMA_DIR(DMA_InitStruct->DMA_DIR));
  //assert_param(IS_DMA_BUFFER_SIZE(DMA_InitStruct->DMA_BufferSize));
  //assert_param(IS_DMA_PERIPHERAL_INC_STATE(DMA_InitStruct->DMA_PeripheralInc));
  //assert_param(IS_DMA_MEMORY_INC_STATE(DMA_InitStruct->DMA_MemoryInc));
  //assert_param(IS_DMA_PERIPHERAL_DATA_SIZE(DMA_InitStruct->DMA_PeripheralDataSize));
  //assert_param(IS_DMA_MEMORY_DATA_SIZE(DMA_InitStruct->DMA_MemoryDataSize));
  //assert_param(IS_DMA_MODE(DMA_InitStruct->DMA_Mode));
  //assert_param(IS_DMA_PRIORITY(DMA_InitStruct->DMA_Priority));
  //assert_param(IS_DMA_M2M_STATE(DMA_InitStruct->DMA_M2M));

/*--------------------------- DMAy Channelx CCR Configuration -----------------*/
  /* Get the DMAy_Channelx CCR value */
  tmpreg = DMAy_Channelx->CCR;
  /* Clear MEM2MEM, PL, MSIZE, PSIZE, MINC, PINC, CIRC and DIR bits */
  tmpreg &= CCR_CLEAR_Mask;
  /* Configure DMAy Channelx: data transfer, data size, priority level and mode */
  /* Set DIR bit according to DMA_DIR value */
  /* Set CIRC bit according to DMA_Mode value */
  /* Set PINC bit according to DMA_PeripheralInc value */
  /* Set MINC bit according to DMA_MemoryInc value */
  /* Set PSIZE bits according to DMA_PeripheralDataSize value */
  /* Set MSIZE bits according to DMA_MemoryDataSize value */
  /* Set PL bits according to DMA_Priority value */
  /* Set the MEM2MEM bit according to DMA_M2M value */
  tmpreg |= DMA_InitStruct->DMA_DIR | DMA_InitStruct->DMA_Mode |
            DMA_InitStruct->DMA_PeripheralInc | DMA_InitStruct->DMA_MemoryInc |
            DMA_InitStruct->DMA_PeripheralDataSize | DMA_InitStruct->DMA_MemoryDataSize |
            DMA_InitStruct->DMA_Priority | DMA_InitStruct->DMA_M2M;

  /* Write to DMAy Channelx CCR */
  DMAy_Channelx->CCR = tmpreg;

/*--------------------------- DMAy Channelx CNDTR Configuration ---------------*/
  /* Write to DMAy Channelx CNDTR */
  DMAy_Channelx->CNDTR = DMA_InitStruct->DMA_BufferSize;

/*--------------------------- DMAy Channelx CPAR Configuration ----------------*/
  /* Write to DMAy Channelx CPAR */
  DMAy_Channelx->CPAR = DMA_InitStruct->DMA_PeripheralBaseAddr;

/*--------------------------- DMAy Channelx CMAR Configuration ----------------*/
  /* Write to DMAy Channelx CMAR */
  DMAy_Channelx->CMAR = DMA_InitStruct->DMA_MemoryBaseAddr;
}
//\Libraries\STM32F10x_StdPeriph_Driver_v3.5\src\stm32f10x_dma.c
```
위의 코드와 같이 DMA_Init()은 DMA channel x configuration register(DMA_CCRx)를 설정하며, DMA_InitTypeDef 구조체에서 설정한 대로 실제 레지스터를 설정한다.

![그림3: DMA channel x configuration register 설명](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/e2185b70-66b0-461b-8aa6-255d7ada1dfb)

(그림3: DMA channel x configuration register 설명)

# 세부 실험 내용
1.	DMA 및 ADC를 사용하여 1개의 조도센서 값을 받아오도록 구현
2.	읽은 조도센서 값을 TFT-LCD에 출력
3.	평상시 TFT-LCD 배경색 WHITE, 조도센서에 스마트폰 플래시로 비출 때 TFT-LCD 배경색 GRAY(배경색 바꾸면 글자도 사라지므로 배경 바꾸고 조도 값 띄우기)
4.	조도센서 값 threshold는 실험적으로 결정

# 실험 방법
1.	조도 센서 한쪽 단자는 그라운드에 연결하고, 다른쪽 단자는 저항을 지나 VCC에 연결한다. 조도 센서와 저항 사이를 보드의 핀(PC2)과 연결하여 전압을 측정한다(PC2는 ADC1(또는 ADC2)의 Channel 12이다).
2.	DMA1, ADC1, GPIOC에 클럭을 인가한다.
3.	PC2를 아날로그 입력으로 설정한다.
4.	ADC1이 PC2로 부터 입력을 받도록 설정한다.
5.	DMA를 설정한다.
6.	조도 센서가 임계값보다 작으면 LCD 배경을 회색으로, 임계값보다 크면 LCD 배경을 흰색으로 바꾸도록 코딩한다.

# 코드 작성

## 헤더 파일 include
```c
#include "stm32f10x_adc.h"
#include "stm32f10x_rcc.h"
#include "stm32f10x_gpio.h"
#include "stm32f10x_usart.h"
#include "misc.h"
#include "lcd.h"
#include "touch.h"
#include "stm32f10x_dma.h"
```
stm32를 설정하는데 쓰이는 구조체와 함수가 정의되어있는 헤더 파일을 include 한다.

## DMA를 통해 들어올 ADC값을 저장하는 메모리 공간 할당
```c
volatile uint32_t ADC_Value[1];
```
DMA를 통해 ADC값을 저장하도록, ADC값을 저장할 메모리 공간을 할당한다.

## RCC 설정
```c
void RccInit(void) {
    RCC_AHBPeriphClockCmd(RCC_AHBPeriph_DMA1, ENABLE);
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_ADC1, ENABLE);
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOC, ENABLE);
}
```
이번 실험에서 쓸 DMA1, ADC1, GPIOC에 클럭을 인가한다.

## GPIO 설정
```c
void GpioInit(void) {
   GPIO_InitTypeDef GPIO_InitStructure;
   GPIO_InitStructure.GPIO_Pin = GPIO_Pin_2;
   GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AIN;
   GPIO_Init(GPIOC, &GPIO_InitStructure);
}
```
조도센서가 연결되어있는 PC2 핀을 GPIO_InitTypeDef 구조체와 GPIO_Init()을 이용하여 PC2를 아날로그 입력으로 설정한다.

## ADC 설정
```c
void AdcInit(void) {
    ADC_InitTypeDef ADC_InitStructure;
   
    ADC_InitStructure.ADC_Mode = ADC_Mode_Independent; // 독립모드(하나의 ADC만 사용) 설정
    ADC_InitStructure.ADC_ScanConvMode = DISABLE; // 스캔 변환 모드(여러 채널을 순차적으로 변환하는 모드) 비활성화
    ADC_InitStructure.ADC_ContinuousConvMode = ENABLE; //
    ADC_InitStructure.ADC_ExternalTrigConv = ADC_ExternalTrigConv_None; // 외부 트리거 변환을 사용하지 않도록 설정
    ADC_InitStructure.ADC_DataAlign = ADC_DataAlign_Right; // 데이터를 오른쪽 정렬
    ADC_InitStructure.ADC_NbrOfChannel = 1; // 변환할 채널 수 1 설정
    ADC_Init(ADC1, &ADC_InitStructure);
   
   // ADC_Channerl_12를 연결, 샘플링 시간 239.5 사이클로 설정
    ADC_RegularChannelConfig(ADC1, ADC_Channel_12, 1, ADC_SampleTime_239Cycles5);
    ADC_DMACmd(ADC1, ENABLE);  // ADC DMA 전송 활성화
   
    ADC_Cmd(ADC1, ENABLE);
    ADC_ResetCalibration(ADC1);

    while(ADC_GetResetCalibrationStatus(ADC1));

    ADC_StartCalibration(ADC1); // ADC 활성화, 보정 시작

    while(ADC_GetCalibrationStatus(ADC1));

    ADC_SoftwareStartConvCmd(ADC1, ENABLE);  // ADC 변환 시작
}
```
ADC1만 사용하므로, Independent 모드로 설정한다. 그리고 채널1개만 사용하므로, ScanConvMode(여러 채널을 순차적으로 변환)는 비활성화한다. 그리고 특정 조건에서만 변환하는 것이 아니라 계속 변환해야하므로, ContinuousConvMode는 활성화한다. 그리고 외부 트리거는 사용하지 않는다(ExternalTrigConv 비활성화). 그리고 ADC의 DR 레지스터(데이터 레지스터, 16비트)에 12비트 변환 데이터를 오른쪽 정렬한다(ADC_DataAlign_Right). 그리고 변환할 채널 수는 1개로 설정하고, 마지막으로 ADC_Init()을 호출하여 설정한 구조체를 실제 레지스터에 반영시킨다.

그리고 ADC1에서 채널12를 통해 신호를 받도록 설정한다. 그리고 ADC_DMACmd()를 호출하여 ADC1 값을 DMA로 전송하도록 설정한다.

## DMA 설정
```c
void DMA_Configure(void) {
   DMA_InitTypeDef DMA_Instructure; 
   
   // 구조체를 사용하여 DMA1 채널 1의 초기화 설정
   DMA_Instructure.DMA_PeripheralBaseAddr = (uint32_t)&ADC1->DR;          
   DMA_Instructure.DMA_MemoryBaseAddr = (uint32_t)&ADC_Value[0]; // ADC_Value배열 주소로 설정
   DMA_Instructure.DMA_DIR = DMA_DIR_PeripheralSRC;
   DMA_Instructure.DMA_BufferSize = 1;   // 전송 버퍼 크기 1 설정                              
   DMA_Instructure.DMA_PeripheralInc = DMA_PeripheralInc_Disable;   // 주변 기기 주소의 증가 비활성화
   DMA_Instructure.DMA_MemoryInc = DMA_MemoryInc_Disable;               
   DMA_Instructure.DMA_PeripheralDataSize = DMA_PeripheralDataSize_Word;   
   DMA_Instructure.DMA_MemoryDataSize = DMA_MemoryDataSize_Word;      
   DMA_Instructure.DMA_Mode = DMA_Mode_Circular;   // 순환 모드 설정
   DMA_Instructure.DMA_Priority = DMA_Priority_High;   // 우선순위 높음 설정            
   DMA_Instructure.DMA_M2M = DMA_M2M_Disable;   // 메모리간 전송 비활성화      
   
   // DMA 초기화, DMA 전송 활성화
   DMA_Init(DMA1_Channel1, &DMA_Instructure);                              
   DMA_Cmd(DMA1_Channel1, ENABLE);                                       
}

```
먼저 주변장치의 주소를 ADC1의 DR(data register)로 설정한다. 그리고 주변장치로부터 데이터를 받을 메모리 주소를 위에서 할당한 ADC_Value[0]의 주소로 한다. 그리고 주변장치로부터 메모리로 데이터를 받으므로, DMA_DIR_PheripheralSRC로 설정한다. 그리고 전송 버퍼 크기는 1로 설정하고, 주변 장치 주소 증가 및 메모리 주소 증가는 비활성화한다. 데이터 크기는 CPU의 기본 데이터 크기(word size)로 설정한다. 그리고 순환모드로 설정하고, 우선순위는 High, 메모리간 전송은 비활성화한다.

## main 함수 작성
```c
int main(){
   SystemInit();
   RccInit();
   GpioInit();
   AdcInit();
    DMA_Configure();
   
   LCD_Init();
   Touch_Configuration();
   Touch_Adjust();
   
   int light = 0;            
    int before_light = -1;   
   
   LCD_Clear(WHITE);         
    
   LCD_ShowString(20, 20, "THUR_Team05", RED, WHITE);
   
    while(1){
      // 임계값보다 작을 경우 1, 크면 0
        light = ADC_Value[0] < 3300; 
      
      // 이전 조도값과 다를 경우
        if (before_light != light){      
         if (light) {
            LCD_Clear(GRAY); 
            LCD_ShowString(20, 20, "THUR_Team05", RED, GRAY);
            LCD_ShowNum(20, 50, ADC_Value[0], 4, RED, GRAY); 
         }
         else {
            LCD_Clear(WHITE); 
            LCD_ShowString(20, 20, "THUR_Team05", RED, WHITE);
            LCD_ShowNum(20, 50, ADC_Value[0], 4, RED, WHITE); 
         }
        }
        before_light = light; 
      }
}
```
먼저 SystemInit()을 통해 클럭을 초기화한다. 그리고 위에서 정의한 각종 함수를 호출하여 여러가지 설정을 하고, LCD_Init()과 Touch_Configuration(), Touch_Adjust()를 호출하여 LCD와 터치를 초기화하고 터치 감도를 조절한다.

조도 센서의 값이 임계값(3300)보다 작을 경우 밝고, 클 경우 어두운 것으로 간주하여, 밝음에서 어두움, 또는 어두움에서 밝음을 감지하여, 밝아졌으면 배경을 회색으로, 글씨를 빨간색으로 설정한다. 어두워졌으면 배경을 흰색으로, 글씨를 빨간색으로 지정한다. 밝음에서 어두움, 또는 어두움에서 밝아졌으면 ADC_Value[0] 값을 LCD에 쓴다.

# 실험 결과
{% include video id="GDmBPgjngj8" provider="youtube" %}

조도 센서 값이 클 때는 배경 색이 흰색이고, 조도 센서에 빛을 비추어 조도 센서 값을 낮추면 배경 색이 흰색으로 바뀐다. “THUR_Team05”와 조도 센서 값이 표시되고 있고, 조도 센서 값은 밝음에서 어두움 또는 어두움에서 밝음 시에만 업데이트된다.

# 알게 된 점
DMA를 통해 인터럽트 없이도 주변 장치의 값을 읽어올 수 있다는 것을 알게 되었다.

# 참고 문헌
- STM32F107 Datasheet https://www.st.com/resource/en/datasheet/stm32f107vc.pdf 
- STM32F107 Reference Manual https://www.st.com/resource/en/reference_manual/rm0008-stm32f101xxstm32f102xxstm32f103xx-stm32f105xx-andstm32f107xxadvancedarmbased32bitmcusstmicroelectronics.pdf 
- STM32F107VCT6 schematic
- https://www.quora.com/What-is-the-function-of-DMA-in-a-computer

# 참고 라이브러리 코드

## DMA_InitTypeDef 구조체
```c
/** 
  * @brief  DMA Init structure definition
  */

typedef struct
{
  uint32_t DMA_PeripheralBaseAddr; /*!< Specifies the peripheral base address for DMAy Channelx. */

  uint32_t DMA_MemoryBaseAddr;     /*!< Specifies the memory base address for DMAy Channelx. */

  uint32_t DMA_DIR;                /*!< Specifies if the peripheral is the source or destination.
                                        This parameter can be a value of @ref DMA_data_transfer_direction */

  uint32_t DMA_BufferSize;         /*!< Specifies the buffer size, in data unit, of the specified Channel. 
                                        The data unit is equal to the configuration set in DMA_PeripheralDataSize
                                        or DMA_MemoryDataSize members depending in the transfer direction. */

  uint32_t DMA_PeripheralInc;      /*!< Specifies whether the Peripheral address register is incremented or not.
                                        This parameter can be a value of @ref DMA_peripheral_incremented_mode */

  uint32_t DMA_MemoryInc;          /*!< Specifies whether the memory address register is incremented or not.
                                        This parameter can be a value of @ref DMA_memory_incremented_mode */

  uint32_t DMA_PeripheralDataSize; /*!< Specifies the Peripheral data width.
                                        This parameter can be a value of @ref DMA_peripheral_data_size */

  uint32_t DMA_MemoryDataSize;     /*!< Specifies the Memory data width.
                                        This parameter can be a value of @ref DMA_memory_data_size */

  uint32_t DMA_Mode;               /*!< Specifies the operation mode of the DMAy Channelx.
                                        This parameter can be a value of @ref DMA_circular_normal_mode.
                                        @note: The circular buffer mode cannot be used if the memory-to-memory
                                              data transfer is configured on the selected Channel */

  uint32_t DMA_Priority;           /*!< Specifies the software priority for the DMAy Channelx.
                                        This parameter can be a value of @ref DMA_priority_level */

  uint32_t DMA_M2M;                /*!< Specifies if the DMAy Channelx will be used in memory-to-memory transfer.
                                        This parameter can be a value of @ref DMA_memory_to_memory */
}DMA_InitTypeDef;

/** @defgroup DMA_data_transfer_direction 
  * @{
  */

#define DMA_DIR_PeripheralDST              ((uint32_t)0x00000010)
#define DMA_DIR_PeripheralSRC              ((uint32_t)0x00000000)
#define IS_DMA_DIR(DIR) (((DIR) == DMA_DIR_PeripheralDST) || \
                         ((DIR) == DMA_DIR_PeripheralSRC))
/**
  * @}
  */

/** @defgroup DMA_peripheral_incremented_mode 
  * @{
  */

#define DMA_PeripheralInc_Enable           ((uint32_t)0x00000040)
#define DMA_PeripheralInc_Disable          ((uint32_t)0x00000000)
#define IS_DMA_PERIPHERAL_INC_STATE(STATE) (((STATE) == DMA_PeripheralInc_Enable) || \
                                            ((STATE) == DMA_PeripheralInc_Disable))
/**
  * @}
  */

/** @defgroup DMA_memory_incremented_mode 
  * @{
  */

#define DMA_MemoryInc_Enable               ((uint32_t)0x00000080)
#define DMA_MemoryInc_Disable              ((uint32_t)0x00000000)
#define IS_DMA_MEMORY_INC_STATE(STATE) (((STATE) == DMA_MemoryInc_Enable) || \
                                        ((STATE) == DMA_MemoryInc_Disable))
/**
  * @}
  */

/** @defgroup DMA_peripheral_data_size 
  * @{
  */

#define DMA_PeripheralDataSize_Byte        ((uint32_t)0x00000000)
#define DMA_PeripheralDataSize_HalfWord    ((uint32_t)0x00000100)
#define DMA_PeripheralDataSize_Word        ((uint32_t)0x00000200)
#define IS_DMA_PERIPHERAL_DATA_SIZE(SIZE) (((SIZE) == DMA_PeripheralDataSize_Byte) || \
                                           ((SIZE) == DMA_PeripheralDataSize_HalfWord) || \
                                           ((SIZE) == DMA_PeripheralDataSize_Word))
/**
  * @}
  */

/** @defgroup DMA_memory_data_size 
  * @{
  */

#define DMA_MemoryDataSize_Byte            ((uint32_t)0x00000000)
#define DMA_MemoryDataSize_HalfWord        ((uint32_t)0x00000400)
#define DMA_MemoryDataSize_Word            ((uint32_t)0x00000800)
#define IS_DMA_MEMORY_DATA_SIZE(SIZE) (((SIZE) == DMA_MemoryDataSize_Byte) || \
                                       ((SIZE) == DMA_MemoryDataSize_HalfWord) || \
                                       ((SIZE) == DMA_MemoryDataSize_Word))
/**
  * @}
  */

/** @defgroup DMA_circular_normal_mode 
  * @{
  */

#define DMA_Mode_Circular                  ((uint32_t)0x00000020)
#define DMA_Mode_Normal                    ((uint32_t)0x00000000)
#define IS_DMA_MODE(MODE) (((MODE) == DMA_Mode_Circular) || ((MODE) == DMA_Mode_Normal))
/**
  * @}
  */

  /** @defgroup DMA_priority_level 
  * @{
  */

#define DMA_Priority_VeryHigh              ((uint32_t)0x00003000)
#define DMA_Priority_High                  ((uint32_t)0x00002000)
#define DMA_Priority_Medium                ((uint32_t)0x00001000)
#define DMA_Priority_Low                   ((uint32_t)0x00000000)
#define IS_DMA_PRIORITY(PRIORITY) (((PRIORITY) == DMA_Priority_VeryHigh) || \
                                   ((PRIORITY) == DMA_Priority_High) || \
                                   ((PRIORITY) == DMA_Priority_Medium) || \
                                   ((PRIORITY) == DMA_Priority_Low))
/**
  * @}
  */

/** @defgroup DMA_memory_to_memory 
  * @{
  */

#define DMA_M2M_Enable                     ((uint32_t)0x00004000)
#define DMA_M2M_Disable                    ((uint32_t)0x00000000)
#define IS_DMA_M2M_STATE(STATE) (((STATE) == DMA_M2M_Enable) || ((STATE) == DMA_M2M_Disable))

/**
  * @}
  */

//\Libraries\STM32F10x_StdPeriph_Driver_v3.5\inc\stm32f10x_dma.h
```

## DMA_Init()
```c
/**
  * @brief  Initializes the DMAy Channelx according to the specified
  *         parameters in the DMA_InitStruct.
  * @param  DMAy_Channelx: where y can be 1 or 2 to select the DMA and 
  *   x can be 1 to 7 for DMA1 and 1 to 5 for DMA2 to select the DMA Channel.
  * @param  DMA_InitStruct: pointer to a DMA_InitTypeDef structure that
  *         contains the configuration information for the specified DMA Channel.
  * @retval None
  */
void DMA_Init(DMA_Channel_TypeDef* DMAy_Channelx, DMA_InitTypeDef* DMA_InitStruct)

//\Libraries\STM32F10x_StdPeriph_Driver_v3.5\inc\stm32f10x_dma.h
```

## DMA_Cmd()
```c
/**
  * @brief  Enables or disables the specified DMAy Channelx.
  * @param  DMAy_Channelx: where y can be 1 or 2 to select the DMA and 
  *   x can be 1 to 7 for DMA1 and 1 to 5 for DMA2 to select the DMA Channel.
  * @param  NewState: new state of the DMAy Channelx. 
  *   This parameter can be: ENABLE or DISABLE.
  * @retval None
  */
void DMA_Cmd(DMA_Channel_TypeDef* DMAy_Channelx, FunctionalState NewState)

//\Libraries\STM32F10x_StdPeriph_Driver_v3.5\inc\stm32f10x_dma.h
```

## ADC_DMACmd()
```c
/**
  * @brief  Enables or disables the specified ADC DMA request.
  * @param  ADCx: where x can be 1 or 3 to select the ADC peripheral.
  *   Note: ADC2 hasn't a DMA capability.
  * @param  NewState: new state of the selected ADC DMA transfer.
  *   This parameter can be: ENABLE or DISABLE.
  * @retval None
  */
void ADC_DMACmd(ADC_TypeDef* ADCx, FunctionalState NewState)

//\Libraries\STM32F10x_StdPeriph_Driver_v3.5\src\stm32f10x_adc.c
```

## SystemInit()
```c
/** @addtogroup STM32F10x_System_Private_Functions
  * @{
  */

/**
  * @brief  Setup the microcontroller system
  *         Initialize the Embedded Flash Interface, the PLL and update the 
  *         SystemCoreClock variable.
  * @note   This function should be used only after reset.
  * @param  None
  * @retval None
  */
void SystemInit (void)
{
  /* Reset the RCC clock configuration to the default reset state(for debug purpose) */
  /* Set HSION bit */
  RCC->CR |= (uint32_t)0x00000001;

  /* Reset SW, HPRE, PPRE1, PPRE2, ADCPRE and MCO bits */
#ifndef STM32F10X_CL
  RCC->CFGR &= (uint32_t)0xF8FF0000;
#else
  RCC->CFGR &= (uint32_t)0xF0FF0000;
#endif /* STM32F10X_CL */   
  
  /* Reset HSEON, CSSON and PLLON bits */
  RCC->CR &= (uint32_t)0xFEF6FFFF;

  /* Reset HSEBYP bit */
  RCC->CR &= (uint32_t)0xFFFBFFFF;

  /* Reset PLLSRC, PLLXTPRE, PLLMUL and USBPRE/OTGFSPRE bits */
  RCC->CFGR &= (uint32_t)0xFF80FFFF;

#ifdef STM32F10X_CL
  /* Reset PLL2ON and PLL3ON bits */
  RCC->CR &= (uint32_t)0xEBFFFFFF;

  /* Disable all interrupts and clear pending bits  */
  RCC->CIR = 0x00FF0000;

  /* Reset CFGR2 register */
  RCC->CFGR2 = 0x00000000;
#elif defined (STM32F10X_LD_VL) || defined (STM32F10X_MD_VL) || (defined STM32F10X_HD_VL)
  /* Disable all interrupts and clear pending bits  */
  RCC->CIR = 0x009F0000;

  /* Reset CFGR2 register */
  RCC->CFGR2 = 0x00000000;      
#else
  /* Disable all interrupts and clear pending bits  */
  RCC->CIR = 0x009F0000;
#endif /* STM32F10X_CL */
    
#if defined (STM32F10X_HD) || (defined STM32F10X_XL) || (defined STM32F10X_HD_VL)
  #ifdef DATA_IN_ExtSRAM
    SystemInit_ExtMemCtl(); 
  #endif /* DATA_IN_ExtSRAM */
#endif 

  /* Configure the System clock frequency, HCLK, PCLK2 and PCLK1 prescalers */
  /* Configure the Flash Latency cycles and enable prefetch buffer */
  SetSysClock();

#ifdef VECT_TAB_SRAM
  SCB->VTOR = SRAM_BASE | VECT_TAB_OFFSET; /* Vector Table Relocation in Internal SRAM. */
#else
  SCB->VTOR = FLASH_BASE | VECT_TAB_OFFSET; /* Vector Table Relocation in Internal FLASH. */
#endif 
}
//\Libraries\CMSIS\DeviceSupport\system_stm32f10x.c
```