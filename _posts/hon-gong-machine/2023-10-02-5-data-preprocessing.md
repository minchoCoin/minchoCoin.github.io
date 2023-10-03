---
title: "2-2. 데이터 다루기 - 데이터 전처리"
last_modified_at: 2023-10-02T22:40:12+09:00
categories:
    - hon-gong-machine
tags:
    - A.I

author_profile: true

toc: true
toc_label: "My Table of Contents"

---
# 넘파이로 데이터 준비하기
도미와 빙어 데이터를 준비할 때 넘파이를 이용해보자. 먼저 생선 데이터를 준비한다.

```py
fish_length = [25.4, 26.3, 26.5, 29.0, 29.0, 29.7, 29.7, 30.0, 30.0, 30.7, 31.0, 31.0,
                31.5, 32.0, 32.0, 32.0, 33.0, 33.0, 33.5, 33.5, 34.0, 34.0, 34.5, 35.0,
                35.0, 35.0, 35.0, 36.0, 36.0, 37.0, 38.5, 38.5, 39.5, 41.0, 41.0, 9.8,
                10.5, 10.6, 11.0, 11.2, 11.3, 11.8, 11.8, 12.0, 12.2, 12.4, 13.0, 14.3, 15.0]
fish_weight = [242.0, 290.0, 340.0, 363.0, 430.0, 450.0, 500.0, 390.0, 450.0, 500.0, 475.0, 500.0,
                500.0, 340.0, 600.0, 600.0, 700.0, 700.0, 610.0, 650.0, 575.0, 685.0, 620.0, 680.0,
                700.0, 725.0, 720.0, 714.0, 850.0, 1000.0, 920.0, 955.0, 925.0, 975.0, 950.0, 6.7,
                7.5, 7.0, 9.7, 9.8, 8.7, 10.0, 9.9, 9.8, 12.2, 13.4, 12.2, 19.7, 19.9]
```
그리고 numpy를 임포트한다.
```py
import numpy as np
```
numpy의 column_stack() 함수는 전달받은 리스트들을 차례대로 나란히 연결한다. 에를 들어 다음 코드는
```py
np.column_stack(([1,2,3], [4,5,6]))
```
아래와 같은 numpy 배열을 만든다.
```py
array([[1, 4],
       [2, 5],
       [3, 6]])
```
이제 fish_length와 fish_weight를 합치자. 다음과 같이 코드를 입력한다.
```py
fish_data = np.column_stack((fish_length, fish_weight))
```
그 다음 정답데이터를 만들자. 정답데이터는 np.ones()와 np.zeros()를 이용하여 만든다. 괄호안에 숫자만큼 각각 1과 0으로 채운 배열을 만든다. 따라서 정답데이터는 다음과 같이 만들수 있다.

```py
fish_target = np.concatenate((np.ones(35), np.zeros(14)))
```

# 사이킷런으로 훈련 세트와 데이터 세트 나누기
사이킷런의 train_test_split()함수는 비율에 맞게 훈련 세트와 테스트 세트로 나누고, 알아서 섞어준다.
아래 코드로 random seed 42를 이용하여 섞고 훈련 세트와 테스트 세트로 나눈다.
```py
from sklearn.model_selection import train_test_split

train_input, test_input, train_target, test_target = train_test_split(
    fish_data, fish_target, random_state=42)
```
이때 print(test_target)을 해보면 1과 0의 비율이 맞지 않는다. 즉 샘플링 편향이 일어난 것이다. 이것을 해결하기 위해 stratify 매개변수에 fish_target을 전달하면, fish_target의 1과 0의 비율을 맞춰 데이터를 나눈다. 다음 코드로 다시 나누자.

```py
train_input, test_input, train_target, test_target = train_test_split(
    fish_data, fish_target, stratify=fish_target, random_state=42)
```
데이터가 적어 완전히 맞출 수 없지만, 어느정도 비율이 맞다(2.25:1)

# 수상한 도미 한마리
```py
from sklearn.neighbors import KNeighborsClassifier

kn = KNeighborsClassifier()
kn.fit(train_input, train_target)
kn.score(test_input, test_target)
```
위 코드를 실행하면 정확도가 1.0이 나온다. 이제 새로운 도미 데이터(25,150)을 넣고 결과를 확인하자.

```py
print(kn.predict([[25, 150]]))
```
위 코드를 실행하면 도미(1)로 예측하지 않고, 빙어(0)으로 예측한다.
이유를 알기 위해 먼저 아래 코드로 (25,150)과 가장 가까운 이웃의 거리와 인덱스를 찾자.

```py
distances, indexes = kn.kneighbors([[25, 150]])
```
그다음 그 인덱스와 도미,빙어 데이터를 산점도로 그려보자

```py
plt.scatter(train_input[:,0], train_input[:,1])
plt.scatter(25, 150, marker='^')
plt.scatter(train_input[indexes,0], train_input[indexes,1], marker='D')
plt.xlabel('length')
plt.ylabel('weight')
plt.show()
```
![scatter of kneighbors,bream and smelt data](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/a0ebdb6a-dc96-416b-9492-d7e34674476a)
위 산점도에서 삼각형이 새로운 샘플이고, 마름모 모양이 새로운 샘플에서 가장가까운 5개 샘플이다.

그리고 distance의 값을 출력해보면 아래와 같다.
```py
[[ 92.00086956 130.48375378 130.73859415 138.32150953 138.39320793]]
```
위 산점도와 distance의 값을 고려하였을 때 산점도에서는 92거리가 130 거리보다 몇배는 더 크다고 보인다. 이는 x축의 범위가 좁고 y축의 범위가 커서 생기는 문제로, 따라서 y축으로 조금만 멀어져도 아주 큰 값으로 계산된다. 따라서 스케일(값이 놓인 범위)을 맞춰줘야한다. 이때 평균과 표준편자가 쓰인다.

아래 코드로 평균과 표준편자를 계산할 수 있다.
```py
mean = np.mean(train_input, axis=0)
std = np.std(train_input, axis=0)
#axis=0: 행을 따라 각 열의 통계 값 계산
```
그리고 계산한 평균과 표준편차를 이용하여 훈련 데이터 스케일에 맞춘 train_scaled와 test_scaled, new((25,150)을 train scale에 맞춤)를 만든다.
```py
train_scaled = (train_input - mean) / std
test_scaled = (test_input - mean) / std
new = ([25, 150] - mean) / std
```
이제 아래와 같이 훈련하고 테스트한다.
```py
kn.fit(train_scaled, train_target)
kn.score(test_scaled, test_target)
```
그러면 정확도가 1.0이 나온다. 이제 새로운 데이터를 잘 맞추는지 테스트하자

```py
print(kn.predict([new]))
```
이제 위 코드를 실행하면 도미(1)로 예측한다.
Kneighbors가 가장가까운 5개지점을 어디로 생각했는지 확인해보자
```py
distances, indexes = kn.kneighbors([new])
plt.scatter(train_scaled[:,0], train_scaled[:,1])
plt.scatter(new[0], new[1], marker='^')
plt.scatter(train_scaled[indexes,0], train_scaled[indexes,1], marker='D')
plt.xlabel('length')
plt.ylabel('weight')
plt.show()
```
![scatter of kneighbors,bream and smelt data](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/a118cc98-0b80-4b45-91ee-35ae267214dd)

이제 삼각형(새로운 데이터)에 가장 가까운 샘플을 도미라고 계산한다.