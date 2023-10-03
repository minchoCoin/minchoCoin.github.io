---
title: "3-2. 회귀 알고리즘와 모델 규제 - 선형회귀"
last_modified_at: 2023-10-03T19:40:12+09:00
categories:
    - hon-gong-machine
tags:
    - A.I

author_profile: true

toc: true
toc_label: "My Table of Contents"

---
# K-최근접 이웃의 한계
이전과 같이 농어의 무게를 예측하는 K-최근접 이웃 회귀 모델을 만들어보자
```py
import numpy as np

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

from sklearn.model_selection import train_test_split

# 훈련 세트와 테스트 세트로 나눕니다
train_input, test_input, train_target, test_target = train_test_split(
    perch_length, perch_weight, random_state=42)
# 훈련 세트와 테스트 세트를 2차원 배열로 바꿉니다
train_input = train_input.reshape(-1, 1)
test_input = test_input.reshape(-1, 1)

from sklearn.neighbors import KNeighborsRegressor

knr = KNeighborsRegressor(n_neighbors=3)
# k-최근접 이웃 회귀 모델을 훈련합니다
knr.fit(train_input, train_target)
```
이제 아래 코드를 실행하면
```py
print(knr.predict([[50]]))
print(knr.predict([[100]]))
```
1033.33333333, 1033.33333333이 출력된다. 어떻게 길이가 2배차이가 나는데 무게는 같다고 예측을 하는 것일까?

이는 그래프를 그려보면 알 수 있다.
```py
# 100cm 농어의 이웃을 구합니다
distances, indexes = knr.kneighbors([[100]])

# 훈련 세트의 산점도를 그립니다
plt.scatter(train_input, train_target)
# 훈련 세트 중에서 이웃 샘플만 다시 그립니다
plt.scatter(train_input[indexes], train_target[indexes], marker='D')
# 100cm 농어 데이터
plt.scatter(100, 1033, marker='^')
plt.xlabel('length')
plt.ylabel('weight')
plt.show()
```
![50cm 농어에 대한 예측](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/e1b8534b-0146-40cd-87e5-6cf8ac15c94d)

(사진1: 100cm 농어에 대한 예측(세모)과 훈련 데이터(파란색),100cm 농어와 가장 가깝다고 계산한 훈련 샘플(주황색) )

사진1에서와 같이, 길이가 커지면 커질수록 무게가 증가하는 경향이 있지만, 100cm 농어에서 가장 가까운것은 45cm 근방이라서 45cm 근방에 있는 샘플의 무게의 평균을 내기 때문에, 1033이라는 결과가 나온다. 따라서 이럴 때는 선형회귀를 써야한다.

# 선형회귀

선형회귀는 널리 사용되는 대표적인 회귀 알고리즘이다. 선형회귀는 주어진 데이터를 잘 표현하는 하나의 직선을 찾는다.
아래와 같이 사이킷런을 이용하여 선형회귀를 사용할 수 있다.

```py
from sklearn.linear_model import LinearRegression

lr = LinearRegression()
# 선형 회귀 모델 훈련
lr.fit(train_input, train_target)

# 50cm 농어에 대한 예측
print(lr.predict([[50]]))
```
이렇게 하면 50cm 농어에 대하여 1241.83의 무게를 가졌다고 예측한다. 또한 선형회귀에서 찾은 직선은 $y=ax+b$ 로 표현할 수 있으며 사이킷런의 linearRegression 클래스에서 a는 coef_, b는 intercept_ 에 저장되어있다. 여기서 coef_와 intercept_는 모델이 찾은 값이라는 의미로 모델 파라미터라고 부른다(모델 기반 학습).

```py
print(lr.coef_, lr.intercept_)
```
따라서 위 코드를 실행시키면 lr.coef_는 약 39.017, lr.intercept_는 약 -709.018이 나온다.

이제 농어의 길이 15에서 50까지 linear Regression으로 나온 직선을 그려보자. 즉 (15,15xcoef + intercept)와 (50,50xcoef+intercept)를 잇는 직선을 그리면 된다. 따라서 아래와 같이 코딩할 수 있다.

