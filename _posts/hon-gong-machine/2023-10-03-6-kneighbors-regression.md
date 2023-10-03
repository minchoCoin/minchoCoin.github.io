---
title: "3-1. 회귀 알고리즘와 모델 규제 - K-최근접 이웃 회귀"
last_modified_at: 2023-10-03T10:40:12+09:00
categories:
    - hon-gong-machine
tags:
    - A.I

author_profile: true

toc: true
toc_label: "My Table of Contents"

---
# K-최근접 이웃 회귀

지도 학습 알고리즘은 크게 분류와 회귀로 나뉜다. 분류는 몇 개의 클래스 중 하나로 분류하는 문제이고, 회귀는 임의의 어떤 숫자를 예측하는 문제이다.

K-최근접 이웃 회귀는 이웃 샘플의 평균을 새로운 샘플 X의 예측값으로 예측하는 방법이다. 예를 들어 새로운 샘플 X 근처에 100,80,60의 값을 가진 샘플이 있다면 X의 예측 값은 (100+80+60)/3 = 80의 값을 가진다.

# 데이터 준비

먼저 넘파이 배열로 아래와 같이 농어의 길이와 농어의 무게 데이터를 준비한다.
```py
perch_length = np.array(
    [8.4, 13.7, 15.0, 16.2, 17.4, 18.0, 18.7, 19.0, 19.6, 20.0,
     21.0, 21.0, 21.0, 21.3, 22.0, 22.0, 22.0, 22.0, 22.0, 22.5,
     22.5, 22.7, 23.0, 23.5, 24.0, 24.0, 24.6, 25.0, 25.6, 26.5,
     27.3, 27.5, 27.5, 27.5, 28.0, 28.7, 30.0, 32.8, 34.5, 35.0,
     36.5, 36.0, 37.0, 37.0, 39.0, 39.0, 39.0, 40.0, 40.0, 40.0,
     40.0, 42.0, 43.0, 43.0, 43.5, 44.0]
     )
perch_weight = np.array(
    [5.9, 32.0, 40.0, 51.5, 70.0, 100.0, 78.0, 80.0, 85.0, 85.0,
     110.0, 115.0, 125.0, 130.0, 120.0, 120.0, 130.0, 135.0, 110.0,
     130.0, 150.0, 145.0, 150.0, 170.0, 225.0, 145.0, 188.0, 180.0,
     197.0, 218.0, 300.0, 260.0, 265.0, 250.0, 250.0, 300.0, 320.0,
     514.0, 556.0, 840.0, 685.0, 700.0, 700.0, 690.0, 900.0, 650.0,
     820.0, 850.0, 900.0, 1015.0, 820.0, 1100.0, 1000.0, 1100.0,
     1000.0, 1000.0]
     )
```
그리고 아래와 같이 훈련 세트와 테스트 세트로 나눈다.
```py
from sklearn.model_selection import train_test_split

train_input, test_input, train_target, test_target = train_test_split(
    perch_length, perch_weight, random_state=42)
```

그리고 사용할 훈련 세트는 2차원 배열이여야하므로, reshape()를 이용하여 배열의 크기를 바꾸자. 예를 들어
```py
np.array([1,2,3,4]).reshape(2,2)
```
는 아래와 같이 (2,2) 크기의 배열로 바뀐다.

```py
array([[1, 2],
       [3, 4]])
```

그리고 reshape(-1,1)로 하면 첫 번째 크기를 나머지 원소 개수로 채우고, 두번째 크기를 1로 하는 것으로,
```py
np.array([1,2,3]).reshape(-1,1)
```
은 
```py
[
    [1],
    [2],
    [3]
]
```
의 넘파이 배열을 만든다.

따라서 아래와 같이 train_input과 test_input의 배열 크기를 바꿀 수 있다.

```py
train_input = train_input.reshape(-1, 1)
test_input = test_input.reshape(-1, 1)
```

# 결정 계수
```py
from sklearn.neighbors import KNeighborsRegressor
knr = KNeighborsRegressor()
# k-최근접 이웃 회귀 모델을 훈련합니다
knr.fit(train_input, train_target)
knr.score(test_input, test_target)
```
위 코드를 실행하며 K-최근접 이웃 회귀 모델을 훈련하고 score를 확인하면 약 0.9928이 나온다. 이 값은 결정 계수라고 부르는 것으로 $R^2$라고도 부른다. 아래와 같이 계산한다.

