---
title: "소프트웨어공학 - 요구공학"
last_modified_at: 2023-10-21T14:12:12+09:00
categories:
    - software-engineering
tags:
    - software-engineering

toc: true
toc_label: "My Table of Contents"
author_profile: true

---
# 요구는 무엇인가?
요구되는 것; 원하거나 필요한 것

문제를 해결하거나 목표를 달성하기 위해 사용자가 필요로 하는 조건 또는 능력

시스템이 어떻게 수행할 것인지에 대한 언급 없이 시스템이 무엇을 할 것인지에 대한 완전한 설명

원하는 시스템의 외부적으로 관찰 가능한 특성

# 요구사항은 다음과 같으면 안된다.
- 모호성
- 완전하지 않음
- 잘못됨
- 일치하지 않음

# 요구사항과 관련된 문제
- 고객은 무엇인 필요한지에 대해 막연한 생각만 가지고 있다.
- 개발자는 세부사항은 나중에 채우겠다는 생각으로 '막연한 생각'을 진행한다.
- 고객이 계속해서 요구사항을 변경한다.
- 개발자는 이러한 변경으로 인해 발생하는 오류에 고통받는다.
- 요구사항을 잘못이해하는 것은 프로젝스 실패의 주요 원인이다.
- 복잡성 : 문제를 이해하기 어렵고, 응용프로그램 도메인을 이해해야한다.
- 요구사항의 변화 : 변화는 무시할 수 없다. 최종 사용자는 실제 프로그램을 얻기 전 까지 자신이 원하는 것을 알지 못한다.
- 커뮤니케이션 : 다양한 배경을 가진 사람들의 의사소통

# 요구공학이란
- 고객이 시스템에 요구하는 서비스를 구축하는 과정과 시스템이 운영되고 개발되는 제약 조건
- 사용자와 개발자 간의 요구사항에 대한 오해로 인해 큰 장애가 발생할 수 있다.
- 구현의 정확성(correctness)의 판단 기준이다.
- 가능한 한 시스템이 어떻게 해야 하는지 보다는 무엇을 해야 하는지 설정해야 한다.
- 기능적 요구와 비기능적 요구가 있다.
    - 기능적 요구 : 시스템의 서비스나 기능을 설명한다.
    - 비기능적 요구 : 시스템이나 개발의 제한사항

## 요구사항의 종류
기능적 요구사항, 비기능적 요구사항(품질 요구사항, 제약사항)

### 기능적 요구사항
시스템에 주어지는 특정 입력에 대해서 시스템이 산출하는 출력에 의해서 정의된다.

### 비기능적 요구사항
시스템 속성 및 제약 조건을 정의한 것이다. 예를 들어, 신뢰성, 응답 시간 및 저장 요건. 제약 조건은 I/O 장치 성능, 시스템 표현 등이다.

기능적 요건보다 비기능적 요건이 더 중요할 수 있다.

이를 충족하지 못할 경우 시스템은 무용지물입니다. 기능적 요건과 비기능적 요건은 원칙적으로 요구사항 사양에서 구별되어야 합니다.

그러나 이는 요구사항이 개별 기능에 대한 제약이 아닌 전체 시스템 요구사항으로 표현될 수 있기 때문에 어렵다

요구사항이 기능적인지 아니면 비기능적인지를 결정하는 것은 때때로 어렵다

예를 들어, 안전을 위한 요구사항은 비기능적 특성과 관련이 있지만 시스템에 기능을 추가해야 할 수도 있다

#### 품질 요구사항
- 성능(performance): 시스템의 자원(CPU, 메모리 등)을 얼마나
효율적으로 사용하는가? 즉 사용자 입력에 대하여 얼마나 빠른
시간에 얼마나 적은 자원을 활용해서 결과를 출력할 수 있는가?

- 신뢰성(reliability): 시스템이 주어진 요구사항을 준수하여
동작하는 정도를 뜻한다. 일반적으로 장애 없이 동작하는 시간의
비율로서 정의된다. 

- 보안성(security): 허가되지 않은 사용자가 시스템에 접근하거나, 사용자가 접근 권한이 없는 시스템의 정보를 접근하거나 해서는
안된다. 보안성은 이러한 측면에 대한 요구사항을 뜻한다. 

- 안전성(safety): 시스템이 주변 환경, 인명, 재산에 피해를 주지
않아야 한다는 요구사항이다. 

-가용성(availability): 사용자가 원하는 순간에 시스템은 서비스를
제공해야 한다는 요구사항이다

#### 제약 사항
개발 방법론, 모델링 언어, 개발 언어, 운영체제, 미들웨어...

