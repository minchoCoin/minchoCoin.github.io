---
title: "Bluetooth Communication and Soldering"
last_modified_at: 2023-11-05T00:28:12+09:00
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
- Bluetooth 모듈(FB755AC)를 이용한 스마트폰과의 통신
- 기판 납땜을 통한 보드와 모듈 연결

# 실험 원리
## 블루투스
블루투스는 ISM(공업용, 과학용, 의료용 등으로 사용하는 고주파 설비) 방식으로써 2402~2480MHz의 총 79개의 채널을 사용한다. 주파수 대역을 동시에 사용하면 충돌할 수 있기 때문에 블루투스는 ‘주파수 호핑’ 방식을 사용한다. 주파수 호핑 팡식이란, 채널을 일정 시간 간격으로 이동하여 패킷을 조금씩 전송하는 방법이다. 블루투스는 할당된 79개 채널을 초당 1600번 호핑한다.

블루투스는 마스터(master) 역할와 슬레이브(slave) 역할로 나누어져있다. Master는 inquiry(검색) 및 Page(연결요청)역할을 하며 Slave는 Inquiry Scan(검색 대기) 및 Page Scan(연결 대기)를 수행한다. Master가 주변 Slave를 찾으면 Slave는 자신의 정보를 Master에게 송신하고, Slave의 정보가 Master와 일치하면 상호 연결이 이루어지고 데이터 전송이 가능하게 된다.

블루투스 프로파일은 어플리케이션 관점에서 블루투스 기기의 기능별 성능을 정하는 사양이고, 블루투스 기기가 다른 블루투스 기기와 통신하는데 사용하는 특성을 규정한다. 다양한 프로파일이 존재하며 예를 들어 SPP(Serial Port Profile)은 RS232 시리얼 케이블 에뮬레이션을 위한 블루투스 기기에 사용되는 프로파일이며 유선 RS232 케이블이 연결된 것처럼 통신을 할 수 있다.

UUID(Universally Unique Identifier)는 네트워크 상에서 서로 다른 개체들을 구별하기 위한 128비트 고유 식별자이며, 서비스 종류를 구분하기 위해 사용된다.
## FB755AC 블루투스 모듈

![FB755AC PIN Assign](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/af59d83c-ce7d-4bfe-af70-be0810496bf1)

(그림1 FB755AC PIN Assign)

STATUS port는 이 블루투스 모듈의 상태를 모니터링하게 해주며, 블루투스 무선 구간의 연결이 원활하게 이루어져 두 디바이스가 통신이 가능한 상태일 때 LOW를 유지한다. 만약 블루투스가 연결 대기, 연결 시도, 장치 검색할 경우 LOW, HIGH를 반복한다.

FACTORY RESET은 블루투스 모듈의 설정값을 최초 상태로 변경하고자 할 때 사용하며, CONFIG SELECT에 HIGH신호를 입력한 상태에서 모듈에 전원을 인가한 후 2초 이상 LOW signal을 FACTORY RESET에 입력하면 설정값이 최초 상태로 변경된다.

UART_CTS(UART Clear To Send), UART_RTS(UART Ready To Send), UART_DTR(UART Data Terminal Ready), UART_DSR(UART Data Set Ready)은 1:1 통신에서 흐름제어에 사용된다.

STREAM CONTROL, STREAM STATUS, MESSAGE CONTROL, MESSAGE STATUS는 1:N 통신을 하기 위해 연결하는 핀이다. 1:1 통신을 할 때는 연결하지 않아도 된다.

CONNECTION CHECK는 1:N 통신에서 사용되며 Slave에서 설정된 연결 수 만큼 Master 장치가 연결되면 LOW가 되고, 하나라도 연결이 해지되면 HIGH로 변경된다. 

CONFIG SELECT에 HIGH를 입력한 채로 전원을 켜면 설정 모드로 진입하며, 설정 모드에서 Device name, Pincode, Conncetion mode 등을 설정할 수 있다.

VCC, GND는 모듈에 전원을 인가하며, 각각 보드의 3.3v, GND 포트와 연결한다.

