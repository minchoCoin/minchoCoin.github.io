---
title: "Turn on/off motor using STM32F107VC board,relay module, button and register"
last_modified_at: 2023-10-05T23:53:12+09:00
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
 - 스캐터 파일의 이해 및 플래시 프로그래밍
 - 릴레이 모듈의 이해 및 임베디드 펌웨어를 통한 동작

# 세부 실험 내용
 - Datasheet 및 Reference Manual을 참고하여 해당 레지스터 및 주소에 대한 설정 이해
 - 스캐터 파일을 통해 플래시 메모리에 프로그램 다운로드
 - 플래시 메모리에 올려진 프로그램 정상적인 동작 확인
 - 스캐터 파일(.icf)을 통해 원하는 메모리 위치에 프로그램 다운로드 확인
    - ROM 크기 0x80000
    - RAM 크기 0x8000
 - **릴레이 모듈을 활용한 모터**
    - **버튼1(KEY1) : 1번 및 2번 모터 1초 회전 후 정지**
    - **버튼2(KEY2) : 1번 모터 1초 회전 후 정지**
    - **버튼3(KEY3) : 2번 모터 1초 회전 후 정지**

# 실험 방법
1. 릴레이 모듈 COM과 모터 단자 하나를 연결하고, NO에 VCC를 연결하며, 다른 모터 단자 하나를 GND와 연결한다.
2. 릴레이 모듈의 VCC와 GND를 각각 3.3v와 GND에 연결하고, 2개의 릴레이 모듈의 2개의 IN을 각각 PD4와 PD7에 연결한다.
3. icf 파일에 작성해야할 부분(ROM_start, ROM_end, RAM_start, RAM_end)을 작성하여 업로드한다..
4. RCC(reset and clock control)를 이용하여 각 포트에 clock 을 인가한다.
5. 각 포트에서 쓸 포트 번호를 설정한다(Port Configuration Low/High register 이용).
6. Port Input Data Register 를 검사하여 버튼이 눌렸는지 확인하고, 버튼이 눌렸으면 Port bit set/reset register 를 사용하여 릴레이 모듈에 신호를 입력한다.

# 실험 원리

## 스위치 연결에서 Floating, Pull up/ Pull Down
### Floating input
1이나 0이 아닌 미결정 상태로, 스위치가 연결되지 않았을 때는 VCC와 GND에 모두 연결되어있지 않아 입력값이 무엇인지 알 수 없는 상태이다.

![Floating input](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/cf562849-c622-4548-9a49-44416c33d21a)

(사진1 : Floating input)

### Pull up 저항 연결
VCC와 input pin 사이를 저항으로 연결하여 플로팅 현상을 해결하는 방법으로, 논리적으로 1(HIGH)을 유지하다가 버튼이 눌려질 때 0(LOW)으로 변경된다.

![pull up 저항](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/fd65d596-63a1-48bb-87b4-68807cd5df65)

(사진2 : pull up 저항)

### Pull down 저항 연결
GND와 input pin 사이를 저항으로 연결하여 플로팅 현상을 해결하는 방법으로, 논리적으로 0(LOW)를 유지하다가 버튼이 눌려질 때 1(HIGH)로 변경된다.

![pull down 저항](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/600fdcf3-d07b-4780-9a42-54ec8a5bf7ce)

(사진3 : pull down 저항)

## 스캐터 파일
실행시킬 바이너리 이미지가 메모리에 로드될 때 바이너리 이미지의 어떤 영역이 어느 주소에 어느 크기만큼 배치되야할 지 작성한 파일이다.
### 분산 적재
분산 적재(scatter loading)는 꺼내기의 한 형식으로, 판독모듈의 제어 섹션을 주기억장치 가운데 각각의 장소에 적재하는 것이다.
### 스캐터 파일의 구조
![building blocks for an image](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/d050a0f3-5630-403e-bc7a-287a806d1e38)
(사진4: building blocks for an image)