# 요구사항 명세서
- 고객 및 사용자 : 시스템의 개발 완료 승인에 대한 기준
- 개발자 : 분석 설계 구현에 대한 기준
- 테스터 : 테스트에 대한 기준
## 요구사항 명세서의 조건
명확성:  기술된 요구사항은 항상 동일한 의미로 해석되어야
한다. 즉 모호하지 않아야 한다. 

완전성 : 사용자가 기대하는 모든 기능/비기능적 요구사항이
기술되어야 한다. 즉 누락되어서는 안 된다. 

일관성:  서로 상충되는 요구사항이 있어서는 안 된다. 

검증가능성:  객관적으로 검증할 수 있도록 구체적이어야 한다. 

구현가능성:  가용한 기술과 한정된 일정/비용으로 구현이 가능해
야 한다

# 요구공학 과정

## 시작
그 문제에 대한 기본적인 이해, 이 문제의 해결을 원하는 사람들, 원하는 해결책의 성격, 고객과 개발자 간의 사전 커뮤니케이션 및 헙업의 효율성을 수립하기 위해 질문을 한다.

즉 이해관계자(이 일을 의뢰한 사람, 이 소프트웨어를 사용할 사람, 경제적 이점 등)를 파악하고, 여러 관점을 파악하며, 협업을 위한 노력을 한다.

## 도출
모든 이해관계자로부터 요구사항을 도출한다.

소프트웨어 엔지니어와 고객 모두 미팅을 진행하고 참석해야한다. 미팅에 관련된 규칙을 수립해야하고, 안건을 상정해야한다.

조력자(고객, 개발자, 외부인)가 회의를 진행하고 워크시트, 전자 게시판, 채팅 등을 사용한다.

요구사항을 도출하는 목적은 문제를 파악하고, 솔루션을 제안하고 다른 솔루션에 관해 협상하여 솔루션 요구사항의 예비 세트를 지정한다.

FAST( Facilitated Application Specification Techniques)

- 참가자는 전체 회의에 참석해야 합니다
- 모든 참가자는 평등하다
- 준비는 만남만큼이나 중요하다
- 회의 전 모든 문서는 "제안"된 것으로 간주됩니다
- 외부 미팅 장소가 선호
- 의제를 설정하고 유지한다
- 기술적인 세부 사항에 연연하지 마세요

## 정교화 또는 분석
데이터, 기능 및 행동 요구 사항을 식별하는 분석 모델 생성

### 분석 모델의 요소
- 시나리오 기반 요소
    - 기능 : 소프트웨어 기능에 대한 설명
    - 사용 사례 : 사람과 시스템간의 상호작용 설명
- 클래스 기반 요소
    - 시나리오에 의해 암시됨
- 행동 요소
    - state diagram
- 흐름 지향적 요소
    - Data Flow Diagram

### 분석 원칙
- 데이터 객체 정의
- 데이터 속성 설명
- 데이터간 관계 수립

- 데이터 객체를 변환하는 기능 식별
- 데이터가 어떻게 흘러가는지 표시
- 데이터의 생산자와 소비자 표시

- 시스템의 각기다른 state 표시
- state가 변화하는 이벤트 정의

- 추상화 수준이 낮도록 각 모델을 세분화
    - 데이터 객체 세분화
    - 함수 계층 구조 생성
    - 세부사항의 행동 표현

- 우선 실행 세부 사항은 고려하지 않고 문제의 본질에 초점을 맞추는 것으로 시작한다

- 필수 정보에서 구현 세부 정보로 이동

### 분석 모델링

statement of scope(Requirements statement): 구축할 시스템에 대한 비교적 간략한 설명. 입력 및 출력되는 데이터와 기본 기능, 조건부 처리(높은 수준), 특정 제약 조건 및 한계를 설명한다.

statement of scope는 FAST에서 나온 문서나 사용사례로부터 나온다.

statement of scope는 데이터, 기능, 행동 도메인 정보를 추출하기 위해 분석되어야 한다.

statement of scope의 모든 명사에 밑줄을 그어 객체를 정의한다. 객체에는 생산자/소비자 데이터, 데이터가 저장되는 곳, 복합 데이터 항목이 있다.

또한 모든 동사에 밑줄을 그어 operation을 정의한다. operation에는 응용 프로그램과 관련된 프로세스, 데이터 변환 등이 있다.
## 협상
개발자와 고객에게 현실적인 제공 가능한 시스템에 대한 동의

주요 이해관계자를 식별하고, 각 이해관계자의 목적을 파악, 서로 윈윈하는 요구사항을 도출한다.

## 명세서
문서, 모델, 사용 사례, 프로토타입