RXD, TXD는 보드와 UART 통신을 하기 위한 포트이며, RXD는 보드의 TX, TXD는 보드의 RX 포트와 연결한다.

### 블루투스 모듈 설정
CONFIG SELECT에 HIGH를 입력한 채로 전원을 켜면 설정 모드로 진입하고, 다양한 요소를 설정할 수 있다.
DEVICE NAME은 장치 이름을 설정할 수 있으며, 최대 12문자까지 입력할 수 있다. 설정한 장치가 SLAVE로 동작하고 MASTER에서 이름을 요청하면 DEVICE NAME에 저장된 문자열을 넘겨준다.

ROLE은 MASTER와 SLAVE중 하나로 선택할 수 있다.

REMOTE BD ADDRESS는 마지막으로 연결된 블루투스 장치의 MAC address이다. 연결이 한 번도 이루어지지 않았으면 ‘000000000000’으로 표시된다.

AUTHENTICATION은 페어링에 필요한 비밀 키를 서로 분배했는지를 확인하는 절차이다. PIN CODE는 AUTHENTICATION에서 사용하는 비밀 키의 시드 값이다.

CONNECTION MODE는 CONNECTION MODE1, CONNECTION MODE2, CONNECTION MODE3, CONNECTION MODE4로 설정할 수 있다. 

예를 들어 CONNECTION MODE2에서는 SLAVE에서 항상 검색대기 및 연결 대기를 하며 Pin code가 동일한 Master가 연결을 요청하면 연결이 이루어진다. CONNECTION MODE4에서는 AT 명령어 대기 상태로서 전원이 인가되면 명령어 대기를 하고 있다. AT 명령어 중 AT+BTSCAN은 블루투스 장치에게 검색 대기와 연결대기를 하도록 한다.

UART CONFIG는 UART 통신을 설정할 수 있으며, BAUDRATE(1200~230400bps), DATA BIT(8bit), PARITY BIT(NONE, ODD, EVEN), STOP BIT(1bit, 2bit)를 설정할 수 있다.

OPERATION MODE는 OPERATION MODE0와 OPERATION MODE1, OPERATION MODE2로 설정할 수 있으며, OPERATION MODE0는 기존의 1:1 방식의 통신이고 OPERATION MODE1은 1:N 모니터링 방식으로, 일정한 간격에 한번씩 데이터를 송수신한다. OPERATION MODE2는 1:N 선별적 양방향 통신방식으로, 내부흐름제어를 이용한다.

![그림 2 FB755AC 설정](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/9e4fa4fe-ce68-457c-a304-2f884e0733b6)

(그림 2 FB755AC 설정)

## 납땜
납땜은 전자 부품간의 전기적 연결을 제공하고 부품 고정에 도움을 준다. 또한 신호의 왜곡을 최소화하고 전달 효율성을 향상시킨다.

블루투스 모듈을 핀 소켓에 끼운 채로 납땜하면 안되고, 인두기의 끝이 더러워 지지않게 인두 팁 크리너를 잘 사용해야한다. 핀 소켓과 헤더, 그리고 만능 기판에 인두를 오래 접촉하면 안 된다.

납땜 방법은 다음과 같다. 먼저 접합 부분을 예열하고, 땜납을 인두에 대고 녹인다. 그리고 적당량의 땜납이 녹으면 땜납을 먼저 분리하고, 땜납이 동판에 완전히 녹아 붙으면 인두를 분리한다.

# 실험 과정
## 블루투스와 보드 연결
VCC, GND핀을 보드의 3.3V와 GND에 연결하고, STATUS 핀과 DCD핀은 LED를 연결하여 상태를 확인할 수 있도록 한다. RXD와 TXD 핀은 보드의 USART2 TX와 USART2 RX에 각각 연결한다. CONFIG SELECT핀은 설정이 필요할 때 3.3V를 인가한다. 블루투스 모듈을 꽂는 소켓과 LED, 저항은 만능 기판에 납땜한다.