```py
# 훈련 세트의 산점도를 그립니다
plt.scatter(train_input, train_target)
# 15에서 50까지 1차 방정식 그래프를 그립니다
plt.plot([15, 50], [15*lr.coef_+lr.intercept_, 50*lr.coef_+lr.intercept_])
# 50cm 농어 데이터
plt.scatter(50, 1241.8, marker='^')
plt.xlabel('length')
plt.ylabel('weight')
plt.show()
```
![농어 데이터를 기반으로 나온 추세 직선 그래프](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/d6503463-40fd-4b7c-b4c3-a00ea0986855)
(사진2: 농어 데이터를 기반으로 나온 추세 직선 그래프)

사진2와 같이 추세선을 기반으로 새로운 농어 데이터에 대해 잘 예측하였다(주황색 삼각형).

이제 $R^2$ 점수를 보자
```py
print(lr.score(train_input, train_target))
print(lr.score(test_input, test_target))
```
훈련 세트와 테스트 세트의 점수가 조금 차이나고, 훈련 세트의 점수도 높지 않다. 왜냐하면 데이터 분포 특성상, 직선으로는 완벽히 농어 데이터를 반영할 수 없기 때문이다. 아래에 더 자세히 서술한다.

# 다항 회귀
농어의 데이터 분포를 보면 일직선이라기보다 왼쪽 위로 구부러진 곡선에 가깝다. 따라서 최적의 직선이 아닌 최적의 곡선을 찾아보자. 즉 $무게 = a \times (길이)^2 + b\times (길이) + c$ 에서 최적의 a,b,c를 찾는 것이다. 이를 위해 먼저 $((길이)^2, (길이))$의 행렬을 만들자. 아래와 같이 만들 수 있다.

```py
train_poly = np.column_stack((train_input ** 2, train_input))
test_poly = np.column_stack((test_input ** 2, test_input))
```

이제 train_poly를 이용하여 선형 회귀 모델을 다시 훈련하자. 훈련한 다음 50cm 농어에 대해 무게를 예측해보자. 테스트 할 때는 모델에 농어 길이의 제곱과 원래 길이를 함께 넣어주어야한다.

```py
lr = LinearRegression()
lr.fit(train_poly, train_target)

print(lr.predict([[50**2, 50]]))
```
이제 모델은 50cm 농어에 대해 약 1573.98의 무게를 가졌다고 예측한다.

아래 코드를 통해 모델이 예측한 a,b,c 값을 보면
```py
print(lr.coef_, lr.intercept_)
```
a는 약 1.01, b는 약 -21.6, c는 116.05로 예측하였다. 따라서 모델이 예측한 곡선의 방정식은 x가 길이, y가 무게일때
$ y=1.01x^2 - 21.6x + 116.05$ 이다.

다시 아래와 같이 그래프를 그려보자

```py
# 구간별 직선을 그리기 위해 15에서 49까지 정수 배열을 만듭니다
point = np.arange(15, 50)
# 훈련 세트의 산점도를 그립니다
plt.scatter(train_input, train_target)
# 15에서 49까지 2차 방정식 그래프를 그립니다
plt.plot(point, 1.01*point**2 - 21.6*point + 116.05)
# 50cm 농어 데이터
plt.scatter([50], [1574], marker='^')
plt.xlabel('length')
plt.ylabel('weight')
plt.show()
```
그러면 아래와 같이 그래프가 나온다.

![모델이 예측한 추세 곡선](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/3c9ebf89-5d8b-47a1-86dd-c92a431080ca)

(사진3: 모델이 예측한 추세 곡선)

사진3에서 알 수 있듯, 단순 선형 회귀 모델보다 나은 그래프가 그려졌다. 이제 $R^2$ 점수를 보자

```py
print(lr.score(train_poly, train_target))
print(lr.score(test_poly, test_target))
```
점수는 각각 약 0.9707, 약 0.9776이 나왔다. 아까보다는 더 좋아졌지만, 아직 과소적합이 남아있는 듯하다. 좀 더 복잡한 모델이 필요할 것 같다.

# Reference
 - 박해선 저,<혼자 공부하는 머신러닝 + 딥러닝>, 한빛미디어, 2020
 - [https://github.com/rickiepark/hg-mldl](https://github.com/rickiepark/hg-mldl)