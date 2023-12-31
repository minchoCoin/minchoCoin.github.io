---
title: "회귀 분석 - 전복의 고리수 추정 신경망"
last_modified_at: 2023-09-25T23:53:12+09:00
categories:
    - alzza-deeplearning
tags:
    - A.I

toc: true
toc_label: "My Table of Contents"
author_profile: true

---
# 목적
[파이썬 날코딩으로 알고 짜는 딥러닝] 책에서 첫번째로 나오는 전복의 고리 수 추정 신경망 파이썬 코드를 분석해보고자 한다.
# 데이터
kaggle의 전복 데이터셋은 4000여마리의 전복에 대해 8가지 특징값과 전복의 고리 수가 들어있다.

# 단층 퍼셉트론 신경망
[https://minchocoin.github.io/alzza-deeplearning/1/](https://minchocoin.github.io/alzza-deeplearning/1/)

# 코드 분석

## 파이썬 모듈 불러들이기
```py
import numpy as np
import csv
import time

np.random.seed(1234)
def randomize(): np.random.seed(time.time())
```
행렬 연산 등에 사용하는 넘파이 모듈과 데이터셋 csv파일을 읽는데 사용하는 csv모듈, 난수 초기화에 사용하는 time 모듈이 있다. 랜덤시드는 1234로 고정이며, 원한다면 randomize()로 랜덤화할 수도 있다.

## 하이퍼파라미터값의 정의
```py
RND_MEAN = 0
RND_STD = 0.0030

LEARNING_RATE = 0.001
```
정규분포 난수값에서 RND_MEAN는 평균이고, RND_STD는 표준편차이다. 즉 저 값으로 설정하고 정규분포 난수값을 무수히 많이 생성한다면, 0이 가장 많이 생성되고, 0에서 멀어질수록 덜 생성되는, 숫자별 확률분포를 그린다면 정규분포(0,0.003)가 나올 것이다. LEARNING_RATE는 학습률이다.

## 실험용 메인 함수 정의

```py
def abalone_exec(epoch_count=10, mb_size=10, report=1):
    load_abalone_dataset()
    init_model()
    train_and_test(epoch_count, mb_size, report)
```
위 함수는 먼저 데이터셋을 읽는 함수를 호출하고, 모델의 파라미터를 초기화하는 함수를 호출하고, 마지막으로 매개변수로 받은 epoch_count,mb_size,report를 train_and_test함수에 전달하고 호출하여 실제로 훈련과 테스트를 한다.

## 데이터 적재 함수 정의

```py
def load_abalone_dataset():
    with open('../../data/chap01/abalone.csv') as csvfile:
        csvreader = csv.reader(csvfile)
        next(csvreader, None)
        rows = []
        for row in csvreader:
            rows.append(row)
            
    global data, input_cnt, output_cnt
    input_cnt, output_cnt = 10, 1
    data = np.zeros([len(rows), input_cnt+output_cnt])

    for n, row in enumerate(rows):
        if row[0] == 'I': data[n, 0] = 1
        if row[0] == 'M': data[n, 1] = 1
        if row[0] == 'F': data[n, 2] = 1
        data[n, 3:] = row[1:]
```
먼저 csv파일을 연다. 그리고 next함수를 이용하여 헤더정보는 읽지 않도록 한다. 그리고 csvreader의 각 행을 rows라는 배열에 넣는다.

그리고 input_cnt와 output_cnt에는 각각 10과 1이 들어가는데, 각각 입력 벡터의 크기와 출력 벡터의 크기이다. data는 입출력벡터 정보를 저장하는 공간이다. rows 배열과의 차이점은 data는 성별 정보가 원-핫 벡터로 표시된다는 점이다.

data의 0번째,1번째,2번째 열을 성별로 사용하는데, 유충이면 0번째열만, 수컷이면 1번째열만 , 암컷이면 2번째열만 1로 설정한다. data의 3번째 열부터는 rows의 데이터를 그대로 붙인다.

## 파라미터 초기화 함수 정의
```py
def init_model():
    global weight, bias, input_cnt, output_cnt
    weight = np.random.normal(RND_MEAN, RND_STD,[input_cnt, output_cnt])
    bias = np.zeros([output_cnt])
```
weight 행렬의 크기를 [10,1]로 하고, bias의 크기를 [1]로 하여 각각 초기화한다. weight는 정규분포를 갖는 난숫값으로 초기화하는데, 경사하강법을 시작할 때 다양한 출발지에서 출발하도록 하기 위함이다. 즉 파라미터의 초기값을 여러개 해보는 것이다. bias는 초기에는 오히려 학습에 역효과만 주기때문에 0으로 한다.

## 학습 및 평가 함수 정의

```py
def train_and_test(epoch_count, mb_size, report):
    step_count = arrange_data(mb_size)
    test_x, test_y = get_test_data()
    
    for epoch in range(epoch_count):
        losses, accs = [], []
        
        for n in range(step_count):
            train_x, train_y = get_train_data(mb_size, n)
            loss, acc = run_train(train_x, train_y)
            losses.append(loss)
            accs.append(acc)
            
        if report > 0 and (epoch+1) % report == 0:
            acc = run_test(test_x, test_y)
            print('Epoch {}: loss={:5.3f}, accuracy={:5.3f}/{:5.3f}'. \
                  format(epoch+1, np.mean(losses), np.mean(accs), acc))
            
    final_acc = run_test(test_x, test_y)
    print('\nFinal Test: final accuracy = {:5.3f}'.format(final_acc))
```
먼저 epoch_count 만큼 학습을 반복하게 되어있다. 1번 학습시, mb_size만큼 쪼갠 데이터를 step_count(=학습용 데이터의 양 // mb_size)만큼 돌며 순서대로 학습하여 학습용 데이터를 모두 학습하도록 되어있다.

미니배치크기의 데이터 학습은 먼저 get_train_data로 학습용 데이터를 얻어와 run_train() 함수로 학습시키고, 손실과 정확도를 리턴받아 리스트에 넣는다. 만약 보고주기가 되면 run_test()로 테스트데이터를 이용하여 정확도를 측정하여 보여준다.

그리고 최종적으로 run_test()를 돌리고 최종 정확도를 출력한다.

## 학습 및 평가 데이터 획득 함수 정의

```py
def arrange_data(mb_size):
    global data, shuffle_map, test_begin_idx
    shuffle_map = np.arange(data.shape[0])
    np.random.shuffle(shuffle_map)
    step_count = int(data.shape[0] * 0.8) // mb_size
    test_begin_idx = step_count * mb_size
    return step_count

def get_test_data():
    global data, shuffle_map, test_begin_idx, output_cnt
    test_data = data[shuffle_map[test_begin_idx:]]
    return test_data[:, :-output_cnt], test_data[:, -output_cnt:]

def get_train_data(mb_size, nth):
    global data, shuffle_map, test_begin_idx, output_cnt
    if nth == 0:
        np.random.shuffle(shuffle_map[:test_begin_idx])
    train_data = data[shuffle_map[mb_size*nth:mb_size*(nth+1)]]
    return train_data[:, :-output_cnt], train_data[:, -output_cnt:]
```
arrange_data()함수는 train_and_test()함수에서 딱 한 번 호출한다. data.shape()함수는 data 행렬의 (행의 개수, 열의 개수)를 출력한다. 또한 arange함수는 np.arange(시작점(생략 시 0), 끝점(미포함), step size(생략 시 1)) 로 호출하며, 시작점부터 끝점(미포함) 까지 step size간격으로 수열을 생성한다. 따라서 data 개수 만큼 일련번호를 만들어 그것을 무작위로 섞는 함수이다.<br>
그리고 데이터를 학습용과 평가용으로 나누는데, 학습용 데이터를 80%로 잡는다. 또한 학습용 데이터 수를 미니배치 사이즈(mb_size)로 나누어 전체 데이터를 한번 학습하는데 미니배치 사이즈 만큼의 학습을 몇번 해야하는지 계산한다.<br>
또한 test_begin_idx에 테스트용 데이터의 시작점을 저장한다(step_count*mb_size는 학습용 데이터의 대략의 크기이다).

get_test_data()함수는 arrange_data()에서 섞어놓은 shuffle_map에서 test_begin_idx 이후의 값(data 행렬의 행의 번호)을 평가용 데이터로 반환한다. <br>
배열 인덱싱에서 A[-2]는 A 배열의 뒤에서 2번째 열을 의미하고, 행렬 인덱싱에서 B[1:3,4:5]는 B행렬에서 1행부터 2행 그리고 4열을 추출한다는 의미이며 C[:,:-2]는 C의 전체 행, 뒤에서 3번째열까지 추출한다는 의미이다.<br> 
따라서 test_data[:,:-output_cnt]는 test_data의 모든 데이터 행(:) 과 뒤에서 output_cnt열까지 추출한다는 의미, 여기서는 output_cnt=1이므로 마지막 열을 빼고 추출한다는 의미이다. 마지막 열은 정답 벡터이고 그 이전 열은 입력 벡터이므로, 'return ...'에서 알 수 있듯 (입력 벡터, 정답 벡터)를 반환한다.

get_train_data()는 get_test_data()와 비슷한데, 단지, 에포크 시작시(nth=0)shuffle_map의 학습용 데이터 부분(0~ test_begin_idx-1까지)를 한번 섞고 train_data를 mb_size만큼 쪼개어 nth번째 shuffle_map데이터를 입력 벡터와 정답 벡터로 분할해 리턴한다.

## 학습 실행 함수와 평가 실행 함수 정의
```py
def run_train(x, y):
    output, aux_nn = forward_neuralnet(x)
    loss, aux_pp = forward_postproc(output, y)
    accuracy = eval_accuracy(output, y)
    
    G_loss = 1.0
    G_output = backprop_postproc(G_loss, aux_pp)
    backprop_neuralnet(G_output, aux_nn)
    
    return loss, accuracy

def run_test(x, y):
    output, _ = forward_neuralnet(x)
    accuracy = eval_accuracy(output, y)
    return accuracy
```
먼저 run_train()함수는 주어진 입력행렬 x와 정답행렬 y를 이용하여 학습한다.

먼저 forward_neuralnet()함수가 순전파를 수행하여 입력행렬 x로 부터 출력 output을 구하고 forward_postproc()함수가 손실값을 계산 하고, eval_accuracy를 통해 정확도를 측정한다.

그리고 초기 loss값 1.0을 설정하고, backprop_postproc()을 호출하고 backprop_neuralnet를 실행하여 파라미터값을 변화시킨다.

run_test()함수는 입력 x에대한 output의 정확도만 측정하여 반환한다.

## 순전파 및 역전파 함수 정의

```py
def forward_neuralnet(x):
    global weight, bias
    output = np.matmul(x, weight) + bias
    return output, x

def backprop_neuralnet(G_output, x):
    global weight, bias
    g_output_w = x.transpose()
    
    G_w = np.matmul(g_output_w, G_output)
    G_b = np.sum(G_output, axis=0)

    weight -= LEARNING_RATE * G_w
    bias -= LEARNING_RATE * G_b
```
forward_neuralnet(x) 함수는 입력 x에 대해 가중치를 곱하고 bias를 더해 output을 계산하여 반환한다. x는 (N,10) 형태이고, weight는 (10,1)형태이므로 두 행렬을 곱하면 (N,1) 형태이다. bias는 (1)의 형태인데, 파이썬이 알아서 각 행에 bias를 더하게 된다.

backprop_neuralnet는 순전파 출력 output에 대한 손실 기울기 G_output을 받아서 weight와 bias의 손실 기울기를 구한다. weight와 bias에 손실기울기를 빼줌으로서 학습을 수행한다. [앞 글](https://minchocoin.github.io/alzza-deeplearning/1/)에서 $w_i$의 손실 기울기는 output의 손실 기울기에 $x_i$를 곱하면 된다고 하였다. 예를 들어 학습용 데이터 입력 세트가 총 5(a,b,c,d,e)개 들어왔다고 하자. 그리고 각 입력 세트는 3개의 입력으로 되어있다고 하자. 그러면 출력도 5개가 될 것이다. 각 입력 세트의 output의 손실 기울기를 $G_x$라 하자. 그러면 다음과 같이 가중치 행렬 W의 손실 기울기는 다음과 같다.

$$ \begin{pmatrix}Gw_0\\Gw_1\\Gw_2\\Gw_3\\ \end{pmatrix} = \begin{pmatrix}1&1&1&1&1\\a_1&b_1&c_1&d_1&e_1\\a_2&b_2&c_2&d_2&e_2\\a_3&b_3&c_3&d_3&e_3\\ \end{pmatrix}\begin{pmatrix}G_1\\G_2\\G_3\\G_4\\G_5\\ \end{pmatrix}$$

여기서 $Gw_x$는 $w_1$의 손실 기울기 의미한다. bias의 경우에는 $w_0x_0$에서 $x_0=1$로 고정이면 $w_0$가 bias 역할을 하므로 위 식에서 $a_0,b_0,c_0,d_0,e_0$가 다 1이면 된다. 따라서 bias의 손실 기울기는 $G_1 + G_2 + G_3 + G_4 + G_5$임을 알 수 있다.

## 후처리 과정에 대한 순전파 및 역전파 함수 정의

```py
def forward_postproc(output, y):
    diff = output - y
    square = np.square(diff)
    loss = np.mean(square)
    return loss, diff

def backprop_postproc(G_loss, diff):
    shape = diff.shape
    
    g_loss_square = np.ones(shape) / np.prod(shape)
    g_square_diff = 2 * diff
    g_diff_output = 1

    G_square = g_loss_square * G_loss
    G_diff = g_square_diff * G_square
    G_output = g_diff_output * G_diff
    
    return G_output
```
forward_postproc() 함수는 먼저 예측값 output 벡터와 실제값 y벡터를 빼서 각 예측값과 각 정답을 빼서 차이를 구하고, 오차를 제곱한 값을 square에 넣고 오차의 제곱 평균을 loss에 넣는다. output, y, diff, square는 (N,1)의 벡터이지만, loss는 스칼라 값이다.

backprop_postproc() 에서 diff는 (미니배치 크기(A), 출력벡터 크기{B}), 여기서는 (N,1) 크기를 갖는다. 여기서 $diff_{ij} = output_{ij} - y_{ij}$ 이고 $square_{ij} = diff_{ij}^2$ 이라 하고, 평균제곱오차이므로 손실기울기 $L = \frac{\sum square}{AB}$ , G_loss = $\frac{\partial L}{\partial L} = 1$ 에서 

$$\frac{\partial L}{\partial square_{ij}} = \frac{1}{AB} $$

이고

$$\frac{\partial square_{ij}}{\partial diff_{ij}} = 2diff_{ij} $$

이며

$$\frac{\partial diff_{ij}}{\partial output_{ij}} = 1 $$

이다.

따라서 

$$\frac{\partial L}{\partial output_{ij}} =\frac{\partial L}{\partial L}\frac{\partial L}{\partial square_{ij}}\frac{\partial square_{ij}}{\partial diff_{ij}}\frac{\partial diff_{ij}}{\partial output_{ij}}= \frac{2diff_{ij}}{AB} $$

이다.

backprop_postproc()은 위 식을 코드로 만든 것이다.

## 정확도 계산 함수 정의

```py
def eval_accuracy(output, y):
    mdiff = np.mean(np.abs((output - y)/y))
    return 1 - mdiff
```
mdiff는 정답 값에 비해 정답 값과 output의 차이가 얼마나 나는지의 평균이고, 이를 오류율이라고 하면 정확도는 1-오류율이라고 할 수 있다.

# 실행 결과

abalone.ipynb를 실행하는 abalone_test.ipynb에서 abalone_exec()을 실행하였을 때 표1과 같은 결과가 나왔다.

|                 |     모델1    |     모델2    |     모델3    |     모델4    |     모델5    |     모델6    |     모델7    |     모델8    |     모델9    |     모델10    |
|-----------------|--------------|--------------|--------------|--------------|--------------|--------------|--------------|--------------|--------------|---------------|
|     학습률      |     0.1      |     0.1      |     0.1      |     0.01     |     0.01     |     0.1      |     0.01     |     0.2      |     0.2      |     0.3       |
|     미니배치    |     100      |     100      |     10       |     100      |     100      |     100      |     200      |     200      |     100      |     200       |
|     에포크      |     100      |     1000     |     100      |     100      |     10000    |     20000    |     10000    |     10000    |     10000    |     10000     |
|     정확도      |     84.1%    |     84.7%    |     82.9%    |     81.6%    |     83.8%    |     84.5%    |     84.6%    |     83.8%    |     83.8%    |     85.4%     |

(표1 : 실험결과)

![alzza example](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/2418f773-97b1-409f-b26a-2d51522097c0)

(사진1 : 실험진행과정)

# 결과 분석

다음은 하이퍼파라미터 값에 따른 정확도의 변화이다.

## 에포크값에 의한 변화

(표1)의 모델1, 모델2, 모델6으로 보았을 때 에포크 값이 100에서 1000으로 늘었을 때는 정확도가 증가하였으나 1000에서 20000으로 늘렸을 때는 오히려 감소하는 모습을 보였다.

## 미니배치에 의한 변화

(표1)의 모델1과 모델3으로 보았을 때 미니배치 값을 100에서 10으로 줄였을 때 정확도가 84.1%에서 82.9%로 감소하였다. 그리고 <표1>의 모델5와 모델7로 보았을 때 미니배치값을 100에서 200으로 늘렸을 때 정확도가 증가하였다.

## 학습률에 의한 변화

(표1)의 모델1과 모델4로 보았을 때 학습률을 0.1에서 0.01로 줄였을 때 오히려 정확도가 감소하였다. (표1)의 모델7, 모델8, 모델10으로 보았을 때, 0.01에서 0.2로 늘렸을 때 정확도가 감소하였으나, 0.2에서 0.3으로 올렸을 때 정확도가 상승하였다. 그리고 학습률이 0.01, 0.1 일때보다 0.2,0.3 일때 학습 중에 정확도가 더 많이 움직이는 것을 알 수 있다.

# 결론

코드 분석을 통해 컴퓨터가 어떻게 데이터를 학습하는지 알 수 있었다.<br/>
(표1)의 10번의 실험을 통해 에포크, 미니배치사이즈, 학습률 모두 정확도에 영향을 주는 것을 알 수 있다. 영향을 선형적으로 주는 것이 아니라, 즉 클수록, 작을수록 좋은 것이 아니라 적당한 값이 있다. 학습률 0.3, 미니배치사이즈 200, 에포크 10000 일때 85.4%(모델 10)로 가장 정확도가 크게 나타났다. 이 값은 학습률 0.01, 미니배치사이즈 100, 에포크 100 일때 81.6%(모델4)와 3.8%차이이다.

# Reference

 - 윤덕호 저, [파이썬 날코딩으로 알고 짜는 딥러닝], 한빛미디어, 2019
 - https://jimmy-ai.tistory.com/45
