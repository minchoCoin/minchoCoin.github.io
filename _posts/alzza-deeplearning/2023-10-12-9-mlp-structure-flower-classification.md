---
title: "다층 퍼셉트론 모델 구조 - 꽃 이미지 분류 신경망 이론 및 실행"
last_modified_at: 2023-10-12T22:12:12+09:00
categories:
    - alzza-deeplearning
tags:
    - A.I

toc: true
toc_label: "My Table of Contents"
author_profile: true

---
# 목적
꽃 이미지 분류 신경망을 여러 은닉 계층 수를 적용해 실행해보고, 결과를 분석한다.

# 실험 원리
## 데이터 분할: 학습, 검증, 평가
일반적으로 데이터는 학습 재료가 되는 훈련 데이터(학습 데이터), 학습 과정에서 중간 점검에 이용되는 검증 데이터, 학습 후에 학습 품질을 확인하는데 이용되는 평가 데이터로 나뉜다. 

딥러닝에서, 학습 데이터의 정확도도 중요하지만, 학습 데이터에 사용되지 않은 데이터를 이용해 평가하는 것이 중요하다. 학습 데이터를 이용해 학습을 하면 검증 데이터를 이용해 중간 점검을 하여 예측 성능을 평가하고 마지막으로 테스트 데이터를 이용해 최종적으로 나온 모델의 성능을 측정한다. 

특히 검증 데이터는 학습의 강도를 결정하는데 사용할 수 있다. 일반적으로, 너무 많이 학습시키면 훈련 데이터에만 적합한 모델이 나오는데(과대적합(Overfitting)이라고 한다) 이를 방지하기 위해 훈련 데이터에 사용하지 않은 검증 데이터를 이용해 정확도를 측정하여, 모델이 모든 데이터에 대해 적합하게 학습하고 있는지 검증하는데 사용할 수 있다.

또한 평가 데이터를 이용해 최종적으로 나온 모델의 성능을 확인할 수 있다. 구체적으로, 학습 데이터에 대한 정확도보다 평가 데이터에 대한 정확도가 매우 낮다면, 이는 과대적합으로 볼 수 있다. 반대로 평가 데이터에 대한 정확도가 더 높거나 또는 두 데이터에 대한 정확도가 모두 매우 낮다면, 이는 과소적합(underfitting, 모델이 학습 데이터로부터 특징을 산출하지 못해 학습을 거의 하지 못한 것)으로 볼 수 있다.

마지막으로 학습 데이터, 검증 데이터, 평가 데이터를 완전히 분리하는 것이 중요하다. 즉 중복 데이터가 들어가면 안 된다는 것이다. 특히 학습 데이터와 평가 데이터가 중복되면 모델의 성능을 제대로 평가할 수 없는데, 만약 중복되었다면, 평가 데이터가 모델 파라미터에 영향을 주어 평가 데이터에 맞는 모델이 나왔을 것이므로, 이 모델이 어떤 데이터가 들어와도 예측을 잘 할 수 있는지 평가할 수 없기 때문이다. 

## 신경망의 파라미터 개수가 많아졌을 때 문제점
예를 들어 꽃 사진 분류에서 사진의 크기를 160x120, 그리고 컬러 이미지일 때 입력 벡터의 크기는 160x120x3 = 57600이며 은닉 계층을 3개, 폭을 각각 5000,500,50으로 설정하면 약 3억개의 파라미터 수가 필요하게 된다.

