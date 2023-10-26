---
title: "다층 퍼셉트론(MLP) - 세 가지 신경망 재구성 코드 실행"
last_modified_at: 2023-10-11T19:12:12+09:00
categories:
    - alzza-deeplearning
tags:
    - A.I

toc: true
toc_label: "My Table of Contents"
author_profile: true

---
# 목적
전복 고리수 추정 신경망, 천체의 펄서 예측 신경망, 철판 분류 신경망을 다층 퍼셉트론을 이용하여 학습하고 결과를 비교한다.

# 실험 원리
[https://minchocoin.github.io/alzza-deeplearning/7-mlp-structure-of-mlp/](https://minchocoin.github.io/alzza-deeplearning/7-mlp-structure-of-mlp/)

## 하이퍼볼릭 탄젠트 함수
비선형 함수로 하이퍼볼릭 탄젠트 함수를 쓸 수 있다. 하이퍼볼릭 탄젠트 함수 tanh(x)는 다음과 같이 정의된다.

$$ tanh(x) = \frac{e^x - e^{-x}}{e^x + e^{-x}}$$

이것을 시그모이드 함수를 이용하여 다음과 같이 구할 수 있다.

$$ tanh(x) = 2\sigma (2x) - 1$$

또한 하이퍼 볼릭 탄젠트 함수의 미분은 아래와 같다.

$$ tanh'(x) = 1-tanh^2(x)$$

# 실험 방법

코드는 아래 코드를 사용하였다.

[https://github.com/KONANtechnology/Academy.ALZZA/blob/master/codes/chap04/mlp.ipynb](https://github.com/KONANtechnology/Academy.ALZZA/blob/master/codes/chap04/mlp.ipynb)

(1)	다층 퍼셉트론에서, 은닉 계층 1개의 폭은 4, 은닉 계층 2개의 폭은 각각 6과 4, 은닉 계층 3개의 폭은 각각 12,6,4로 정한다.

(2)	전복 고리수 추정 신경망을 단층 퍼셉트론, 은닉 계층 1개, 은닉 계층 2개, 은닉 계층 3개(또, 각각 학습률 0.001과 0.0001)로 학습하고, 결과를 확인한다.

(3)	천체의 펄서 예측 신경망을 단층 퍼셉트론, 은닉 계층 1개, 은닉 계층 2개, 은닉 계층 3개(또, 각각 학습률 0.001과 0.0001)로 학습하고, 결과를 확인한다.

(4)	철판 분류 신경망을 단층 퍼셉트론, 은닉 계층 1개, 은닉 계층 2개, 은닉 계층 3개(또, 각각 학습률 0.001과 0.0001)로 학습하고, 결과를 확인한다.

(5)	(1)~(4) 과정을 tanh 함수를 적용하여 똑같이 실험한다.

이때 tanh 함수는 아래와 같이 만든다.

```py
def tanh(x):
    return 2 * sigmoid(2*x) - 1

def tanh_derv(y):
    return (1.0 + y) * (1.0 - y)

def relu(x):
    return np.maximum(x, 0)

def relu_derv(y):
    return np.sign(y)

def sigmoid(x):
    return np.exp(-relu(-x)) / (1.0 + np.exp(-np.abs(x)))
        
def sigmoid_derv(y):
    return y * (1 - y)

def sigmoid_cross_entropy_with_logits(z, x):
    return relu(x) - x * z + np.log(1 + np.exp(-np.abs(x)))

def sigmoid_cross_entropy_with_logits_derv(z, x):
    return -z + sigmoid(x)
```

그리고 아래와 같이 forward_neuralnet_hidden1(), backprop_neuralnet_hidden1(), forward_neuralnet_hiddens(),backprop_nerualnet_hiddens()을 아래와 같이 변경한다(relu를 tanh, relu_derv를 tanh_derv로 바꾼다)


```py
def forward_neuralnet_hidden1(x):
    global pm_output, pm_hidden
    
    hidden = tanh(np.matmul(x, pm_hidden['w']) + pm_hidden['b'])
    output = np.matmul(hidden, pm_output['w']) + pm_output['b']
    
    return output, [x, hidden]

def backprop_neuralnet_hidden1(G_output, aux):
    global pm_output, pm_hidden
    
    x, hidden = aux

    g_output_w_out = hidden.transpose()                      
    G_w_out = np.matmul(g_output_w_out, G_output)            
    G_b_out = np.sum(G_output, axis=0)                       

    g_output_hidden = pm_output['w'].transpose()             
    G_hidden = np.matmul(G_output, g_output_hidden)          

    pm_output['w'] -= LEARNING_RATE * G_w_out                
    pm_output['b'] -= LEARNING_RATE * G_b_out                
    
    G_hidden = G_hidden * tanh_derv(hidden)
    
    g_hidden_w_hid = x.transpose()                           
    G_w_hid = np.matmul(g_hidden_w_hid, G_hidden)            
    G_b_hid = np.sum(G_hidden, axis=0)                       
    
    pm_hidden['w'] -= LEARNING_RATE * G_w_hid                
    pm_hidden['b'] -= LEARNING_RATE * G_b_hid      

def forward_neuralnet_hiddens(x):
    global pm_output, pm_hiddens
    
    hidden = x
    hiddens = [x]
    
    for pm_hidden in pm_hiddens:
        hidden = tanh(np.matmul(hidden, pm_hidden['w']) + pm_hidden['b'])
        hiddens.append(hidden)
        
    output = np.matmul(hidden, pm_output['w']) + pm_output['b']
    
    return output, hiddens

def backprop_neuralnet_hiddens(G_output, aux):
    global pm_output, pm_hiddens

    hiddens = aux
    
    g_output_w_out = hiddens[-1].transpose()
    G_w_out = np.matmul(g_output_w_out, G_output)
    G_b_out = np.sum(G_output, axis=0)

    g_output_hidden = pm_output['w'].transpose() 
    G_hidden = np.matmul(G_output, g_output_hidden)

    pm_output['w'] -= LEARNING_RATE * G_w_out
    pm_output['b'] -= LEARNING_RATE * G_b_out
    
    for n in reversed(range(len(pm_hiddens))):
        G_hidden = G_hidden * tanh_derv(hiddens[n+1])

        g_hidden_w_hid = hiddens[n].transpose()
        G_w_hid = np.matmul(g_hidden_w_hid, G_hidden)
        G_b_hid = np.sum(G_hidden, axis=0)
    
        g_hidden_hidden = pm_hiddens[n]['w'].transpose()
        G_hidden = np.matmul(G_hidden, g_hidden_hidden)

        pm_hiddens[n]['w'] -= LEARNING_RATE * G_w_hid
        pm_hiddens[n]['b'] -= LEARNING_RATE * G_b_hid
```

# 실험 결과

|     파라미터 및 활성화 함수    |               |                           |     정확도              |                         |                         |
|--------------------------------|---------------|---------------------------|-------------------------|-------------------------|-------------------------|
|     은닉계층   수              |     학습률    |     비선형 활성화 함수    |     전복 고리 신경망    |     펄서 여부 신경망    |     철판 분류 신경망    |
|     0                          |     0.001     |     ReLU                  |     0.813               |     0.975               |     0.223               |
|     0                          |     0.001     |     Hyperbolic tan        |     0.813               |     0.975               |     0.223               |
|     0                          |     0.0001    |     ReLU                  |     0.801               |     0.970               |     0.202               |
|     0                          |     0.0001    |     Hyperbolic tan        |     0.801               |     0.970               |     0.202               |
|     1                          |     0.001     |     ReLU                  |     0.846               |     0.846               |     0.353               |
|     1                          |     0.001     |     Hyperbolic tan        |     0.849               |     0.974               |     0.366               |
|     1                          |     0.0001    |     ReLU                  |     0.742               |     0.742               |     0.399               |
|     1                          |     0.0001    |     Hyperbolic tan        |     0.808               |     0.976               |     0.399               |
|     2                          |     0.001     |     ReLU                  |     0.840               |     0.840               |     0.345               |
|     2                          |     0.001     |     Hyperbolic tan        |     0.841               |     0.973               |     0.345               |
|     2                          |     0.0001    |     ReLU                  |     0.791               |     0.791               |     0.312               |
|     2                          |     0.0001    |     Hyperbolic tan        |     0.766               |     0.912               |     0.312               |
|     3                          |     0.001     |     ReLU                  |     0.733               |     0.733               |     0.366               |
|     3                          |     0.001     |     Hyperbolic tan        |     0.730               |     0.903               |     0.340               |
|     3                          |     0.0001    |     ReLU                  |     0.755               |     0.755               |     0.458               |
|     3                          |     0.0001    |     Hyperbolic tan        |     0.755               |     0.905               |     0.366               |

# 결과 분석
전복 고리 신경망에서는 은닉 계층이 1개와 2개일 때, 그리고 학습률이 0.001일 때 정확도가 약 0.84~0.85로 가장 높았다. 비선형 활성화 함수는 ReLU, 하이퍼볼릭 탄젠트 둘다 비슷한 결과를 나타냈으며, 하이퍼볼릭 탄젠트를 적용하였을 때 소폭 감소하거나 소폭 증가하는 모습을 보이기도 하였다.

펄서 여부 판정 신경망에서는, 은닉 계층 0개일 때, 그리고 은닉 계층이 1개나 2개이면서 하이퍼볼릭 탄젠트를 적용하였을 때 정확도가 약 0.97~0.98로 가장 높았다.

철판 분류 신경망에서는 은닉 계층 3개와 ReLU함수 적용 및 학습률 0.0001일 때 정확도가 약 0.46으로 가장 높았다.

각 신경망마다 은닉 계층 수, 학습률, 비선형 활성화 함수에 따라 정확도가 높아지기도 하고 낮아지기도 하여, 일정한 패턴을 찾기는 어렵다. 

# 결론

은닉 계층 수와 학습률, 그리고 비선형 활성화 함수에 따른 전복 고리수 추정 신경망, 천체의 펄서 여부 판정 신경망, 철판 분류 신경망의 정확도를 비교하였다. 전복 고리 신경망은 은닉 계층1개, 학습률 0.001, hyperbolic tan 비선형 함수를 적용하였을 때 정확도 0.849로 제일 정확도가 높았으며, 펄서 여부 판정 신경망에서는 은닉 계층 1개, 학습률 0.0001 hyperbolic tan 비선형 함수를 적용하였을 때 정확도 0.976으로 제일 높았다. 철판 분류 신경망은 은닉 계층 3개, 학습률 0.0001, ReLU 비선형 함수를 적용하였을 때 정확도 0.458로 가장 높았다. 따라서 무조건 은닉 계층이 많을수록 정확도가 올라가는 것은 아니며, ReLU와 hyperbolic tan도 모델에 따라 ReLU가 더 좋을 때도 있고, hyperbolic tan가 더 좋을 때도 있다. 따라서 모델의 복잡성(선형 연산만으로 좋은 결과를 얻을 수 있는 여부 등), 훈련 데이터의 개수 등 여러 요인을 고려하여 그리고 실험을 통해 은닉 계층과 비선형 활성화 함수를 정해야 할 것이다.

# 참고 문헌
- 윤덕호 저, [파이썬 날코딩으로 알고 짜는 딥러닝], 한빛미디어, 2019
- https://reniew.github.io/12/