$$ R^2 = 1- \frac{(타깃 - 예측)^2}{(타깃-평균)^2}$$

위 식에서 알 수 있듯, 타깃의 평균 정도를 예측하는 수준이라면 0에 가까워지고, 타깃이 예측에 아주 가까워지면 1에 가까운 값이 된다.

또한 아래 코드로 타깃과 예측한 값 사이의 차이를 구할 수 있다.

```py
from sklearn.metrics import mean_absolute_error
# 테스트 세트에 대한 예측을 만듭니다
test_prediction = knr.predict(test_input)
# 테스트 세트에 대한 평균 절댓값 오차를 계산합니다
mae = mean_absolute_error(test_target, test_prediction)
print(mae)
```
위 코드를 실행하면 약 19가 나오고, 이는 예측이 타깃값과 19g정도 차이난다는 것을 알 수 있다.

# 과대 적합과 과소적합
```py
print(knr.score(train_input, train_target))
```
위 코드를 실행하며 훈련 세트를 얼마나 잘 맞추는지 보면 약 0.9688이 나오고 이는 테스트 세트보다 훈련 세트를 덜 맞춘다는 이야기가 된다. 보통은 훈련 세트로 훈련하므로 훈련 세트에 잘 맞는 모델이나오고, 이는 훈련 세트의 결정계수가 더 높게 나온다는 것을 의미한다.

훈련 세트에서 점수가 좋았는데 테스트 세트에서 점수가 굉장히 나쁘다면 **과대적합(overfitting)**되었다고 한다. 그리고 훈련 세트보다 테스트 세트의 점수가 더 높거나 두 점수 너무 낮은 경우 모델이 훈련 세트에 **과소 적합(underfitting)**되었다고 한다. 보통 과소적합은 훈련 세트와 테스트 세트의 크기가 매우 작기 때문에 일어난다.

따라서 위 모델은 과소적합되어있다. 이것을 해결하려면 훈련 세트에 더 잘 맞게 만들면 된다. K-최근접 이웃 회귀 알고리즘에서 모델을 훈련 세트에 더 맞게 만드는 방법은 이웃의 개수를 줄이는 것이다(이웃의 개수를 줄이면 각 훈련 샘플 하나하나에 민감해진다). 따라서 k값을 5에서 3으로 낮춰보자.

```py
# 이웃의 갯수를 3으로 설정합니다
knr.n_neighbors = 3
# 모델을 다시 훈련합니다
knr.fit(train_input, train_target)
print(knr.score(train_input, train_target))
print(knr.score(test_input, test_target))
```
위 코드를 실행하면 훈련 세트의 결정계수는 약 0.9805, 테스트 세트의 결정계수는 0.9746이 나오므로 과소적합 문제가 해결되었다.

# 확인문제2
```py
# k-최근접 이웃 회귀 객체를 만듭니다
knr = KNeighborsRegressor()
# 5에서 45까지 x 좌표를 만듭니다
x = np.arange(5, 45).reshape(-1, 1)

# n = 1, 5, 10일 때 예측 결과를 그래프로 그립니다.
for n in [1, 5, 10]:
    # 모델 훈련
    knr.n_neighbors = n
    knr.fit(train_input, train_target)
    # 지정한 범위 x에 대한 예측 구하기
    prediction = knr.predict(x)
    # 훈련 세트와 예측 결과 그래프 그리기
    plt.scatter(train_input, train_target)
    plt.plot(x, prediction)
    plt.title('n_neighbors = {}'.format(n))
    plt.xlabel('length')
    plt.ylabel('weight')
    plt.show()
```
위 코드를 실행하면 k-최근접 이웃 회귀 모델의 k 값을 1,5,10으로 바꿔가며 농어의 길이가 5~45일때 모델이 예측하는 무게를 알 수 있다.
![k-최근접 이웃 회귀 모델의 예측 그래프](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/aea48c09-9a12-47f0-90b6-8d3d886b06d1)
(k=5일때 예측 그래프)

# Reference
 - 박해선 저,<혼자 공부하는 머신러닝 + 딥러닝>, 한빛미디어, 2020
 - [https://github.com/rickiepark/hg-mldl](https://github.com/rickiepark/hg-mldl)