## 평가
내용 또는 해석에 오류가 있는지, 좀더 구체적으로 적혀야될 부분이 있는지, 누락된 정보, 불일치, 비현실적인 요구사항이 있는지 검사한다.

- 각 요구사항이 시스템/제품의 전반적인 목표와 일치합니까?
- 모든 요건이 적절한 수준의 추상화로 명시되었는가? 즉, 어떤 요건은 이 단계에서 부적절한 수준의 기술적 세부사항을 제공하는가?
- 요구사항이 정말로 필요한가요 아니면 시스템의 목표에 필수적이지 않을 수 있는 추가 기능을 나타내는가요?
- 각각의 요구사항은 경계가 있고 모호하지 않은가?
- 각 요건은 귀속성을 가지고 있는가? 즉, 각 요건에 대해 출처(일반적으로 특정 개인)를 주목하고 있는가?
- 다른 요구사항과 상충되는 요구사항이 있습니까?
- 시스템 또는 제품을 수용할 기술적 환경에서 각 요구사항을 달성할 수 있습니까?
- 각 요구사항이 구현되면 테스트 가능한가?
- 구축하고자 하는 시스템의 information, function 및 behavior을 적절히 반영하는 요구사항 모델을 실시하고 있는가.
- 요구사항 모델을 점진적으로 시스템에 대한 보다 상세한 정보를 노출하는 방식으로 "분할"하였는가
- 요구사항 모델을 단순화하기 위해 요구사항 패턴을 사용하였는가. 모든 패턴을 적절하게 검증하였는가? 모든 패턴이 고객의 요구사항과 일치하는가?
## 요구사항 관리

## 실현가능성 연구
가용 기술과 예산을 고려할 때 사용자의 요구사항이 만족되는지 확인한다

## 요구사항 분석
이해관계자가 요구하는 시스템 확인

## 요구사항 정의
고객이 이해할 수 있는 형태로 요구사항 정의

- 요구사항에 대한 고수준 추상적 설명
- 자연어로 된 진술
- 비전문가가 이해할 수 있게 작성
- 고객이 제공한 정보를 토대로 작성
- 고객이 시스템에 기대하는 모든 것의 목록
## 요구사항 문서화
요구사항을 세부적으로 정의한다.

시스템이 무엇을 해야하는지에 대한 상세한 설명
- 좀더 정확하고 형식적으로 적어야함
- 기술자들이 이해할 수 있게

객관적인 방법으로 외부 시스템 동작 지정

- 제약 조건 지정
- 유지보수를 위한 참고문서
- 시스템의 라이프 사이클에 대한 생각
- 예기치 않은 이벤트에 대한 응답 생성
- 추적 가능하게 함
- 정확하고, 일관되고, 완전하고, 모호하지 않고, 이해할 수 있게
- 요구사항은 사용자 요구사항을 더 잘 이해하고 조직의 목표가 변화함에 따라 항상 바뀐다.
- 시스템이 개발되고 사용됨에 따라 요구사항의 변경을 계획하는 것이 필수적이다

### 요구사항 문서 구조1
- 서론 : 시스템의 필요성 및 목표와 어떻게 부합하는지 설명
- 용어집 : 사용되는 기술 용어 정의
- 시스템 모델 : 시스템 구성 요소와 관계를 보여주는 모델 정의
- 기능 요구사항 정의 : 제공할 서비스 설명
- 비기능 요구사항 정의 : 시스템 및 개발 프로세스의 제약 조건 정의
- 시스템 변화 : 시스템의 예상되는 변화 정의
- 요구사항 : 기능 요구사항 상세 명세
- 부록 : 하드웨어 플랫폼 설명, 데이터베이스 요구사항(ER 모델)
- 색인

### 요구사항 문서 구조2
- 제품 특징
- 개발, 운영 및 유지보수 환경
- 외부 인터페이스 및 데이터 흐름
- 기능 요구사항
- 성능 요구사항
- 예외 처리
- 구현 우선순위
- 예측 가능한 수정 및 향상
- 합격기준
- 설계 힌트 및 지침
- 교차 참조 색인
- 용어집


## 소프트웨어 문서화
소프트웨어 설계에 대한 추상적 설명. 설계 및 구현의 기초이며, 소프트웨어 설계자가 읽는다. 요구사항 문서로 대체될 수 있다.

# 요구 공학이 어려운 이유
사용자마다 요구사항과 우선순위가 다르다. 요구사항은 끊임없이 변화한다.
문제가 너무 복잡해서 이해가 안되거나 정확하게 표현이 어려울 수 있다. 애플리케이션 도메인을 이해해야한다. 커뮤니케이션의 부재는 요구사항의 불완전과 비일관성을 초래한다.