## 코드 작성
1. 실험에 필요한 GPIOA와 USART1, USART2 및 AFIO clock을 활성화한다.
2.	USART1 TX(PA9), USART1 RX(PA10), USART2 TX(PA2), USART RX(PA3)의 핀모드와 속도를 설정한다.
3.	USART1과 USART2의 BaudRate, WordLength, StopBIts, HardwareFlowControl, Parity bit, mode를 설정한다.
4.	USART1 인터럽트와 USART2 인터럽트의 우선순위를 설정한다.
5.	USART1 인터럽트 핸들러를 만든다. USART1, 즉 Putty에서 데이터를 받았을 때, 받은 데이터를 USART2로 보낸다.
6.	USART2 인터럽트 핸들러를 만든다. USART2, 즉 블루투스 모듈에서 데이터를 받았을 때, 받은 데이터를 USART1으로 보낸다.

## 블루투스 설정
Bluetooth의 CONFIG SELECT 핀에 3.3V를 인가한 상태에서 보드를 켜 설정을 진행한다. DEVICE NAME, Pincode를 설정하고, CONNECTION MODE는 mode0, ROLE은 SLAVE로 설정한다. UART CONFIG는(9600,8,n,1)로 설정한다. 각각 9600 Baud Rate, 8 data bit, no parity bit, 1 stop bit를 의미한다.

## 블루투스 연결 대기
CONFIG SELECT 핀에 3.3V 입력을 해제하고 보드 전원을 껐다 켜 AT 명령어 대기 모드로 진입한다. 여기서 AT+BTSCAN 커맨드를 입력하면 연결 대기에 들어간다.

## 스마트폰와 블루투스 연결
스마트폰을 Serial Bluetooth Terminal 앱을 통해 보드와 연결하여 통신한다.

# 코드 작성
## RCC 설정

```c
void RCC_Configure(void)
{  
    // TODO: Enable the APB2 peripheral clock using the function 'RCC_APB2PeriphClockCmd'
   
   /* USART1, USART2 TX/RX port clock enable */
   RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);
   /* USART1, USART2 clock enable */
    RCC_APB1PeriphClockCmd(RCC_APB2Periph_USART1, ENABLE);
    RCC_APB2PeriphClockCmd(RCC_APB1Periph_USART2, ENABLE);
   /* Alternate Function IO clock enable */ 
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_AFIO, ENABLE);
}

```
RCC_APB1PeriphClockCmd와 RCC_APB2PeriphClockCmd를 이용하여 TX, RX핀 이있는 GPIOA와 USART1, USART2를 활성화한다. 그리고 alternative function push-pull로 설정하기 위해 AFIO 핀에도 클럭을 인가한다. RCC_APB1PeriphClockCmd로 USART1, RCC_APB2PeriphClockCmd로 GPIOA, USART2, AFIO에 클럭을 인가한다.

