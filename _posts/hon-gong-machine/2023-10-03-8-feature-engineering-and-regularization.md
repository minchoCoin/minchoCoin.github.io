---
title: "3-3. 회귀 알고리즘와 모델 규제 - 특성공학과 규제"
last_modified_at: 2023-10-03T19:40:12+09:00
categories:
    - hon-gong-machine
tags:
    - A.I

author_profile: true

toc: true
toc_label: "My Table of Contents"

---
# 다중 회귀
농어 무게에 대한 예측에서, 농어의 길이뿐만 아니라 농어의 높이과 두께도 함께 사용하여 예측하여보자. 또한 이전 절에서 처럼 3개의 특성을 각각 제곱한 것을 추가한다. 도한 각 특성을 서로 곱하여 또 다른 특성을 만들 수도 있다. 이렇게 기존의 특성을 사용해 새로운 특성을 뽑아내는 작업을 특성 공학이라고 부른다.

이것을 사이킷런에서 제공하는 함수로 만들어보자.

# 데이터 준비
농어의 특성이 3개로 늘어나 일일이 복사해 붙여넣는 것도 번거롭기 때문에, pandas를 사용하여 데이터를 불러오자.

pandas는 데이터 분석 라이브러리로, pandas에서 제공하는 데이터프레임은 판다스의 핵심 데이터 구조이다. 판다스를 이용하여 csv파일을 읽는 방법은 pd.read_csv(파일위치)로 하면 된다. 따라서 아래와 같이 농어의 특성 데이터를 불러올 수 있다.

```py
df = pd.read_csv('https://bit.ly/perch_csv_data')
perch_full = df.to_numpy()
print(perch_full)
```

타깃 데이터는 아래와 같이 준비한다.
```py
import numpy as np

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
그리고 훈련데이터와 테스트 데이터로 분리한다.
```py
from sklearn.model_selection import train_test_split

train_input, test_input, train_target, test_target = train_test_split(perch_full, perch_weight, random_state=42)
```

이제 사이킷런의 변환기를 이용하여 농어의 특성을 제곱하고 조합하여 새로운 특성을 만들자.

```py
poly = PolynomialFeatures(include_bias=False) # 절편을 만들지 않음

poly.fit(train_input)
train_poly = poly.transform(train_input)
print(train_poly.shape)
```
마지막 print를 통해 train_poly는 42개의 데이터와 각각의 데이터는 9개의 특성을 가지고 있는 것을 알 수 있다.

아래 코드로 특성을 어떻게 만들었는지 볼 수 있다.
```py
poly.get_feature_names_out()
```
```py
array(['x0', 'x1', 'x2', 'x0^2', 'x0 x1', 'x0 x2', 'x1^2', 'x1 x2',
       'x2^2'], dtype=object)
```
위와 같이 첫번쨰특성, 두번째특성, 세번째특성, 첫번쨰특성의 제곱,..., 세번쨰 특성의 제곱 이렇게 특성을 만든 것을 알 수 있다.

이제 테스트 세트도 같이 변환한다.
```py
test_poly = poly.transform(test_input) # train set를 기준으로 테스트 세트를 변환한다.
```
이제 이것으로 linear Regression 모델을 훈련해보자
```py
from sklearn.linear_model import LinearRegression

lr = LinearRegression()
lr.fit(train_poly, train_target)
print(lr.score(train_poly, train_target))
print(lr.score(test_poly, test_target))
```
각각 0.99, 0.971이 나온다. 즉 과소적합 문제는 해결된 것으로 보인다. 근데, 그러면 3승 4승 항을 넣으면 더 정확해지지 않을까? 아래 코드로 한번 확인해보자.

```py
poly = PolynomialFeatures(degree=5, include_bias=False)#5제곱항까지 넣는다.

poly.fit(train_input)
train_poly = poly.transform(train_input)
test_poly = poly.transform(test_input)
```
이제 다시 아래 코드로 훈련하면 0.9999999999996433 가 나온다. 엄청 정확해졌다! 그러면 테스트 세트에 대해서 점수를 확인해보자.
```py
lr.fit(train_poly, train_target)
print(lr.score(train_poly, train_target))
```
아래 코드를 실행하면 점수가 -144가 나온다. 즉, 훈련 세트에 너무 과적합되어 테스트 세트에는 낮은 점수를 보인 것 같다.
```py
print(lr.score(test_poly, test_target))
```

# 규제
규제는 머신러닝 모델이 훈련 세트를 과도하게 학습하지 못하도록 하게 하는 것이다. 선형 회귀 모델에서는 특성에 곱해지는 계수의 크기를 작게 만드는 일이다.

앞서 5제곱항까지 넣은 특성으로 훈련한 모델의 계수를 규제하여 테스트 세트의 점수를 높여보자. 이때 특성의 스케일이 모두 다르므로, 평균과 표준편차를 이용해 스케일을 똑같게 만들어야한다. 이때 사이킷런의 StandardScaler를 사용하자.

```py
from sklearn.preprocessing import StandardScaler

