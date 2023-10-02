---
title: "2-1. 데이터 다루기 - 훈련 세트와 테스트 세트"
last_modified_at: 2023-10-02T20:40:12+09:00
categories:
    - hon-gong-machine
tags:
    - A.I

author_profile: true

toc: true
toc_label: "My Table of Contents"

---
# 지도 학습과 비지도 학습
지도학습은 훈련하기 위한 데이터와 정답이 필요하다. 데이터와 정답을 입력(input)과 타깃(target)이라 부르고, 이 둘을 합쳐
훈련 데이터라고 부른다. 지도학습은 정답을 맞히는 것을 학습한다. 지도학습에는 K-최근접 이웃 알고리즘이 있다.

비지도 학습은 입력데이터만 사용한다. 따라서 데이터를 잘 파악하거나 변형하는데 도움을 준다.

# 훈련 세트와 데이터 세트
앞서 만든 [도미와 빙어를 분류하는 모델](https://minchocoin.github.io/hon-gong-machine/3/)의 경우 도미와 빙어의 데이터와 타깃을 주고 훈련한 다음
같은 데이터로 테스트하였고, 따라서 정답을 모두 맞히는 것이 당연하다.

따라서 머신러닝 알고리즘의 성능을 제대로 평가하려면 훈련 데이터와 평가에 사용할 데이터가 달라야한다. 이렇게 하는 간단한 방법은 평가를 위한 데이터를 따로 준비하거나 이미 준비한 데이터 중 일부를 평가 데이터로 활용하는 것이고, 보통 후자가 많이 사용된다. 평가에 사용하는 데이터를 **테스트 세트** , 훈련에 사용하는 데이터를 **훈련 세트**라고 부른다. 이제 도미와 빙어 데이터를 훈련 세트와 테스트 세트로 나눠보자.

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

그리고 fish_length와 fish_weight를 순회하면서 각 생선의 길이와 무게를 하나에 담고, 정답 데이터 fish_target을 만든다.

```py
fish_data = [[l, w] for l, w in zip(fish_length, fish_weight)]
fish_target = [1]*35 + [0]*14
```

전체 데이터는 총 49개의 샘플이고, 이 데이터의 처음 35개를 훈련 세트로, 나머지 14개를 테스트 세트로 사용하자. 먼저 KNeighborsClassifier 객체를 만든다.

```py
from sklearn.neighbors import KNeighborsClassifier

kn = KNeighborsClassifier()
```

그 다음 fish_data와 fish_target 데이터를 파이썬의 리스트 슬라이싱 기능을 이용하여 train과 test 데이터로 분리한다.

```py
train_input = fish_data[:35] # fish_data의 0~34까지
train_target = fish_target[:35]# fish_target의 0~34까지

test_input = fish_data[35:] #fish_data의 35부터 끝까지
test_target = fish_target[35:]#fish_target의 35부터 끝까지
```
이제 fit()으로 모델을 훈련하고, score()로 얼마나 훈련이 잘되었는지 평가하자.

```py
kn.fit(train_input, train_target)
kn.score(test_input, test_target)
```
**이것을 실행하면 정확도가 0.0이 나온다.** 이유는 아래에 서술한다.

# 샘플링 편향

왜냐하면 훈련 세트에는 도미만, 테스트 세트에는 빙어만 들어가있기 때문이다. 훈련 세트에는 도미만 있기 때문에 무조건 도미로 분류하게 된다. 따라서 훈련세트와 테스트세트는 골고루 섞여있어야한다. 만약 골고루 섞여있지 않다면 이를 **샘플링 편향** 이라고 부른다.

# 넘파이
넘파이는 파이썬의 대표적인 배열 라이브러리이다.
다음 코드로 numpy 라이브러리를 임포트할 수 있다(먼저 cmd창에서 pip install numpy가 필요하다)

```py
import numpy as np
```

파이썬 리스트를 배열로 바꾸는 것은 아래 코드로 가능하다.

```py
input_arr = np.array(fish_data)
target_arr = np.array(fish_target)
```
또한 배열의 크기를 알려면 다음과 같은 코드로 확인할 수 있다.
```py
print(input_arr.shape)
```
위 코드를 실행하면 (49,2) 라는 결과가 나온다.

이제 input_arr과 target_arr을 무작위로 골라 일부는 테스트세트로, 일부는 훈련세트로 가야한다.주의할점은 input_arr과 target_arr에서 같은 위치는 함께 선택되어야한다는 점이다.

이때 사용할 방법은 0~48까지 인덱스를 무작위로 섞어 이 값을 input_arr과 target_arr에서 선택할 인덱스로 활용하면 된다. 인덱스를 랜덤하게 섞기 위해 다음과 같이 할 수 있다.

```py
np.random.seed(42)#랜덤에 사용할 시드를 설정. 시드가 같으면 똑같게 섞인다.
index = np.arange(49) #0~48까지 1씩 증가하는 배열을 만듦
np.random.shuffle(index)#무작위로 섞음
```
이제 index에는 0부터 48까지 값이 무작위로 배열되어있다.

따라서 다음과 같이 훈련 세트와 테스트 세트를 나눌 수 있다.

```py
#index의 0~34번째에 저장되어있는 인덱스에 저장되어있는 input_arr과 target_arr을 각각 train_input과 train_target으로 한다.
train_input = input_arr[index[:35]]
train_target = target_arr[index[:35]]

#index의 35번째부터 끝까지 저장되어있는 인덱스에 저장되어있는 input_arr과 target_arr을 각각 test_input과 test_target으로 한다.
test_input = input_arr[index[35:]]
test_target = target_arr[index[35:]]
```

이제 훈련 세트와 테스트 세트가 잘 섞였는지 산점도를 그려보자

```py
import matplotlib.pyplot as plt

plt.scatter(train_input[:, 0], train_input[:, 1])
plt.scatter(test_input[:, 0], test_input[:, 1])
plt.xlabel('length')
plt.ylabel('weight')
plt.show()
```
![scatterplot of train set and test set](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/b754bf08-057a-4eee-b761-f19421a20c24)

위 사진에서, 파란색 점은 훈련 세트이고, 주황색은 테스트 세트이다. 잘 섞여져있으므로, 다시 학습하여보자.

# 두 번째 머신러닝 프로그램
```py
kn.fit(train_input, train_target)
kn.score(test_input, test_target)
```
이제 위 코드를 실행시키면 정확도가 1.0이 나온다.

아래 코드로 어떻게 예측했는지 확인할 수 있다.
```py
kn.predict(test_input)
```
# Reference
 - 박해선 저,<혼자 공부하는 머신러닝 + 딥러닝>, 한빛미디어, 2020
 - [https://github.com/rickiepark/hg-mldl](https://github.com/rickiepark/hg-mldl)