---
title: "복합 출력의 처리 방법 - 오피스31 다차원 분류 신경망"
last_modified_at: 2023-10-30T22:18:12+09:00
categories:
    - alzza-deeplearning
tags:
    - A.I

toc: true
toc_label: "My Table of Contents"
author_profile: true

---
# 목적
오피스31 다차원 분류 신경망을 여러 은닉 계층 수와 아담 알고리즘을 적용하여 실행해보고 은닉 계층 수와 아담알고리즘 적용여부에 따른 정확도 변화를 확인한다.

# 실험 원리

## 전이 학습
전이학습이란 한 도메인에서 학습시킨 결과를 다른 도메인에 활용하여 학습 효과를 높이는 기법이다. 예를 들어 아마존 도메인으로 학습한 모델이 있으면, 이 모델을 이용하여 dSLR 도메인에서 적은 데이터만 이용하여 dSLR 도메인을 학습시킬 수 있다.

전이학습을 사용하면 학습이 빠르게 수행될 수 있다. 예를 들어 아마존 도메인으로 학습한 모델을 이용하여 dSLR 데이터로부터 특징(어느 오피스 용품인지)을 빠르게 추출하기 때문이다. 또한 오버피팅을 예방할 수 있는데, 전이 학습을 이용하여 마지막 레이어만 학습한다면, 학습할 가중치 수가 줄어들기 때문이다.

## 딥러닝에서 복합 출력의 학습법
신경망 회로에서 알맞은 크기의 출력 벡터를 만들어내기만 하면 된다. 예를 들어 오피스 31의 경우 출력 벡터의 크기는 도메인 판별 3 + 품복 판별 31을 합쳐 34가지 성분을 가지면 된다. 그리고 레이블 정보로 준비된 정답 y1, y2와 복합 출력의 두 성분 output1, output2를 비교해 손실값 L1, L2가 얻어질 때, 전체적인 손실함수값은 L=L1+L2로 정의한다. 편미분의 특성때문에, L을 이용해 손실기울기를 구할 때 output1의 손실기울기는 L1과 y1, output2의 손실기울기는 L2과 y2를 이용하여 구하면 된다. 손실 기울기는 두 부분으로 갈라진 출력 계층에 따로 반영된다. 그러나 입력으로 들어갈 때는 뒤섞여 반영된다.

## 아담 알고리즘
아담 알고리즘은 파라미터에 적용되는 실질적인 학습률을 개별 파라미터 별로 동적으로 조절하는 방법이다.

경사하강법은 파라미터의 손실 기울기와 미리 정해진 학습률을 곱한 값을 각 파라미터에서 빼주는 방식으로 학습을 수행한다.

아담 알고리즘은 파라미터의 1차 모멘텀 정보(손실기울기)와 2차 모멘텀 정보(손실기울기를 한 번 더 미분한 값)(모멘텀은 파라미터값의 변화 추세를 나타낸다)을 이용하여 학습률을 조절하는 방식으로 학습을 수행한다.

구체적으로 다음과 같이 계산한다.

$$W=W-\frac{\alpha}{\sqrt{S_{dw(t)+\epsilon}}}V_{dw(t)} $$

$$V_{dw(0)} = \overrightarrow{0}, V_{dw(t)}=\beta _ {1} V_{dw(t-1)} + (1-\beta _ {1})dW$$

$$S_{dw(0)} = \overrightarrow{0}, S_{dw(t)}=\beta _ {2} S_{dw(t-1)} + (1-\beta _ {2})dW^2$$

이때 ε 은 0으로 나누는 것을 방지하기 위해 사용하는 값이며, 보통 $10^{-8}$ 을 사용한다. 그리고 $\beta _1,\beta _2$ 는 일반적으로 각각 0.9와 0.99를 쓴다.

추가적으로 Bias Correction을 할 수 있는데, 이는 지수적인 이동 평균에서 β가 커질 수록 처음에 원래의 데이터에 비해 낮게 나오므로 다음과 같이 값을 조정할 수 있다.

$$V_{dw(t)} = \frac{V_{dw(t)}}{1-\beta _ 1 ^ t} $$

$$S_{dw(t)} = \frac{S_{dw(t)}}{1-\beta _ 2 ^ t} $$

이와 같이 아담 알고리즘은 각 파라미터의 변화 추세를 이용하여 파라미터별로 학습률을 조정한다. 따라서 아담 알고리즘은 실질적인 학습률을 보정해 적용함으로서 일괄적인 학습률 적용에서 오는 품질 저하를 막아준다. 구체적으로, 2차 모멘텀이 크다면, 즉 기울기가 빠르게 가파르게 된다면 학습률이 떨어지고, 기울기가 천천히 가파르게 된다면 학습률이 올라간다.

# 실험에 사용한 코드 및 데이터
## 코드 출처

