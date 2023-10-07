---
title: "선택 분류 - 불량 철판 분류 신경망"
last_modified_at: 2023-10-06T23:53:12+09:00
categories:
    - alzza-deeplearning
tags:
    - A.I

toc: true
toc_label: "My Table of Contents"
author_profile: true

---
철판과 관련된 데이터 27가지를 이용하여 철판을 7가지로 분류해보자.

# abalone.ipynb 실행
```py
%run ../chap01/abalone.ipynb
```
abalone.ipynb의 코드를 재활용한다.

# 메인 함수 정의
```py
def steel_exec(epoch_count=10, mb_size=10, report=1):
    load_steel_dataset()
    init_model()
    train_and_test(epoch_count, mb_size, report)
```
load_steel_dataset()을 호출하여 steel_dataset을 로드하고, 모델을 초기화하고 훈련및 테스트를 수행한다.

# 데이터 적재 함수 정의
```py
def load_steel_dataset():
    with open('../../data/chap03/faults.csv') as csvfile:
        csvreader = csv.reader(csvfile)
        next(csvreader, None)
        rows = []
        for row in csvreader:
            rows.append(row)
            
    global data, input_cnt, output_cnt
    input_cnt, output_cnt = 27, 7
    data = np.asarray(rows, dtype='float32')
```
faults.csv파일을 배열로 바꾸고 그 배열을 넘파이 배열로 바꾼다. 또한 입력으로 들어온 27개의 데이터를 7개 항목에 대한 로짓값으로 바꾸어야하므로 input_cnt, output_cnt = 27,7로 지정한다.

# 후처리 과정에 대한 순전파와 역전파 함수 재정의
```py
def forward_postproc(output, y):
    entropy = softmax_cross_entropy_with_logits(y, output)
    loss = np.mean(entropy) 
    return loss, [y, output, entropy]

def backprop_postproc(G_loss, aux):
    y, output, entropy = aux
    
    g_loss_entropy = 1.0 / np.prod(entropy.shape)
    g_entropy_output = softmax_cross_entropy_with_logits_derv(y, output)
    
    G_entropy = g_loss_entropy * G_loss
    G_output = g_entropy_output * G_entropy
    
    return G_output
```
앞서 펄서 분류 신경망에서 sigmoid가 softmax로 바뀐 것 밖에 없다. y와 output은 (미니배치크기, 출력벡터크기)이며 entropy는 (미니배치크기,1)이다. g_entropy_output도 (미니배치크기, 출력벡터크기)이다.

# 정확도 함수 재정의
```py
def eval_accuracy(output, y):
    estimate = np.argmax(output, axis=1)
    answer = np.argmax(y, axis=1)
    correct = np.equal(estimate, answer)
    
    return np.mean(correct)
```
철판 데이터의 output은 (N,7)이므로, 각 데이터로 산출된 로짓값 분포가 가로로 나열되어있고, 로짓값 분포 세트가 세로로 나열되어있다. np.argmax(output,axis=1)은 각 데이터로 산출된 로짓값 분포에서 각 분포마다 최대값(로짓값을 클 수록 확률이 크다)의 인덱스를 찾아 저장하고, (N,1)의 형태가 된다. 그리고 신경망이 추정한 것과 실제 답에서 같은 것은 1, 아닌 것은 0으로 두어 평균을 구하면 정확도가 산출되게 된다.