## GPIO 설정
```c

void GPIO_Configure(void)
{
    GPIO_InitTypeDef GPIO_InitStructure;
    
      // TODO: Initialize the GPIO pins using the structure 'GPIO_InitTypeDef' and the function 'GPIO_Init'

    /* USART1 pin setting */
    //TX
    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_9;
    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AF_PP;
    GPIO_Init(GPIOA, &GPIO_InitStructure);
   //RX
    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_10;
    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IPD;
    GPIO_Init(GPIOA, &GPIO_InitStructure);
    /* USART2 pin setting */
    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_2;
    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AF_PP;
    GPIO_Init(GPIOA, &GPIO_InitStructure);
   //RX
    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_3;
    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IPD;
    GPIO_Init(GPIOA, &GPIO_InitStructure);
}

```
GPIO_InitTypeDef 구조체를 활용하여 TX(USART1은 PA9, USART2는 PA2)은 Alternative function output push-pull, RX(USART1은 PA10, USART2는 PA3)는 Input with pull-up/pull-down으로 설정하고 속도는 모두 50MHz로 설정한다. 그리고 GPIO_Init을 이용하여 설정값이 들어있는 구조체를 실제로 레지스터에 반영한다.

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
   USART1_InitStructure.USART_HardwareFlowControl = USART_HardwareFlowControl_None; 
   USART1_InitStructure.USART_Parity = USART_Parity_No;
   USART1_InitStructure.USART_Mode = USART_Mode_Rx | USART_Mode_Tx;
   USART_Init(USART1, &USART1_InitStructure); //          
   
   // TODO: Enable the USART1 RX interrupts using the function 'USART_ITConfig' and the argument value 'Receive Data register not empty interrupt'
    USART_ITConfig(USART1, USART_IT_RXNE, ENABLE); // Rx -> interrupt enable
}
```
먼저 USART_Cmd로 USART1를 활성화한다.

그리고 USART_InitTypeDef 구조체를 사용하여 BaudRate는 9600, WordLengths는 8bit, StopBIt는 1bit, HardwareFlowControl은 사용하지 않고, Parity bit는 사용하지 않는다. 그리고 RX와 TX모두 사용한다. USART_Init 함수를 활용하여 설정값이 들어있는 구조체를 실제 레지스터에 반영한다. 마지막으로, USART_ITConfig 함수를 사용하여 USART1에서RX를 통해 데이터가 들어왔을 때 인터럽트가 발생하도록(USART_IT_RXNE) 설정한다.

## USART2 설정
```c
void USART2_Init(void)
{
    USART_InitTypeDef USART2_InitStructure;

   // Enable the USART2 peripheral
   USART_Cmd(USART2, ENABLE);
   
   // TODO: Initialize the USART using the structure 'USART_InitTypeDef' and the function 'USART_Init'
   USART2_InitStructure.USART_BaudRate = 9600;
   USART2_InitStructure.USART_WordLength = USART_WordLength_8b;
   USART2_InitStructure.USART_StopBits = USART_StopBits_1;
   USART2_InitStructure.USART_HardwareFlowControl = USART_HardwareFlowControl_None; 
   USART2_InitStructure.USART_Parity = USART_Parity_No;
   USART2_InitStructure.USART_Mode = USART_Mode_Rx | USART_Mode_Tx;
   USART_Init(USART2, &USART2_InitStructure); //         
   
   // TODO: Enable the USART2 RX interrupts using the function 'USART_ITConfig' and the argument value 'Receive Data register not empty interrupt'
   USART_ITConfig(USART2, USART_IT_RXNE, ENABLE); 
}

```
먼저 USART_Cmd로 USART2 활성화한다.

그리고 USART_InitTypeDef 구조체를 사용하여 BaudRate는 9600, WordLengths는 8bit, StopBIt는 1bit, HardwareFlowControl은 사용하지 않고, Parity bit는 사용하지 않는다. 그리고 RX와 TX모두 사용한다. USART_Init 함수를 활용하여 설정값이 들어있는 구조체를 실제 레지스터에 반영한다. 마지막으로, USART_ITConfig 함수를 사용하여 USART2서 RX를 통해 데이터가 들어왔을 때 인터럽트가 발생하도록(USART_IT_RXNE) 설정한다.

## NVIC 설정
```c
void NVIC_Configure(void) {

    NVIC_InitTypeDef NVIC_InitStructure;
   
    // TODO: fill the arg you want
    NVIC_PriorityGroupConfig(NVIC_PriorityGroup_0);

    // USART1
    // 'NVIC_EnableIRQ' is only required for USART setting
    NVIC_EnableIRQ(USART1_IRQn);
    NVIC_InitStructure.NVIC_IRQChannel = USART1_IRQn;
    NVIC_InitStructure.NVIC_IRQChannelPreemptionPriority = 0; // TODO
    NVIC_InitStructure.NVIC_IRQChannelSubPriority = 0; // TODO
    NVIC_InitStructure.NVIC_IRQChannelCmd = ENABLE;
    NVIC_Init(&NVIC_InitStructure);

    // USART2
    // 'NVIC_EnableIRQ' is only required for USART setting
    NVIC_EnableIRQ(USART2_IRQn);
    NVIC_InitStructure.NVIC_IRQChannel = USART2_IRQn;
    NVIC_InitStructure.NVIC_IRQChannelPreemptionPriority = 0; // TODO
    NVIC_InitStructure.NVIC_IRQChannelSubPriority = 0; // TODO
    NVIC_InitStructure.NVIC_IRQChannelCmd = ENABLE;
    NVIC_Init(&NVIC_InitStructure);
}