[https://github.com/KONANtechnology/Academy.ALZZA/tree/master/codes/chap06](https://github.com/KONANtechnology/Academy.ALZZA/tree/master/codes/chap06)

## 데이터 출처

[https://drive.google.com/file/d/0B4IapRTv9pJ1WGZVd1VDMmhwdlE/view](https://drive.google.com/file/d/0B4IapRTv9pJ1WGZVd1VDMmhwdlE/view)

# 실험 과정
1. Epoch는 50, learning rate는 0.0001로 한다.
2. 은닉 계층 (10), 아담 알고리즘을 미적용하고 학습하여 결과를 확인한다.
3. 은닉 계층 (10), 아담 알고리즘을 적용하고 학습하여 결과를 확인한다.
4. 은닉 계층 (64,32,10), 아담 알고리즘을 미적용하고 학습하여 결과를 확인한다.
5. 은닉 계층 (64,32,10), 아담 알고리즘을 적용하고 학습하여 결과를 확인한다.
6. 은닉 계층 (384,192,64,32,10), 아담 알고리즘을 미적용하고 학습하여 결과를 확인한다.
7. 은닉 계층 (384,192,64,32,10), 아담 알고리즘을 적용하고 학습하여 결과를 확인한다.

# 실험 결과

## 은닉 계층 (10)
### 아담 알고리즘 미적용
![은닉 계층 (10), 아담 알고리즘 미적용 시 학습 결과](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/e2e54f60-8d05-425c-b3b8-f62721aa023b)

(그림 1 은닉 계층 (10), 아담 알고리즘 미적용 시 학습 결과)

도메인 정확도는 0.661, 품목 선택 정확도는 0.032이다. 정답이 (amazon, amazon, amazon)인 데이터에 대해 (amazon, amazon, amazon)으로 예측하였으며, 정답이(stapler, calculator, calculator)인 데이터에 대해 (laptop_computer, laptop_computer, laptop_computer)로 예측하였다. 학습 시간은 93초 소요되었다.

### 아담 알고리즘 적용
![은닉 계층 (10), 아담 알고리즘 적용시 학습 결과](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/807428ed-fbaf-40d9-a7ac-501e1cbd2bc0)

(그림 2 은닉 계층 (10), 아담 알고리즘 적용시 학습 결과)

도메인 정확도는 0.661, 품목 선택 정확도는 0.032이다. 정답이 (amazon, amazon, amazon)인 데이터에 대해 (amazon, amazon, amazon)으로 예측하였으며, 정답이(bike_helmet, pen, keyboard)인 데이터에 대해 (laptop_computer, laptop_computer, laptop_computer)로 예측하였다. 학습 시간은 396초 소요되었다.

## 은닉 계층 (64,32,10)

### 아담 알고리즘 미적용
![은닉 계층 (64,32,10), 아담 알고리즘 미적용 시 학습 결과](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/ed0c1fb5-6f2c-4a5e-83d3-1ca29c1f40d1)

(그림 3 은닉 계층 (64,32,10), 아담 알고리즘 미적용 시 학습 결과)

도메인 정확도는 0.872, 품목 선택 정확도는 0.195이다. 정답이 (amazon, amazon, amazon)인 데이터에 대해 (amazon, amazon, amazon)으로 예측하였으며, 정답이(stapler, mug, speaker)인 데이터에 대해 (punchers, scissors, monitor)로 예측하였다. 학습 시간은 257초 소요되었다.

### 아담 알고리즘 적용
![은닉 계층 (64,32,10), 아담 알고리즘 적용 시 학습 결과](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/e68fabdc-5815-44d4-ae22-a5134ab3cea0)

(그림 4 은닉 계층 (64,32,10), 아담 알고리즘 적용 시 학습 결과)

도메인 정확도는 0.858, 품목 선택 정확도는 0.269이다. 정답이 (amazon, dslr, amazon)인 데이터에 대해 (amazon, webcam, amazon)으로 예측하였으며, 정답이(speaker, printer, keyboard)인 데이터에 대해 (speaker, speaker, keyboard)로 예측하였다. 학습 시간은 2260초 소요되었다.

## 은닉 계층 (384,192,64,32,10)

### 아담 알고리즘 미적용
![은닉 계층 (384,192,64,32,10), 아담 알고리즘 미적용 시 학습 결과](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/36be63cb-5efb-438e-8414-60f7493d3cc4)

(그림 5 은닉 계층 (384,192,64,32,10), 아담 알고리즘 미적용 시 학습 결과)

### 아담 알고리즘 적용
![은닉 계층 (384,192,64,32,10), 아담 알고리즘 적용 시 학습 결과](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/fa6debea-1067-4a13-985d-84061c5de7eb)

(그림 6 은닉 계층 (384,192,64,32,10), 아담 알고리즘 적용 시 학습 결과)

도메인 정확도는 0.890, 품목 선택 정확도는 0.283이다. 정답이 (amazon, amazon, dslr)인 데이터에 대해 (amazon, amazon, webcam)으로 예측하였으며, 정답이(projector, ring_binder, mouse)인 데이터에 대해 (bike_helmet, projector, calculator)로 예측하였다. 학습 시간은 11752초 소요되었다.

# 결과 분석
|     은닉 계층             |     아담 알고리즘     |     도메인 정확도    |     품목 정확도    |     소요시간(초)    |
|---------------------------|-----------------------|----------------------|--------------------|---------------------|
|     (10)                  |     미적용            |             0.661    |           0.032    |               93    |
|     (10)                  |     적용              |             0.661    |           0.032    |              396    |
|     (64,32,10)            |     미적용            |             0.872    |           0.195    |              257    |
|     (64.32,10)            |     적용              |             0.858    |           0.269    |             2260    |
|     (384,192,64,32,10)    |     미적용            |             0.837    |           0.214    |             2376    |
|     (384,192,64,32,10)    |     적용              |             0.890    |           0.283    |            11752    |

(표 1 은닉 계층과 아담 알고리즘 적용 여부에 따른 도메인 정확도, 품목 정확도와 소요시간(초))

그림 1, 그림 2와 표1에서, 은닉 계층 (10)을 적용하였을 때는 아담 알고리즘 적용여부와 상관없이 도메인 추정 결과와 상품 추정 결과 모두 데이터에 관계없이 동일환 확률 분포로 추정하고, 같은 답을 선택하였다. 은닉 계층의 수가 부족하여 데이터 중 가장 많은 데이터인 것으로 추정하고 있다.

그림3, 표1에서, 은닉 계층의 수를 3개로 늘렸을 때 도메인 정확도 0.872, 품목 정확도 0.195로 향상되었다. 이제 데이터에 따라 확률 분포가 다르게 구해지고, 이에 따라 데이터에 따라 다른 답을 선택하고 있다. 그림4와 표 1에서, 은닉 계층의 수 3개와 아담알고리즘을 적용하였을 때는 아담 알고리즘을 적용하지 않았을 때 보다 정확도는 감소하였지만 품목 정확도는 7.4%p 상승하였다.

그림5, 표1에서, 은닉 계층의 수를 5개로 늘렸을 때 은닉 계층 3개일 때 보다 도메인 정확도는 소폭 감소하였지만 품목 정확도는 소폭 증가하였다. 그림6과 표1에서, 은닉 계층의 수를 5개로 늘리고 아담 알고리즘까지 적용하였을 때, 아담 알고리즘을 적용하지 않았을 때보다 도메인 정확도와 품목 정확도 모두 약 5~6%p 정도 상승하였다.

학습 소요시간은 은닉 계층의 수와 폭을 늘릴 수록, 그리고 아담 알고리즘을 적용했을 때 더 많은 시간이 소요되었다. 은닉 계층의 수를 1개, 3개, 5개로 늘릴 때마다 약 3배~9배 정도 더많은 시간이 소요되었으며, 아담 알고리즘을 적용하면 적용하지 않았을 때보다 약 4배~9배 정도 시간이 더 소요되었다. 그러나 학습을 노트북으로 진행하여 배터리와 CPU 온도에 따라 CPU 성능이 변하여 정확한 비교는 아닐 수 있다.

따라서 아담 알고리즘을 적용했을 때 소요시간이 더 많이 걸리고, 정확도도 상승하는 것을 알 수 있었다. 

# 결론
오피스31 다차원 분류 신경망을 은닉 계층 (10), (64,32,10), (384,192,64,32,10) 그리고 각 은닉 계층에 아담 알고리즘을 적용/미적용하여 학습한 후 정확도와 소요시간을 비교하였다. 은닉 계층이 늘어날 수록, 그리고 아담 알고리즘을 적용했을 때 정확도가 대체적으로 상승하였다. 그리고 소요시간도 약 3배~9배 정도 증가하였다. 따라서 아담 알고리즘을 적용하였을 때 적용하지 않았을 때 보다 정확도가 상승하는 것을 알 수 있었다.

# 참고 문헌
- 윤덕호 저, [파이썬 날코딩으로 알고 짜는 딥러닝], 한빛미디어, 2019
- https://velog.io/@tjdtnsu/%EB%94%A5%EB%9F%AC%EB%8B%9D-%EA%B8%B0%EC%B4%88-%EC%A0%84%EC%9D%B4-%ED%95%99%EC%8A%B5-%EC%9D%B4%ED%95%B4%ED%95%98%EA%B8%B0
- https://angeloyeo.github.io/2020/09/26/gradient_descent_with_momentum.html
- Diederik P. Kingma, Jimmy Lei Ba, “ADAM: A METHOD FOR STOCHASTIC OPTIMIZATION”, arXiv:1412.6980v9 [cs.LG] 30 Jan 2017, 
https://arxiv.org/pdf/1412.6980.pdf
- https://amber-chaeeunk.tistory.com/23
- https://mole-starseeker.tistory.com/48