# 소프트맥스 관련 함수 정의
```py
def softmax(x):
    max_elem = np.max(x, axis=1)
    diff = (x.transpose() - max_elem).transpose()
    exp = np.exp(diff)
    sum_exp = np.sum(exp, axis=1)
    probs = (exp.transpose() / sum_exp).transpose()
    return probs

def softmax_derv(x, y):
    mb_size, nom_size = x.shape
    derv = np.ndarray([mb_size, nom_size, nom_size])
    for n in range(mb_size):
        for i in range(nom_size):
            for j in range(nom_size):
                derv[n, i, j] = -y[n,i] * y[n,j]
            derv[n, i, i] += y[n,i]
    return derv

def softmax_cross_entropy_with_logits(labels, logits):
    probs = softmax(logits)
    return -np.sum(labels * np.log(probs+1.0e-10), axis=1)

def softmax_cross_entropy_with_logits_derv(labels, logits):
    return softmax(logits) - labels
```
소프트맥스는 로짓 벡터 $(x_1,x_2,x_3,...,x_n)$ 에서 최대값((1행의 최댓값, 2행의 최댓값... n행의최댓값)의 행 벡터로 나타남)을 찾고 그 값을 벡터안에 있는 로짓값마다 빼준다(각 열에 대해 반복 계산 하기위해 transpose한다). 그리고 각 로짓값마다 $e^{x_i}$ 함수를 적용시키고 각 로짓값에 대응하는 확률을 구한다. 따라서 입력으로 (N,7)이 들어왔으면 출력도 (N,7)이다.

softmax_cross_entropy_with_logits은 소프트맥스 크로스엔트로피 함수로 입력이 (N,7), (N,7)이면 두 확률 분포의 유사도를 하나의 값으로 나타내주기 때문에($\sum p_i \log q_i$) (N,1)로 출력된다.

softmax_cross_entropy_with_logits_derv 는 소프트맥스 크로스엔트로피 편미분 함수로, 출력은 (N,7)이다. 이것을 이용해 각 로짓값을 얼마나 조정해야하는지 계산하게 된다.

# 실행 결과
![철판 분류 신경망 실행 결과](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/4f52e9c2-7ef4-48a0-a232-2842fc60bcd3)

# 시그모이드 함수 적용해보기
forward_postproc()과 backward_postproc()함수에 sigmoid함수를 적용시키면 넘파이 행렬의 성질에 따라 아래 시그모이드 함수는

```py
def relu(x):
    return np.maximum(x, 0)

def sigmoid(x):
    return np.exp(-relu(-x)) / (1.0 + np.exp(-np.abs(x)))
        
def sigmoid_derv(x, y):
    return y * (1 - y)

def sigmoid_cross_entropy_with_logits(z, x):
    return relu(x) - x * z + np.log(1 + np.exp(-np.abs(x)))

def sigmoid_cross_entropy_with_logits_derv(z, x):
    return -z + sigmoid(x)
```
로짓 벡터 $(x_1,x_2,...,x_n)$ 에 대해 각 로짓값의 시그모이드 함수를 적용한 벡터가 나온다. 단 합이 1이 보장되지는 않는다.

![시그모이드 함수 적용](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/a2f7c188-4858-4879-9931-5649cbbb9757)

위 사진은 시그모이드 함수를 적용했을 때 결과이다.

# 시그모이드 함수와 MSE 적용하기
```py
def forward_postproc(output, y):
    diff = sigmoid(output) - sigmoid(y) # 실제 로짓값과 예측한 로짓값을 시그모이드 함수를 적용시켜 차이를구함
    square = np.square(diff)
    loss = np.mean(square)
    return loss, [diff,output,y]

def backprop_postproc(G_loss, aux):
    diff,output,y = aux
    shape = diff.shape
    g_loss_square = np.ones(shape) / np.prod(shape)
    g_square_diff = 2*diff * sigmoid_derv(np.ones(np.shape(output)),sigmoid(output))
    # L = (sigmoid(output) - sigmoid(y))^2/AB 을 output에 편미분 하면 2*(sigmoid(output) - sigmoid(y))* sigmoid_derv(y,output)/ AB
    g_diff_output = 1

    G_square = g_loss_square * G_loss
    G_diff = g_square_diff * G_square
    G_output = g_diff_output * G_diff
    
    return G_output
```
위와 같이 시그모이드 함수와 Mean Squared error를 적용시키면 다음과같은 결과가 나온다.

![시그모이드 MSE 적용 결과](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/8b9d4282-5576-4423-a128-e03cda7184e6)

# Reference

 - 윤덕호 저, [파이썬 날코딩으로 알고 짜는 딥러닝], 한빛미디어, 2019