ss = StandardScaler()
ss.fit(train_poly) # train_poly의 평균과 표준편차를 내부에서 구한다.

train_scaled = ss.transform(train_poly) # train_poly의 평균과 표준편차를 이용하여 scale을 맞춘다.
test_scaled = ss.transform(test_poly) #train_poly의 평균과 표준편차를 이용하여 train set와 같이 test set도 scale을 맞춘다.
```

이제 선형회귀에 규제를 추가한 모델 릿지와 라쏘를 이용하여 학습하자. 릿지(ridge)는 계수를 제곱한 값을 기준으로 규제를 적용하고, 라쏘(lasso)는 계수의 절댓값을 기준으로 규제를 적용한다.

## 릿지 회귀
```py
from sklearn.linear_model import Ridge

ridge = Ridge()
ridge.fit(train_scaled, train_target)
print(ridge.score(train_scaled, train_target))
print(ridge.score(test_scaled, test_target))
```
릿지로 학습하였을때 훈련 세트에 대한 점수는 0.9896이 나왔고 테스트 세트에 대해서는 0.979가 나왔다. 즉 많은 특성을 사용했음에도, 규제덕분에 과대적합이 되지 않았다.

또 Ridge()에서 괄호안에 alpha 값을 적용할 수 있다. alpha는 규제의 양으로, 값이 크면 규제 강도가 더 세져 과소적합되도록 유도한다.

이제 아래와 같이 alpha 값에 따른 정확도를 분석하여 보자
```py
import matplotlib.pyplot as plt

train_score = []
test_score = []

alpha_list = [0.001, 0.01, 0.1, 1, 10, 100]
for alpha in alpha_list:
    # 릿지 모델을 만듭니다
    ridge = Ridge(alpha=alpha)
    # 릿지 모델을 훈련합니다
    ridge.fit(train_scaled, train_target)
    # 훈련 점수와 테스트 점수를 저장합니다
    train_score.append(ridge.score(train_scaled, train_target))
    test_score.append(ridge.score(test_scaled, test_target))

plt.plot(np.log10(alpha_list), train_score)
plt.plot(np.log10(alpha_list), test_score)
plt.xlabel('alpha')
plt.ylabel('R^2')
plt.show()
```

![alpha 값에 따른 모델의 정확도 그래프-ridge](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/24381514-ef57-4056-a0c0-b660eb709ec8)

여기서 x축의 -3,-2,...,2는 각각 0.001,0.01...,100에 상용로그를 취한 값이다. 위 그래프와 같이, alpha=0.001일때는 과대적합되는 모습을, alpha=100일떄 과소적합되는 모습을 보여준다. 여기서는 -1 즉 0.1일때 가장 테스트 세트에 대해 정확도가 좋다.

## 라쏘 회귀
위 코드에서 ridge만 lasso로 바꿔주면 된다.

```py
from sklearn.linear_model import Lasso

train_score = []
test_score = []

alpha_list = [0.001, 0.01, 0.1, 1, 10, 100]
for alpha in alpha_list:
    # 라쏘 모델을 만듭니다
    lasso = Lasso(alpha=alpha, max_iter=10000)
    # 라쏘 모델을 훈련합니다
    lasso.fit(train_scaled, train_target)
    # 훈련 점수와 테스트 점수를 저장합니다
    train_score.append(lasso.score(train_scaled, train_target))
    test_score.append(lasso.score(test_scaled, test_target))

plt.plot(np.log10(alpha_list), train_score)
plt.plot(np.log10(alpha_list), test_score)
plt.xlabel('alpha')
plt.ylabel('R^2')
plt.show()
```
![alpha 값에 따른 모델의 정확도 그래프-lasso](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/811cf430-c350-40d9-81b3-371971d8b16d)

lasso의 경우 그래프에서 x가 1일 떄, 즉 alpha가 10일 때 제일 높은 정확도를 보여준다.

아래 코드를 통해

```py
print(np.sum(lasso.coef_ == 0))
```
lasso 모델이 얼마나 많은 특성을 무시했는지 알 수 있으며 여기서는 40개의 특성의 계수를 0으로 하여 그 특성을 무게를 예측하는데 사용하지 않았다.
# Reference
 - 박해선 저,<혼자 공부하는 머신러닝 + 딥러닝>, 한빛미디어, 2020
 - [https://github.com/rickiepark/hg-mldl](https://github.com/rickiepark/hg-mldl)