```
NVIC_PriorityGroupConfig는 group0으로 설정한다. Group0은 PreemptionPriority에 0bit, SubPriority에 4bit를 할당한다.

NVIC_EnableIRQ를 통해 NVIC interrupt controller에 USART1와 USART2를 활성화한다. 그리고 NVIC_InitTypeDef 구조체를 사용하여 USART1과 USART2의 NVIC 인터럽트 PreemptionPriority와 Subpriority를 각각 0으로 설정한다. 마지막으로 NVIC_Init을 사용하여 설정값이 들어있는 구조체를 레지스터에 반영한다.

## 인터럽트 핸들러 작성
```c
void USART1_IRQHandler() {
    uint16_t word;
    if(USART_GetITStatus(USART1,USART_IT_RXNE)!=RESET){
        // the most recent received data by the USART1 peripheral
        word = USART_ReceiveData(USART1);

        // TODO implement
        USART_SendData(USART2, word);

        // clear 'Read data register not empty' flag
       USART_ClearITPendingBit(USART1,USART_IT_RXNE);
    }
}

void USART2_IRQHandler() {
    uint16_t word;
    if(USART_GetITStatus(USART2,USART_IT_RXNE)!=RESET){
        // the most recent received data by the USART2 peripheral
        word = USART_ReceiveData(USART2);

        // TODO implement
        USART_SendData(USART1, word);

        // clear 'Read data register not empty' flag
       USART_ClearITPendingBit(USART2,USART_IT_RXNE);
    }
}
```
USART1과 USART2 인터럽트 핸들러 모두 RX를 통해 들어올 데이터가 있어 인터럽트가 발생했다면 데이터를 받고, 받은 데이터를 각각 USART2와 USART1으로 보내고 Pendingbit를 초기화한다.

# 실험 결과
![그림 3 스마트폰과 보드가 데이터를 서로 주고받는다1](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/071814dc-690c-4787-b431-26290c48f6ce)
![그림 3 스마트폰과 보드가 데이터를 서로 주고받는다2](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/0f7b7e67-0b4d-4425-bb9f-487d75d429a1)

(그림 3 스마트폰과 보드가 데이터를 서로 주고받는다.)

USART1과 연결된 Putty에서 데이터를 입력했을 때 블루투스 모듈과 연결된 스마트폰에서 해당 데이터가 출력되었다. 반대로 블루투스 모듈과 연결된 스마트폰에서 데이터를 보냈을 때 Putty에 해당 데이터가 출력되었다.

# 알게 된점
블루투스 모듈을 사용하여 무선으로 다른 장치와 통신하는 방법을 알게 되었다. 또한 납땜을 통해 부품 고정을하고 전달 효율성을 향상시킨다는 것을 알게 되었다.

# 참고 문헌
- STM32F107 Datasheet 
    https://www.st.com/resource/en/datasheet/stm32f107vc.pdf 
- STM32F107 Reference Manual 
    https://www.st.com/resource/en/reference_manual/rm0008- stm32f101xxstm32f102xxstm32f103xx-stm32f105xx-and-stm32f107xx-advanced armbased-32bitmcusstmicroelectronics.pdf 
- STM32F107VCT6 schematic
- FB755AC 사용자가이드 
	https://drive.google.com/file/d/1Br7Evx_k58qSpBgM7wPoJlqjDk6qLnjb/view
-	FB755AC 환경설정 세부설명
    https://drive.google.com/file/d/17wZyuzfTGCBFkGr4TZgauAfPMvzuwS7k/view
-	FB755AC AT 명령어 세부설명 및 사용방법
    https://drive.google.com/file/d/1hHo5eZwaaROHi40HOtSzdTMZp-Yqioum/view
-	FB755AC 1:N 데이터 송수신방법
    https://drive.google.com/file/d/1Bhg-uFtnYlDARe-9ZtSjOZb3oetiFIlp/view