이렇게 파라미터 수가 많아지게 되면 지나치게 많은 데이터가 필요하다는 단점이 있다. 파라미터를 학습시킬 데이터가 부족하면, 제대로 학습이 되지 않을 수 있는데 이는 손실함수의 평균 $L = \frac{\sum 손실함수(y,output)}{AB}$ (여기서 A는 이전 계층에서 출력한 출력 벡터의 개수, B는 현재 계층에서의 출력 벡터의 개수이다)에서 A와 B가 지나치게 커지면, 역전파 처리 과정을 처리할 때, $\frac{\partial L}{\partial output_{ij}} = \frac{\partial L}{\partial 손실함수_{ij}}\frac{\partial 손실함수_{ij}}{\partial output_{ij}}$ 에서 $frac{\partial L}{\partial 손실함수_{ij}} = \frac{1}{AB}$ 이므로, 한 데이터를 넣었을 때 파라미터 값이 조정되는 양이 매우 작아지기 때문이다.
일반적으로 모델의 파라미터 개수의 10배 되는 데이터가 있어야 정확도(f-score)가 0.85이상 나온다고 알려져 있다([https://malay-haldar.medium.com/how-much-training-data-do-you-need-da8ec091e956](https://malay-haldar.medium.com/how-much-training-data-do-you-need-da8ec091e956)).

## 다층 퍼셉트론이 이미지 처리에 부적합한 이유
앞서 꽃 사진 분류에서 약 3억개의 파라미터 수가 필요하게 된다고 하였다. 이는 다층 퍼셉트론 신경망의 은닉 계층은 인접한 양쪽 계층의 모든 퍼셉트론이 서로 연결되기 때문이다. 이를 완전 연결 계층(fully connected layer)라고 한다. 즉 이전 계층의 모든 출력을 이용하여 계층의 한 퍼셉트론의 가중치를 계산하게 된다. ‘2. 실험원리 – (2) 신경망의 파라미터 개수가 많아졌을 때 문제점’에서 살펴보았듯이, 파라미터 수가 많으면 지나치게 많은 양의 데이터가 필요하게 된다.

# 실험 방법
데이터 셋은 다음 데이터 셋을 사용하였다.

https://www.kaggle.com/datasets/alxmamaev/flowers-recognition/

코드는 다음 코드를 사용하였다.

https://github.com/KONANtechnology/Academy.ALZZA/tree/master/codes/chap05

사진의 크기가 다 다르므로, [100,100]으로 resolution하여 학습에 사용한다.
1. 은닉 계층을 1개, 은닉 계층의 폭을 10으로 두어 학습하고, 결과를 확인한다.
2. 은닉 계층을 2개, 은닉 계층의 폭을 각각 30,10으로 두어 학습하고, 결과를 확인한다.
3. 은닉 계층을 3개, 은닉 계층의 폭을 각각 90,30,10으로 두어 학습하고, 결과를 확인한다.
4. 은닉 계층을 4개, 은닉 계층의 폭을 각각 270,90,30,10으로 두어 학습하고, 결과를 확인한다.
5. 위의 (1)~(4) 과정을 은닉 계층의 폭을 각각 2배 늘려 학습하고 결과를 확인한다.

| 입력 계층 | 은닉계층1 | 은닉계층2 | 은닉계층3 | 은닉계층4 | 출력계층 | 총 파라미터 개수  |
|-------|-------|-------|-------|-------|------|------------|
| 30000 | 10    |       |       |       | 5    | 300065     |
| 30000 | 20    |       |       |       | 5    | 600125     |
| 30000 | 30    | 10    |       |       | 5    | 900395     |
| 30000 | 60    | 20    |       |       | 5    | 1801385    |
| 30000 | 90    | 30    | 10    |       | 5    | 2703185    |
| 30000 | 180   | 60    | 20    |       | 5    | 5412365    |
| 30000 | 270   | 90    | 30    | 10    | 5    | 8127755    |
| 30000 | 540   | 180   | 60    | 20    | 5    | 16310105   |

(표1 실험에 사용하는 은닉계층의 수와 폭에 따른 총 파라미터 개수)

# 실험 결과
추정 확률 분포는 [daisy, dandelion, rose, sunflower, tulip] 순이다.

은닉 계층이 [x,y,z]라는 뜻은 은닉 계층의 수가 3개이며, 각각 x,y,z의 폭을 가진 은닉 계층을 말한다.

![데이터셋이 들어있는 폴더의 폴더 구조](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/bf1a76f9-1805-4331-b11c-fc68ee948834)

(그림 1 데이터셋이 들어있는 폴더의 폴더 구조)

![꽃 데이터를 분류하는 타깃을 설정하는 과정. 그림1에 있는 폴더 순서로 타깃을 가져온다.](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/714c3e2b-0058-4ea1-b8d2-d06f9e958cbb)

(그림 2 꽃 데이터를 분류하는 타깃을 설정하는 과정. 그림1에 있는 폴더 순서로 타깃을 가져온다.)

이는 flower_init()함수가 target_name을 flowers 폴더 안에 있는 daisy, dandelion, rose, sunflower, tulip 폴더의 이름과 순서대로 하기 때문이다.

## 은닉 계층 1개
### 은닉 계층의 폭 (10)
![은닉 계층이 (10)일 때 학습 결과](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/c299b55e-2bc4-4bf2-aec3-72c0e89d32f9)

(그림 3 은닉 계층이 (10)일 때 학습 결과)

검증 데이터로 측정한 정확도는 0.247, 테스트 데이터로 측정한 정확도는 0.232가 나왔으며, 정답이 daisy, daisy, rose인 데이터에 대해 각각 dandelion, dandelion, dandelion으로 예측하였다.

### 은닉 계층의 폭 (20)
![은닉 계층이 (20)일 때 학습 결과](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/4df392be-e381-4638-a369-08ee33bdd7e1)

(그림 4 은닉 계층이 (20)일 때 학습 결과)

검증 데이터로 측정한 정확도는 0.246, 테스트 데이터로 측정한 정확도는 0.238이 나왔으며, 정답이 daisy, daisy, rose인 데이터에 대해 각각 dandelion, dandelion, dandelion으로 예측하였다.

## 은닉 계층 2개
### 은닉 계층의 폭 (30,10)
![은닉 계층이 (30,10)일 때 학습 결과](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/eaeb23c4-3d99-4fb7-8393-89c765c9ae07)

(그림 5 은닉 계층이 (30,10)일 때 학습 결과)

검증 데이터로 측정한 정확도는 0.483, 테스트 데이터로 측정한 정확도가 0.394가 나왔으며 정답이 daisy, dandelion, dandelion인 데이터에 대해 각각daisy, tulip, dandelion이라고 예측하였다.

### 은닉 계층의 폭 (60,20)
![은닉 계층이 (60,20)일 때 학습 결과](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/d1c7bd5d-00f6-4899-b8d3-c14564509169)

(그림6 은닉 계층이 (60,20)일 때 학습 결과)

검증 데이터로 측정한 정확도는 0.464, 테스트 데이터로 측정한 정확도는 0.399가 나왔으며, 정답이 rose, tulip, rose인 데이터에 대해 각각 dandelion, tulip, tulip이라고 예측하였다.

## 은닉 계층 3개
### 은닉 계층의 폭 (90,30,10)
![은닉 계층이 (90,30,10)일 때 학습 결과](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/845d9cec-cbf2-4acf-ac46-5fb5debb10b1)

(그림7 은닉 계층이 (90,30,10)일 때 학습 결과)

검증 데이터로 측정한 정확도는 0.560, 테스트 데이터로 측정한 정확도가 0.462가 나왔으며 정답이 daisy, tulip, sunflower인 데이터에 대해 daisy, tulip, sunflower로 예측하였다.

### 은닉 계층의 폭 (180,60,20)
![은닉 계층의 폭 (180,60,20)일 때 학습 결과](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/6b0f3866-7496-483b-bb18-e399ae3c96fa)

(그림8 은닉 계층이 (180,60,20)일 때 학습 결과)

검증 데이터로 측정한 정확도는 0.582, 테스트 데이터로 측정한 정확도는 0.449가 나왔으며, 정답이 tulip, tulip, rose인 데이터에 대해 각각 tulip, tulip, rose으로 예측하였다.

## 은닉 계층 4개
### 은닉 계층의 폭 (270,90,30,10)
![은닉 계층의 폭 (270,90,30,10)일 때 학습 결과](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/bef0fb7c-649a-469d-bd84-c3c38c8bd579)

(그림9 은닉 계층이 (270,90,30,10)일 때 학습 결과)

검증 데이터로 측정한 정확도는 0.530, 테스트 데이터로 측정한 정확도가 0.468가 나왔으며 정답이 daisy, rose, tulip인 데이터에 대해 sunflower, tulip, rose으로 예측하였다.

### 은닉 계층의 폭 (540,180,60,20)
![은닉 계층이(540,180,60,20)일 때 학습 결과](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/a8724c97-45b6-47bf-94b1-f81a81a16336)

(그림 10 은닉 계층이 (540,180,60,20) 일 때 학습 결과)

# 결과 분석

| 은닉 계층 폭         | 총 파라미터 개수 | 검증 데이터 정확도 | 테스트 데이터 정확도  |
|-----------------|-----------|------------|--------------|
| [10]            | 300065    | 0.247      | 0.232        |
| [20]            | 600125    | 0.246      | 0.238        |
| [30,10]         | 900395    | 0.483      | 0.394        |
| [60,20]         | 1801385   | 0.464      | 0.399        |
| [90,30,10]      | 2703185   | 0.560      | 0.462        |
| [180,60,20]     | 5412365   | 0.582      | 0.449        |
| [270,90,30,10]  | 8127755   | 0.530      | 0.468        |
| [540,180,60,20] | 16310105  | 0.677      | 0.468        |

(표2 은닉 계층 수와 폭에 따른 파라미터 개수와 검증 데이터 정확도 및 테스트 데이터 정확도)

그림 3과 그림 4에서 알 수 있듯 은닉 계층이 1개일 때 테스트 정확도는 각각 0.232(은닉 계층 폭 10), 0.238(은닉 계층 폭 20)이다. 그러나 답을 언제나 dandelion으로 예측하며, 확률 분포도 다른 데이터에 대해 똑같이 예측하고 있다. 이는 학습 데이터 중 dandelion이 제일 많이 때문인 것으로 보인다. 즉 모델이 단순하여 학습 데이터로부터 특징을 뽑아내지 못하고, 학습 데이터에서 각 타깃의 양으로 추정하고 있다. 즉 과소적합이 일어나 검증 데이터의 정확도와 테스트 데이터의 정확도 모두 낮다.

은닉 계층의 수와 폭을 늘렸을 때, 오히려 감소하기도 하나 대체적으로 증가하는 모습을 보이며, 특히 은닉 계층의 수 4개와 폭 [540,180,60,20]에서 검증 데이터의 정확도가 0.677이 나왔다. 그러나 테스트 데이터의 정확도는 0.468로 은닉 계층 3개일 때와 비교하여 크게 좋아지지 않았다. 약간의 과대적합이 일어난 것으로 보인다. 따라서 다층 퍼셉트론은 은닉 계층의 수가 증가할 때 파라미터의 수가 많이 증가하고, 증가한 파라미터 수에 비해 정확도가 높지 않고(앞서 살펴보았듯이, 정확도를 증가시키기 위해서는 증가한 파라미터 수 때문에, 더 많은 데이터셋이 필요해 보인다), 이미지 분류 신경망에 다층 퍼셉트론을 적용하는 것은 부적합한 것을 알 수 있다.

# 결론
꽃 분류 신경망을 은닉 계층 [10], [20], [30,10], [60,20], [90,30,10], [180,60,20], [270,90,30,10], [540,180,60,20]에 적용시켜보고, 결과를 확인하였다. 은닉 계층의 수가 1개일 때는 항상 dandelion으로 추정하고, 검증 데이터와 테스트 데이터의 정확도가 모두 낮게 나와 과소적합이 일어났으며, 은닉 계층이 [540,180,60,20] 일 때는 검증 데이터의 정확도가 0.677로 제일 좋게 나왔으나 테스트 데이터의 정확도는 그만큼 증가하지 않았다. 증가한 파라미터 수에 비해 데이터의 수가 적어서 그런 것으로 보인다. 그러나 증가한 파라미터 수에 맞는 데이터셋의 양을 충족하기는 현실적으로 어려우므로, 다층 퍼셉트론은 이미지 분류 신경망에 부적합한 것을 알 수 있다.

# 참고 문헌
-	윤덕호 저, [파이썬 날코딩으로 알고 짜는 딥러닝], 한빛미디어, 2019
-	박해선 저, [혼자 공부하는 머신러닝+딥러닝], 한빛미디어, 2020
-	https://tbr74.tistory.com/entry/%EB%8D%B0%EC%9D%B4%ED%84%B0%EC%85%8B%EC%9D%98-%EB%B6%84%EB%A6%AC-Validation-%EA%B2%80%EC%A6%9D-%EB%8D%B0%EC%9D%B4%ED%84%B0%EC%85%8B%EC%9D%80-%EB%AC%B4%EC%97%87%EC%9D%B8%EA%B0%80

- https://malay-haldar.medium.com/how-much-training-data-do-you-need-da8ec091e956