(사진출처 : [https://developer.arm.com/documentation/dui0041/c/Linker/Building-blocks-for-objects-and-images](https://developer.arm.com/documentation/dui0041/c/Linker/Building-blocks-for-objects-and-images))

- input section
    - RO(readonly: code, constant data)
    - RW(readwrite: 초기화된 global data)
    - ZI(zero initialized: 0으로 초기화된, 초기화되지않은 global data)
    - 위의 3개 중 하나의 속성을 갖는 집합
- output section
    - input section들 중에서 같은 속성을 갖는 것들을 묶어놓은것
- Region
    - Output section을 묶어놓은 것

![Simple scatter-loaded memory map](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/15b3982e-dd7e-4693-90d0-6e4f95a48dfa)
(사진5 : Simple scatter-loaded memory map)

(사진출처: [https://developer.arm.com/documentation/dui0803/a/Using-scatter-files/Images-with-a-simple-memory-map](https://developer.arm.com/documentation/dui0803/a/Using-scatter-files/Images-with-a-simple-memory-map))

 - Load View: Flash에 실행 image가 담겨있을 때의 형태
 - Execution View: Flash에 실행 image가 실행될때의 형태

### STM32 보드 스캐터 파일
![STM32F107VC memory map](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/eb9150fb-6714-4826-afc0-c480c67cfe97)

(사진6: STM32F107VC memory map)

또한 아래 코드는 IAR embedded workbench에 있는 STM32F107VC의 스캐터 파일이다.
```c
//C:\Program Files\IAR Systems\Embedded Workbench 9.2\arm\config\linker\ST\stm32f107xC.icf
/*###ICF### Section handled by ICF editor, don't touch! ****/
/*-Editor annotation file-*/
/* IcfEditorFile="$TOOLKIT_DIR$\config\ide\IcfEditor\cortex_v1_0.xml" */
/*-Specials-*/
define symbol __ICFEDIT_intvec_start__ = 0x08000000;
/*-Memory Regions-*/
define symbol __ICFEDIT_region_ROM_start__ = 0x08000000;
define symbol __ICFEDIT_region_ROM_end__   = 0x0803FFFF;
define symbol __ICFEDIT_region_RAM_start__ = 0x20000000;
define symbol __ICFEDIT_region_RAM_end__   = 0x2000FFFF;
/*-Sizes-*/
define symbol __ICFEDIT_size_cstack__ = 0x1000;
define symbol __ICFEDIT_size_heap__   = 0x1000;
/**** End of ICF editor section. ###ICF###*/

define memory mem with size = 4G;
define region ROM_region   = mem:[from __ICFEDIT_region_ROM_start__   to __ICFEDIT_region_ROM_end__];
define region RAM_region   = mem:[from __ICFEDIT_region_RAM_start__   to __ICFEDIT_region_RAM_end__];

define block CSTACK    with alignment = 8, size = __ICFEDIT_size_cstack__   { };
define block HEAP      with alignment = 8, size = __ICFEDIT_size_heap__     { };

initialize by copy { readwrite };

place at address mem:__ICFEDIT_intvec_start__ { readonly section .intvec };

place in ROM_region   { readonly };
place in RAM_region   { readwrite,
                        block CSTACK, block HEAP };

```
stm32f107의 memory map과 스캐터파일에서 보면 ROM_start와 ROM_end의 주소는 memory map에 있는 flash의 주소와 같고, RAM_start와 RAM_end의 주소는 memory map의 SRAM의 주소와 같은 것을 알 수 있다.

또한 위의 icf 스캐터파일에서, intvec(interrupt vector) 시작주소를 정의하고, 지역변수와 함수매개변수를 저장하는 cstack과 동적 메모리할당에 필요한 heap의 크기가 정의되어있다. 그리고 메모리의 크기를 4G로 정의하고, ROM 영역에는 읽기전용 데이터만, RAM 영역에는 readwrite데이터와 Cstack, heap도 함께 배치하는 것을 알 수 있다.

## 릴레이 모듈
![relay module](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/32fe9320-92f9-4a19-9ca7-b735591c61f3)

(사진7: 릴레이 모듈)
릴레이 모듈은 전자석과 전기적 신호를 사용하여 기계적 스위치를 제어하는 장치이다. 아래 그림에서 GND와 VCC 는 각각 그라운드와 3.3V에 연결하고, IN에는 신호를 입력한다.
- NC(Normally Close): 평상시에 Close(연결됨), IN이 HIGH시 Open(연결 끊김)
- NO(Normally Open): 평상시에 Open(연결 끊김), IN이 HIGH시 Close(연결됨)
- COM(Common): 평상시에 NC와 연결되어있다가, 전류가 흐르면 NO와 연결된다.

따라서 외부 기기는 NC 또는 NO 중 하나와 COM에 연결되고, NC와 COM에 연결시 신호가 LOW일 때 동작하므로 Pull-up 방식이고, NO와 COM에 연결시 신호가 HIGH일 때 동작하므로 Pull-down 방식이다.

## 폴링
비동기적 이벤트를 처리하는 방식 중 폴링 방식은 하드웨의 변화를 지속적으로 읽어들여 변화를 알아채는 방법으로, 신호를 판단하기 위해 지속적으로 확인해야한다. 구현은 간단하지만 상대적으로 느린 응답과 주기적으로 리소스를 소모한다는 점이 단점이다.

# 코드 작성

## icf파일 작성 및 적용
제공받은 스캐터파일에서 다음부분이 비어있다.
```c
define symbol __ICFEDIT_region_ROM_start__ = // TODO
define symbol __ICFEDIT_region_ROM_end__   = // TODO
define symbol __ICFEDIT_region_RAM_start__ = // TODO
define symbol __ICFEDIT_region_RAM_end__   = // TODO
```
이를 사진6의 memory map과 세부실험내용의 ROM과 RAM 크기를 이용하여 아래와 같이 채운다. Memory map의 rom과 ram 시작 주소를 ROM_start와 RAM_start에 넣고, 각 시작주소에 크기를 더한 값을 ROM_end와 RAM_end에 대입한다.

```c
define symbol __ICFEDIT_region_ROM_start__ = 0x08000000;
define symbol __ICFEDIT_region_ROM_end__   = 0x08080000;
define symbol __ICFEDIT_region_RAM_start__ = 0x20000000;
define symbol __ICFEDIT_region_RAM_end__   = 0x20008000;
```
작성한 icf 파일을 프로젝트 option에서 Liner-Config-Override default-...를 눌러 업로드한다.
![icf 파일 업로드 방법](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/0f9861be-dd45-4c85-87c8-7489bfe2ef8f)

## stm32f10x.h 넣기
```c
#include "stm32f10x.h"
```
## 버튼의 포트와 포트번호 찾기
![button schematic of STM32F107VCT6](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/9fe1e2e1-0643-46a0-9fca-f1fa46681e57)

(사진8. STM32F107VCT6의 회로도 중 버튼 부분)

STM32F107VCT6의 Schematic을 보면

- Key1: PC4(포트 C의 4번)
- Key2: PB10(포트 B의 10번)
- Key3: PC13(포트 C의 13번)

그리고 버튼은 풀업저항으로 연결되어있다. 즉 버튼이 눌리면 0, 눌리지 않았을 때 1을 입력받는다.

## RCC와 포트의 base 주소 찾기
![STM32F107xx memory mapping](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/fa700eb6-3e56-49f0-b308-93fbef8be583)

(사진9. STM32F107xx memory mapping)

STM32 datasheet에 있는 memory mapping을 보면, RCC와 포트의 base주소는 다음과 같다.

- RCC: 0x4002 1000
- Port A: 0x4001 0800
- Port B: 0x4001 0C00
- port C: 0x4001 1000
- port D: 0x4001 1400

## RCC_APB2ENR의 주소 찾기
![STM32F105xx and STM32F107xx connectivity line block diagram 일부](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/e767c06f-3226-492b-8085-6a12a66fca87)

(사진10. STM32F105xx and STM32F107xx connectivity line block diagram 일부)

위 그림과 같이 STM32F107xx connectivity line block diagram 을 보면 GPIO는 APB2에 연결되어있다. 따라서 RCC_APB2ENR에서 각 GPIO에 clock을 인가하여야한다.

![RCC_APB2ENR 레지스터 정보](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/77c8dcc8-0728-4879-8655-c02b865f631a)
(사진11. RCC_APB2ENR 레지스터 정보)

Address가 0x18이므로, 우리가 쓸 RCC_APB2ENR의 주소는 RCC의 base주소에 0x18을 더한 값이다.

## Port Configuration Low Register의 주소 찾기
![port configuration low register 정보](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/848381af-0503-4366-82a6-66550dbcd05f)

(사진12. port configuration register low 정보)

port configuration low register는 각 포트의 0번부터 7번까지 input/output 모드를 설정하고, input일 경우 아날로그 또는 pull-up/down 등, output일 경우 push-pull/open-drain을 설정하게 된다. 나중에 port configuration low register를 설정할 때 더 자세히 기술한다.

위 사진12에서, port configuration low register의 address는 0x00으로, 각 포트의 base주소와 같다.

## Port configuration High Register의 주소 찾기
![port configuration high register 정보](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/a1bf65a6-03c0-4a9d-aa8c-4fbdee51356d)

(사진13. port configuration high register)
port configuration low register는 각 포트의 8번부터 15번까지 input/output 모드를 설정하고, input일 경우 아날로그 또는 pull-up/down 등, output일 경우 push-pull/open-drain을 설정하게 된다. 나중에 port configuration low register를 설정할 때 더 자세히 기술한다.

위 사진13에서, port configuration high register의 address는 0x04로, 각 포트의 base 주소 + 0x04와 같다.

## Port input data register의 주소 찾기
![port input data register](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/d2cfbd93-d5e6-476d-874f-dbf864d9c6d0)

(사진14. port input data register)

port input data register는 각 포트의 0번부터 15번까지 input 값이 젹혀있는 register이다. 버튼이 풀업저항이므로, 버튼이 눌렸을 때 레지스터에 0이 저장되고, 버튼이 눌리지 않았을 때 해당하는 레지스터에 1이 저장된다.

위 사진14에서, port input data register의 address는 0x08로, 각 포트의 base 주소 + 0x08과 같다.

## Port bit set/reset register의 주소 찾기

![port bit set/reset register](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/8a57a4cc-c041-4d1e-bc75-54fcca1e7af0)

(사진15. Port bit set/reset register)

포트D의 set/reset register를 이용하여 LED에 해당하는 포트의 set register를 1로 하면 LED가 켜지고, reset register를 1로 하면 LED가 꺼지게 된다. set register와 reset register가 동시에 1인 경우, set register를 우선으로 한다.

위 사진15에서, port bit set/reset register의 address는 0x10으로, 각 포트의 base 주소 + 0x10과 같다.

## 여러 주소 define하여 매크로로 사용하기
포트B는 configuration high register, 포트C는 둘다, 포트D는 configuration low register만 필요하다.

또한 port input data register의 경우 포트 B,C에 필요하고, port bit set/reset register는 포트D에 필요하다. 따라서 다음과 같이 define 할 수 있다.

```c
#include "stm32f10x.h"
/*
 - RCC:     0x4002 1000
 - Port A:  0x4001 0800
 - Port B:  0x4001 0C00
 - port C:  0x4001 1000
 - port D:  0x4001 1400
*/
//RCC_APB2ENR == 0x40021000 + 0x18
#define RCC_APB2ENR (*(volatile unsigned int*)0x40021018) // 

#define PB_CRH (*(volatile unsigned int*)0x40010C04) //PB_base_address + 0x04

#define PC_CRL (*(volatile unsigned int*)0x40011000) //PC_CRL == PC_base_address
#define PC_CRH (*(volatile unsigned int*)0x40011004) //PC_CRH == PC_base_address + 0x04

#define PD_CRL (*(volatile unsigned int*)0x40011400) //PD_CRL == PD_base_address


#define PB_IDR (*(volatile unsigned int*)0x40010C08)//PB_IDR == PB_base_address + 0x08
#define PC_IDR (*(volatile unsigned int*)0x40011008)//PC_IDR == PC_base_address + 0x08

#define PD_BSRR (*(volatile unsigned int*)0x40011410)//PD_BSRR == PD_base_address + 0x10
```
## delay 함수 작성
모터를 일정시간 동작시키기 위해 딜레이 함수를 작성한다.
```c
void delay() {
    int i;
    for (i = 0; i < 10000000; i++) {}
}

```

## 포트 번호 define 하기
```c
//IDR과 BSRR 레지스터에서 해당 핀을 확인하는 용도 및 해당핀을 set/reset하는데 쓰인다.
#define PIN_4 0x00000010
#define PIN_7 0x00000080
#define PIN_10 0x00000400
#define PIN_13 0x00002000

//사진14에서 알 수 있듯, 예를 들어 포트A의 0번 핀이 0인지 확인하려면
//(PA_IDR & PIN_0) == 0으로 확인할 수 있다.

//사진15에서 알 수 있듯, 예를 들어 BSRR 레지스터에 2번핀을 set으로 하려면
//PD_BSRR |= PIN_2로 하면된다. BSRR 레지스터 2번핀을 reset할려면 PD_BSRR |= PIN_2<<16을 하면 된다. 왜냐하면 BSRR 레지스터의 앞16비트는 reset, 뒤 16비트는 set bit이기 때문이다.
```

## RCC clock enable 하기
![RCC_APB2ENR에 대한 자세한 설명](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/728ec3f0-4fd8-4863-9940-84373586faf5)

(사진16. RCC_APB2ENR에 대한 자세한 설명)

사진16에서, 우리는 IOPB,IOPC,IOPD를 활성화해야하므로(계산 편의를 위해 IOPE도 활성화하였다.) RCC를 0b0000 0000 0000 0000 0000 0000 0011 1000으로 초기화해야한다. 이를 아스키코드로 바꾸면 0x38이므로 다음과 같이 초기화한다.
```c
RCC_APB2ENR |= 0x00000038; // Port B, C, D 에 클럭 활성화(bit 3, 4, 5)
```

## CRL, CRH 초기화
![GPIOx_CRL 정보](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/2885236a-e9aa-440e-a1ec-a403e9dcfece)

(사진17. Port configuration register low 정보)

![GPIOx_CRH 정보](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/aeec72cb-2542-4e7b-b59a-7a331d4bb4ae)

(사진18. Port configuration register high 정보)

위의 코드에서 알 수 있듯, CRL에서 0번핀을 초기화하려면 hex값에서 뒤에서 0번째 hex값을 input일 경우 8(=0b1000, 10은 CNF에서 input with pull-up/pull-down을 의미하고, 뒤의 00은 input mode를 의미한다.)로 설정하고

CRH에서 8번핀을 초기화하려면 input mode에서 hex값에서 뒤에서 0번쨰 hex값을 8로 설정하면 되고

CRL에서 0번핀을 output으로 초기화하려면 뒤에서 0번째 hex값을 1(=0b0001, 앞의 00은 General purose output push-pull을 의미하고, 뒤의 01은 output mode max speed 10mhz를 의미한다)로 설정한다.

```c
PB_CRH &= 0xfffff0ff;//PB10을 설정하기 위해 PB10의 CRH의 값을 0으로 먼저 초기화한다.
PB_CRH |= 0x00000800; // 10 00(Input with pull-up / pull down, Input mode(reset state))

PC_CRL &= 0xfff0ffff;//PC4를 설정하기 위해 PC4의 CRL값을 0으로 먼저 초기화한다.
PC_CRL |= 0x00080000; // 10 00(Input with pull-up / pull down, Input mode(reset state))

PC_CRH &= 0xff0fffff; //PC13을 설정하기 위해 PC13의 CRH의 값을 0으로 먼저 초기화한다.
PC_CRH |= 0x00800000; // 10 00(Input with pull-up / pull down, Input mode(reset state))

PD_CRL &= 0x0ff0ffff; //PD4, PD7을 설정하기 위해 각 포트의 CRL값을 0으로 먼저 초기화한다.
PD_CRL |= 0x10010000; // 00 01(General purpose output push-pull, Output mode, max speed 10MHz)
```
## IN 신호 끄기
```c
PD_BSRR |= PIN_4 << 16; // BSRR을 이용하여 PD4번 핀을 reset한다
PD_BSRR |= PIN_7 << 16; // BSRR을 이용하여 PD7번 핀을 reset한다.
```

## 무한반복문 작성
```c
 int key = 0;
  while(1) {
    // 눌렀을 때가 낮은 전압
    if (!(PC_IDR & PIN_4)){//Key1 버튼이 눌리면 PC_IDR에서 뒤에서 4번째 비트가 0이된다. 따라서 이것을 port_4와 & 하면 0이되고, key=1이 들어가게 된다.
      key = 1;
    }
    else if (!(PB_IDR & PIN_10)){// key2 버튼이 눌리지 않는다면 PC_IDR에서 뒤에서 10번째 비트가 1이된다. 따라서 이것을 port_10과 & 하면 1이 되고, key=2가 실행되지 않는다.
      key = 2;
    }
    else if (!(PC_IDR & PIN_13)){
      key = 3;
    }

    switch (key) {
    case 0:
      break;
    case 1: //  kEY1 :  1번, 2번 
     //PD_BSRR을 이용하여 포트4번과 포트7번을 set한다.
      PD_BSRR |= PIN_7;   
      PD_BSRR |= PIN_4;
      delay();
      //PD_BSRR을 이용하여 포트4번과 포트7번을 reset한다.
      PD_BSRR |= PIN_7 << 16;
      PD_BSRR |= PIN_4 << 16;
      break;
      
    case 2: // KEY2 :  1번
      //PD_BSRR을 이용하여 포트7번을 set한다.
      PD_BSRR |= PIN_7;   
      delay();
      //PD_BSRR을 이용하여 포트7번을 reset한다.
      PD_BSRR |= PIN_7 << 16;
      delay();
      break;
      
    case 3: // KEY3 :  2번
    //PD_BSRR을 이용하여 포트4번을 set한다.
      PD_BSRR |= PIN_4;   
      delay();
      //PD_BSRR을 이용하여 포트7번을 reset한다.
      PD_BSRR |= PIN_4 << 16;
      delay();
      break;
    }
    key = 0;
  }
```
# 작동 영상
{% include video id="IYnApG_BLLI" provider="youtube" %}

Key1을 누르면 모터 2개 다 작동되고, Key2를 누르면 1번 모터만, Key3를 누르면 2번모터만 작동된다.
# Reference
- STM32F107 Datasheet [https://www.st.com/resource/en/datasheet/stm32f107vc.pdf](https://www.st.com/resource/en/datasheet/stm32f107vc.pdf)

 - STM32F107 Reference Manual [https://www.st.com/resource/en/reference_manual/rm0008-stm32f101xx-stm32f102xx-stm32f103xx-stm32f105xx-and-stm32f107xx-advanced-armbased-32bit-mcus-stmicroelectronics.pdf](https://www.st.com/resource/en/reference_manual/rm0008-stm32f101xx-stm32f102xx-stm32f103xx-stm32f105xx-and-stm32f107xx-advanced-armbased-32bit-mcus-stmicroelectronics.pdf)

 - STM32F107VCT6 schematic

 - https://terms.naver.com/entry.naver?docId=836306&cid=50376&categoryId=50376

 - https://developer.arm.com/documentation/dui0041/c/Linker/Building-blocks-for-objects-and-images

 - https://developer.arm.com/documentation/dui0803/a/Using-scatter-files/Images-with-a-simple-memory-map

 - https://minchocoin.github.io/stm32f107vc/3/