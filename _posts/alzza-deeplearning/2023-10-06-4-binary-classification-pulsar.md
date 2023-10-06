---
title: "이진 판단 - 천체의 펄서 여부 판정 신경망"
last_modified_at: 2023-10-06T14:53:12+09:00
categories:
    - alzza-deeplearning
tags:
    - A.I

toc: true
toc_label: "My Table of Contents"
author_profile: true

---
# 목적
[파이썬 날코딩으로 알고 짜는 딥러닝] 책에서 두번째로 나오는 천체의 펄서 여부 판정 신경망 파이썬 코드를 분석해보고자 한다.
# 데이터
kaggle의 펄서 예측 데이터셋을 활용한다.

# 신경망의 이진 판단 방법
[https://minchocoin.github.io/alzza-deeplearning/3-slp-binary-classification/](https://minchocoin.github.io/alzza-deeplearning/3-slp-binary-classification/)

# 코드 분석
## 코드 재활용
```py
%run ../chap01/abalone.ipynb
```
전에 만들었던 init_model, train_and_test, arrange_data, get_train_data,get_test_data,run_train,run_test, forward_neuralnet, backprop_neuralnet을 재활용한다.
## 메인 함수 정의
```py
def pulsar_exec(epoch_count=10, mb_size=10, report=1):
    load_pulsar_dataset()
    init_model()
    train_and_test(epoch_count, mb_size, report)
```
위 함수는 먼저 데이터셋을 읽는 함수를 호출하고, 모델의 파라미터를 초기화하는 함수를 호출하고, 마지막으로 매개변수로 받은 epoch_count,mb_size,report를 train_and_test함수에 전달하고 호출하여 실제로 훈련과 테스트를 한다.

## 데이터 적재 함수 정의
```py
def load_pulsar_dataset():
    with open('../../data/chap02/pulsar_stars.csv') as csvfile:
        csvreader = csv.reader(csvfile)
        next(csvreader, None)
        rows = []
        for row in csvreader:
            rows.append(row)
            
    global data, input_cnt, output_cnt
    input_cnt, output_cnt = 8, 1
    data = np.asarray(rows, dtype='float32')
```
csv 패키지의 기능을 활용해 펄서 예측 데이터 셋 파일을 메모리로 읽어들인다. 펄서 예측 데이터 셋은 입력 8개와 출력 1개이다. 따라서 input_cnt와 output_cnt을 8과 1로 설정한다. 그리고 읽어들인 내용을 numpy array로 변경한다.

## 후처리 과정에서 순전파와 역전파 함수의 재정의
```py
def forward_postproc(output, y):
    entropy = sigmoid_cross_entropy_with_logits(y, output)
    loss = np.mean(entropy)
    return loss, [y, output, entropy]

def backprop_postproc(G_loss, aux):
    y, output, entropy = aux
    
    g_loss_entropy = 1.0 / np.prod(entropy.shape)
    g_entropy_output = sigmoid_cross_entropy_with_logits_derv(y, output)    
    
    G_entropy = g_loss_entropy * G_loss
    G_output = g_entropy_output * G_entropy
    
    return G_output
```
순전파 처리를 하는 forward_postproc은 모델이 예측한 output과 실제값 y에 대해 시그모이드 교차 엔트로피를 구하고 이들의 평균으로 손실 값을 구한다.

역전파처리에서 y,output,entropy는 모두 (미니배치 크기(A), 출력벡터 크기(B))의 크기를 갖는다. 여기서 $entropy_{ij} = sigmoidCrossEntropyWithLogits(y,output)$ 이고 손실기울기 $L = \frac{\sum entropy}{AB}$, G_loss = $\frac{\partial L}{\partial L} = 1$ 에서

$$ \frac{\partial L}{\partial entropy_{ij}} = \frac{1}{AB}$$

이고

$$ \frac{\partial entropy_{ij}}{\partial output_{ij}} = sigmoidCrossEntropyWithLogitsDerv(y,output)$$

이다.

따라서

$$ \frac{\partial L}{\partial output_{ij}} = \frac{\partial L}{\partial L}\frac{\partial L}{\partial entropy_{ij}} \frac{\partial entropy_{ij}}{\partial output_{ij}} = \frac{sigmoidCrossEntropyWithLogitsDerv(y,output)}{AB}$$

이며

backprop_postproc()은 위 식을 코드로 만든 것이다.

## 정확도 계산 함수 재정의
```py
def eval_accuracy(output, y):
    estimate = np.greater(output, 0)
    answer = np.greater(y, 0.5)
    correct = np.equal(estimate, answer)
    
    return np.mean(correct)
```
이진 판단 문제에서 정확도는 신경망이 추정한 로짓값에 따른 판단과 정답으로 주어진 판단이 일지하는 비율이다. output 로짓값이 0보다 크면 참으로 보고, y는 0과 1밖에 없으므로 1인 것을 참으로 본다. 그리고 output이 참/거짓으로 본것과 실제 참/거짓인지에 따라 true(output과 실제 둘다 참 또는 거짓으로 본 것)/false(output과 실제 중 하나만 참으로 보고 나머지 하나는 거짓으로 본 것)로 나누고 true를 1, false를 0으로 계산해 평균을 구하면 정확도를 구할 수 있다.

## 시그모이드 관련 함수 정의
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

시그모이드 함수의 안전한 계산을 위해 relu함수를 사용하고 있다 relu함수는 음수일때는 0을 반환하고, 양수일 때는 그대로 반환한다. [이진 판단](https://minchocoin.github.io/alzza-deeplearning/3-slp-binary-classification/)에서 본 시그모이드 함수(sigmoid(x)), 시그모이드의 편미분(sigmoid_derv(x,y)), 시그모이드 교차 엔트로피 함수(sigmoid_cross_entropy_with_logits(z,x)), 시그모이드 교차 엔트로피 함수의 편미분(sigmoid_cross_entropy_with_logits_derv(z,x)) 함수를 코드로 옮겨 놓았다.

# pulsar_test.ipynb 실행
이제 실제로 위 코드를 실행하여 보면 아래와 같은 결과가 나온다.

![pulsar_test.ipynb 실행결과](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/7fdcac90-2452-4f29-88f9-ac1ca641dcda)

실제 답(참/거짓)과 신경망이 예측한 답(참/거짓)이 97.6%가 일치하므로, 학습이 제대로 된 것으로 보인다.

# 확장하기 - 균형 잡힌 데이터셋과 착시 없는 평가 방법
펄서 예측 데이터셋의 데이터에서 9.2%만 펄서이다. 따라서 무조건 펄서가 아니라고 답하면 90.8%의 높은 정확도를 얻을 수 있다.

따라서 데이터셋에서 별과 펄서가 비슷한 수가 되도록 데이터 비율을 바꿔보자. 펄서 데이터를 중복 사용하여 일반 별 데이터와 같은 수로 만들면 된다.

데이터의 균형을 잡으면 성능이 향상되지만, 겉보기 정확도는 하락할 가능성이 있다. 따라서 성능을 더 잘보여주는 평가지표가 필요하다.

이때 정밀도와 재현율이 쓰인다. 정밀도는 신경망이 참으로 추정한 것 가운데 정답이 참인 것의 비율, 재현율은 정답이 참인 것들 가운데 신경망이 참으로 추정한 것의 비율이다.

만약 참이라는 대답을 남발하면 재현율이 낮아지고, 거짓이라는 답을 함부로 남발하면 정밀도가 낮아진다.

정밀도와 재현율 이 두 값을 함께 높혀야 의미가 있으며, 이 두 값을 함께 평가하기 위해 정밀도와 재현율의 조화평균인 F1 값을 사용한다. F1값은 두 값 사이의 값만 가질 수 있고, 작은 값에 가까워지는 성질이 있다.

이때 까지 정의한 것을 수식으로 표현해보자. 정답이 참인 것을 T, 거짓인 것을 F라하고, 추정이 참인 것을 P, 추정이 거짓인 것을 N이라 할때(여기서 참/거짓은 예를 들어 펄서인 것/ 펄서가 아닌 것을 의미한다) 정답과 추정에 따라 TP,TN,FP,FN의 네 집단으로 나눌 수 있다. 따라서 평가지표는 다음과 같이 정의된다.

$$ 정확도 = \frac{TP+FN}{TP+TN+FP+FN}$$

$$ 정밀도 = \frac{TP}{TP+FP}$$

$$ 재현율 = \frac{TP}{TP+TN}$$

$$ F1 = \frac{2\times 정밀도 \times 재현율}{정밀도 + 재현율} = \frac{2\times TP}{2\times TP+FP+TN}$$

# 확장된 펄서 신경망 코드 분석

## pulsar.ipynb 재활용
```py
%run ../chap02/pulsar.ipynb
```
pulsar.ipynb 코드를 대부분 재활용한다.

## 메인 실행 함수의 재정의
```py
def pulsar_exec(epoch_count=10, mb_size=10, report=1, adjust_ratio=False):
    load_pulsar_dataset(adjust_ratio)
    init_model()
    train_and_test(epoch_count, mb_size, report)
```
앞서 정의한 pulsar_main()에 비해서 adjust_ratio 매개변수값만 새로 추가되었다.adjust ratio가 True 시 dataset를 로드할 때 펄서와 비펄서 데이터 간의 균형을 맞춘다.

## 데이터 적재 함수의 재정의
```py
def load_pulsar_dataset(adjust_ratio):
    pulsars, stars = [], []
    with open('../../data/chap02/pulsar_stars.csv') as csvfile:
        csvreader = csv.reader(csvfile)
        next(csvreader, None)
        rows = []
        for row in csvreader:
            if row[8] == '1': pulsars.append(row)
            else: stars.append(row)
            
    global data, input_cnt, output_cnt
    input_cnt, output_cnt = 8, 1
    
    star_cnt, pulsar_cnt = len(stars), len(pulsars)

    if adjust_ratio:
        data = np.zeros([2*star_cnt, 9])
        data[0:star_cnt, :] = np.asarray(stars, dtype='float32')
        for n in range(star_cnt):
            data[star_cnt+n] = np.asarray(pulsars[n % pulsar_cnt], dtype='float32')
    else:
        data = np.zeros([star_cnt+pulsar_cnt, 9])
        data[0:star_cnt, :] = np.asarray(stars, dtype='float32')
        data[star_cnt:, :] = np.asarray(pulsars, dtype='float32')
```
먼저 파일을 읽을 때 펄서 데이터와 비펄서 데이터를 각각 pulsars와 stars 배열에 따로 담는다.

만약 adjust_ratio 값이 True인 경우 별 데이터는 그대로 data에 담고, pulsar 데이터는 별 데이터 만큼 복사하여 data에 넣어 별과 펄서의 데이터의 균형을 맞추고 있다.

adjust_ratio 값이 False인 경우 data에 펄서와 별 데이터를 복사하지않고 그냥 있는 그대로 넣는다.

## 정확도 계산 함수 재정의
```py
def eval_accuracy(output, y):
    est_yes = np.greater(output, 0)
    ans_yes = np.greater(y, 0.5)
    est_no = np.logical_not(est_yes)
    ans_no = np.logical_not(ans_yes)
    
    tp = np.sum(np.logical_and(est_yes, ans_yes))
    fp = np.sum(np.logical_and(est_yes, ans_no))
    fn = np.sum(np.logical_and(est_no, ans_yes))
    tn = np.sum(np.logical_and(est_no, ans_no))
    
    accuracy = safe_div(tp+tn, tp+tn+fp+fn)
    precision = safe_div(tp, tp+fp)
    recall = safe_div(tp, tp+fn)
    f1 = 2 * safe_div(recall*precision, recall+precision)
    
    return [accuracy, precision, recall, f1]

def safe_div(p, q):
    p, q = float(p), float(q)
    if np.abs(q) < 1.0e-20: return np.sign(p)
    return p / q
```
정확도뿐만 아니라 정밀도, 재현율, F1 값을 추가로 계산하고 있다. 추정이 참인 것은 로짓값이 0보다 큰 것이라 하고, 실제값이 참인 것은 0.5보다 큰 것(즉 1)으로 간주한다.

예를 들어 output이 (-1,1,2)일 경우 est_yes는 (False,True,True)가 되고, 이는 추정이 참인 것이 2개인 것이다. 또한 y가 (0,0,1)인 경우 ans_yes는 (False,False,True)가 되고, 이는 실제 답이 참인 것이 1개인 것을 의미한다. est_no와 ans_no는 각각 est_yes와 ans_yes를 논리적 not 연산을 한 것으로, 실제 답이 거짓인 것을 True 나타낸 것이다.

따라서 예를 들어 tp는 정의에 따라 추정도 참이고 실제도 참이므로, np.logical_and(est_yes,ans_yes)하면 est_yes와 ans_yes 둘다 참인 것만 True 라고 적혀있을 것이고, True는 1로 간주하므로, 값을 더하면 추정도 참이고 실제도 참인 것의 개수가 구해진다.

또한 q의 값이 너무 작은 경우, 나눌 때 무한대가 될 수 있으므로, 값이 너무 작은 경우 -1이나 1로 치환한다.

## 실행함수 재정의

```py
def train_and_test(epoch_count, mb_size, report):
    step_count = arrange_data(mb_size)
    test_x, test_y = get_test_data()
    
    for epoch in range(epoch_count):
        losses = []
        
        for n in range(step_count):
            train_x, train_y = get_train_data(mb_size, n)
            loss, _ = run_train(train_x, train_y)
            losses.append(loss)
            
        if report > 0 and (epoch+1) % report == 0:
            acc = run_test(test_x, test_y)
            acc_str = ','.join(['%5.3f']*4) % tuple(acc)
            print('Epoch {}: loss={:5.3f}, result={}'. \
                  format(epoch+1, np.mean(losses), acc_str))
            
    acc = run_test(test_x, test_y)
    acc_str = ','.join(['%5.3f']*4) % tuple(acc)
    print('\nFinal Test: final result = {}'.format(acc_str))
```
run_test시 반환되는 eval_accuracy 결과가 (정확도, 정밀도, 재현율, F1)의 배열이므로 acc_str에 (정확도, 정밀도, 재현율, F1)값을 a.bcd의 형태로 각각 넣어 출력한다. '%5.3f'의 의미는 소수점이하 3째자리까지출력 총 자릿수(점 포함) 5자리를 의미한다.

# 확장된 펄서 판정 신경망 실행
## 원래의 데이터셋으로 신경망 학습시키기
![원래의 데이터셋으로 신경망 학습시키기](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/0cec6531-4301-4d30-99de-c48cd6c49774)

result = (정확도, 정밀도, 재현율, F1)값에 해당한다. 별 데이터가 많아 별로 판정하는 쪽으로 몰리면서 정밀도는 높지만 재현율이 낮다. 즉 신경망이 펄서로 추정한 것중에는 펄서가 많지만, 실제 펄서를 펄서라고 예측한 비율은 낮다. 별 데이터가 많으므로 별을 별로 답하여 정확도는 높지만, 펄서도 별로 예측하는 비중이 높아 재현율이 낮은 것이다. 따라서 F1 값도 낮다.

## 균형 잡힌 데이터로 신경망 학습시키기
![균형 잡힌 데이터로 신경망 학습시키기](https://github.com/minchoCoin/minchoCoin.github.io/assets/62372650/3592e07f-6cdb-4aef-b01d-13fd39b8e07f)

정확도는 낮아졌지만 재현율이 크게 좋아졌다. 즉 실제 펄서를 펄서라고 예측한 비중이 높은 것으로, 전체적인 성능은 크게 좋아진 것이다. 다만 중복 데이터가 있으므로, 같은 데이터가 학습 데이터와 테스트 데이터에 모두 이용될 수 있으므로, 이는 과적합을 초래하고 따라서 새로운 데이터에 대해서는 예측을 잘 못할 수도 있다.
# Reference

 - 윤덕호 저, [파이썬 날코딩으로 알고 짜는 딥러닝], 한빛미디어, 2019