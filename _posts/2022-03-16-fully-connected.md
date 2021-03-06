---
layout: post
title: numpy로 Fully Connected layer 구성하기
subtitle: cs231n 과제 리뷰
date: 2022-03-16
last_modified_at: 2022-03-17
tags: [deep learning, cs231n, fully_connected, numpy]
---

Stanford의 cs231n 수업의 과제 FullyConnectedNets.ipynb를 하며 이해한 내용을 복기하고자 작성하는 포스트. 인터넷에 있는 내용을 참고했지만, 직접 고민하며 작성한 코드라 최적의 효율을 보장하지는 않지만 원하는 결과는 내었다.

## affine forward

딥러닝의 가장 기본적인 층 유형은 모든 유닛들이 서로 연결되어 있는 Fully Connected (FC) layer이다. 각 유닛들은 선형변환 이후 bias를 더하는 아핀 변환을 하는데, 이는 다음 수식과 같다. <!-- $$Y = WX + b$$ --> 

<div align="center"><img style="background: white;" src="..\svg\ykTcYmBp2F.svg"></div> 

```python
def affine_forward(x, w, b):
    """
    Computes the forward pass for an affine (fully-connected) layer.

    The input x has shape (N, d_1, ..., d_k) and contains a minibatch of N
    examples, where each example x[i] has shape (d_1, ..., d_k). We will
    reshape each input into a vector of dimension D = d_1 * ... * d_k, and
    then transform it to an output vector of dimension M.

    Inputs:
    - x: A numpy array containing input data, of shape (N, d_1, ..., d_k)
    - w: A numpy array of weights, of shape (D, M)
    - b: A numpy array of biases, of shape (M,)

    Returns a tuple of:
    - out: output, of shape (N, M)
    - cache: (x, w, b)
    """
    out = None
   
    x_flat = x.reshape(x.shape[0], -1) # of shape(N, D)
    out = np.matmul(x_flat, w) + b

    cache = (x, w, b)
    return out, cache
```
input data인 X의 shape이 (batch_size, d1, d2, ..., dk)로 batch마다 다차원의 feature를 받는다고 가정함으로 우선 (N, D)의 크기로 flatten 해야 한다.

## affine backward

훈련 과정 중 역전파를 위해 FC layer의 local derivative를 구하는 코드. 고등학교 때 기본(?) 미적분만 배웠지 삼각함수, 분수 등의 미적분은 해보지 못한 문과 출신이지만, 역전파의 원리만 이해하면 코드로 구현하기가 크게 어렵지는 않았다.

```python
def affine_backward(dout, cache):
    """
    Computes the backward pass for an affine layer.

    Inputs:
    - dout: Upstream derivative, of shape (N, M), i.e. 상위 layer들의 hidden unit마다의 cumulative derivative 
    - cache: Tuple of:
      - x: Input data, of shape (N, d_1, ... d_k)
      - w: Weights, of shape (D, M)
      - b: Biases, of shape (M,)

    Returns a tuple of:
    - dx: Gradient with respect to x, of shape (N, d1, ..., d_k)
    - dw: Gradient with respect to w, of shape (D, M)
    - db: Gradient with respect to b, of shape (M,)
    """
    x, w, b = cache

    dx = dout.dot(w.T) # dx = w. Thus, (N, M) @ (M, D) = (N, D)
    dx = dx.reshape(x.shape) #  (N, D) -> (N, d1, d2, .. d_k)
    dw = x.reshape(x.shape[0], -1).T.dot(dout) # (D, N) @ (N, M) -> (D, M) 
    db = dout.sum(axis=0) # b/c local db = 1 

    return dx, dw, db
```
역전파시 현재 FC layer의 히든 유닛의 loss 미분값이 dout라는 변수로 인풋으로 주어진다. 정확히 말해 이 dout는 affine 변환이 이뤄진 후, 즉 <!-- $ Y = WX+B$ --> <img style="transform: translateY(0.1em); background: white;" src="..\svg\ppmG7GCQxZ.svg">의 결과인 Y로 loss를 미분한 값(<!-- $ \frac{\partial loss}{\partial y}$ --> <img style="transform: translateY(0.1em); background: white;" src="..\svg\1dlzoS1PWO.svg">)이다. 미분의 chain rule에 의하여 <!-- $ \frac{\partial loss}{\partial variable} = \frac{\partial loss}{\partial y}\cdot\frac{\partial y}{\partial variable} $ --> <img style="transform: translateY(0.1em); background: white;" src="..\svg\IPHaPNI4Xm.svg">이고 <!-- $ \frac{\partial loss}{\partial y}$ --> <img style="transform: translateY(0.1em); background: white;" src="..\svg\zKhzIkoWsv.svg"> 가 인풋으로 주워져 있으므로 <!-- $ \frac{\partial y}{\partial variable} $ --> <img style="transform: translateY(0.1em); background: white;" src="..\svg\H6psNBdZ1v.svg"> 만 구하면 된다.
<!-- $$ y^i= w^iX + b^i, \\
\frac{\partial y}{\partial X}  = w^i; \frac{\partial y}{\partial w^i} = X; \frac{\partial y}{\partial b^i} = 1 $$ --> 

<div align="center"><img style="background: white;" src="..\svg\5JtFr0ZDle.svg"></div> 이므로 이 값들을 원래 X, W, b의 shape에 맞게끔 dout와 연산하면 된다. bias의 경우 각 hidden unit마다 하나의 bias를 갖으므로, hidden unit의 크기를 갖게끔 batch axis를 따라 sum을 해준다.  

## relu forward

딥러닝의 핵심 요소 중 하나가 비선형 함수로 활성화하는 것이다. 초기에는 활성화 함수로 sigmoid 함수를 주로 사용했지만, 치역이 항상 0과 1사이의 값이기 때문에 층이 깊어질수록 누적되는 미분값이 0에 수렴하여 하위 층의 히든 유닛이 사실상 업데이트 되지 않는 문제가 생긴다. 이 문제를 해결하고자 1이상의 결과값을 낼 수 있는 relu 활성화 함수가 널리 사용된다.
<!-- $$ Relu(x) = max(0, x) $$ --> 

<div align="center"><img style="background: white;" src="..\svg\Hu2d8C0RfE.svg"></div>

```python
def relu_forward(x):
    """
    Computes the forward pass for a layer of rectified linear units (ReLUs).

    Input:
    - x: Inputs, of any shape

    Returns a tuple of:
    - out: Output, of the same shape as x
    - cache: x
    """
    out = np.maximum(x, 0) # cf. np.maximum : elmentwise <-> np.max : column- or row-wise 
    cache = x
    return out, cache
```
numpy의 함수 중 maximum 함수는 element-wise로 작동하는 함수로 주어진 함수의 각 요소와 기준값을 비교한다. 흔히 사용하는 max 함수는 주어진 array내 최댓값을 구한다.

## relu backward
```python
def relu_backward(dout, cache):
    """
    Computes the backward pass for a layer of rectified linear units (ReLUs).

    Input:
    - dout: Upstream derivatives, of any shape
    - cache: Input x, of same shape as dout

    Returns:
    - dx: dldx; Gradient with respect to x
    """
    dx, x = None, cache

    dx = np.where(x > 0, 1 * dout, 0) # if-condition, true-yield, false-yield

    return dx
```
함수의 미분은 해당 함수의 기울기와 같은데, relu 함수의 경우 input x가 0보다 같거나 작을 때는 기울기가 0, 0 초과일 때는 기울기가 1이다. 따라서 0보다 큰 인풋에 한해서 dout에 1를 곱한 값을 편미분값으로 가진다.

## softmax loss
<!-- $$ softmax(z_i) = \frac{e^{z_i}}{\sum_{j=1}^{k}e^{z_j}} $$ --> 

<div align="center"><img style="background: white;" src="..\svg\dSsTQc8lqf.svg"></div>
softmax 함수의 장점은 결과값이 항상 0과 1사이의 값이며 모든 결과값을 더했을 때 1이기 때문에 변수의 확률값으로도 해석할 수 있다는 점이다.

```python
def softmax_loss(x, y):
    """
    Computes the loss and gradient for softmax classification.

    Inputs:
    - x: Input data, of shape (N, C) where x[i, j] is the score for the jth
      class for the ith input.
    - y: Vector of labels, of shape (N,) where y[i] is the label for x[i] and
      0 <= y[i] < C

    Returns a tuple of:
    - loss: Scalar giving the loss
    - dx: Gradient of the loss with respect to x
    """
    shifted_logits = x - np.max(x, axis=1, keepdims=True)
    Z = np.sum(np.exp(shifted_logits), axis=1, keepdims=True)
    log_probs = shifted_logits - np.log(Z)
    probs = np.exp(log_probs)
    N = x.shape[0]
    loss = -np.sum(log_probs[np.arange(N), y]) / N
    dx = probs.copy()
    dx[np.arange(N), y] -= 1
    dx /= N
    return loss, dx
```
위 코드는 대학측에서 기본적으로 제공하는 코드로 이를 내 식으로 이해해 보고자 노력해 봤다. 

#### logits
reference: 
1. [StatQuest](https://youtu.be/ARfXDSkQf1Y)
2. https://haje01.github.io/2019/11/19/logit.html

딥러닝 모델 코드들을 보다 보면 logits이라는 단어가 많이 나오는데, 주로 분류모델의 마지막 층에서 확률값으로 변환되기 전 raw 값들을 일컫는 데 사용된다. 원래 수학적으로 logit은 log+odds로 <!-- $$ logit = \ln \frac{p}{1-p} $$ --> 

<div align="center"><img style="background: white;" src="..\svg\J1rJLGYGXx.svg"></div> 즉 odds(실패 확률 대비 성공 확률)에다가 자연함수를 씌운 값이다. logit과 probability 는 서로 역함수의 관계로 <!-- $ [-\infty, \infty] $ --> <img style="transform: translateY(0.1em); background: white;" src="..\svg\lXFt7vhs8R.svg">인 logit을 [0,1]의 probability를 변환시켜주기 때문에 분류모델에서 자주 사용된다고 한다.
```python
# 위 코드에서 
    shifted_logits = x - np.max(x, axis=1, keepdims=True)
    # 분자; softmax는 어차피 상대적인 분포이기 때문에 최댓값으로 빼도 상관 없음.
    Z = np.sum(np.exp(shifted_logits), axis=1, keepdims=True)
    # 분모
    log_probs = shifted_logits - np.log(Z)
    # 소프트맥스 함수를 자연 log의 인풋으로 줌. 자연로그와 자연상수는 서로 서로 상쇄함으로 계산이 편함.
    probs = np.exp(log_probs)
    # log를 exp로 상쇄함으로써 확률값을 구함.
```

#### cross entropy loss
reference:
1. [StatQuest](https://youtu.be/YtebGVx-Fxw)
2. [Aurélien Géron](https://youtu.be/ErfnhcEV1O8)

분류모델에서 소프트맥스 활성함수와 자주 쓰이는 목적함수는 크로스 엔트로피(cross entropy) 함수이다. 이 목적함수는 각 클래스의 예측 확률값과 정답 레이블 값(이때 각 값들은 0 또는 1)을 곱한 값들을 더한 것이다.

![Cross-entropy](..\assets\img\cross-entropy.png)
```python
# 위 코드에서
    loss = -np.sum(log_probs[np.arange(N), y]) / N
```
여기서는 numpy array의 인덱싱 기능을 활용하여 각 배치마다 **정답** (*예측 레이블 아님*) 레이블 클래스의 로그 확률을 배치 사이즈만큼 스케일한 값을 loss 값으로 활용했다. 이 손실함수에 따르면, 모델이 정답 레이블을 1과 비슷한 확률에 가깝게 예측할 경우 손실은 log1, 즉 0으로 파라미터가 거의 업데이트 되지 않으나, 정답 레이블을 낮은 확률로 예측할 시 손실은 log0 즉 무한대로 수렴한다.   

### softmax 미분
reference:
1. [자세한 소프트맥스 미분 과정](https://ratsgo.github.io/deep%20learning/2017/10/02/softmax/)

간략히 말해 소프트맥스의 미분값은 실제 정답 클래스의 예측 확률값에서 1을 뺀 값이다. 위 loss function에서 배치 사이즈 만큼 스케일했으므로 미분값에서도 배치 사이즈 만큼 스케일해준다.
```python
# 위 코드에서
    dx = probs.copy()
    dx[np.arange(N), y] -= 1
    dx /= N
```

### L2 regularization

overfitting을 완화하기 위해 위에서 구한 최종 loss값에 추가 규제값을 더하기도 한다. 과제에서는 l2 규제를 더했다.
![L2_Reg](https://androidkt.com/wp-content/uploads/2021/09/l2_regula.png)
미분식을 간단하게 하기 위해 다음 코드처럼 스케일링을 <!-- $ \lambda$ --> <img style="transform: translateY(0.1em); background: white;" src="..\svg\hH9ahK03Lz.svg"> 가 아닌 <!-- $ \frac{\lambda}{2}$ --> <img style="transform: translateY(0.1em); background: white;" src="..\svg\AgX18t4vYj.svg">(또는 <!-- $ \frac{\lambda}{2N}$ --> <img style="transform: translateY(0.1em); background: white;" src="..\svg\KkyJIigQdM.svg">)로 하기도 한다.
```python
# excerpt from cs231n/classifiers/fc_net.py/FullyConnectedNet
    self.reg = labmda # 하이퍼파라미터로 주어지는 labmda값

    def loss(X, y):

        # 중략

        loss, dx = softmax_loss(scores, y)
        
        # add l2 regularization
        loss += 0.5 * self.reg * ( np.sum(self.params['W1']**2) + np.sum(self.params['W2']**2 ) )

        # FC_2 back prop
        dx, dw, db = affine_backward(dx, cache_fc2)
        grads['W2'], grads['b2'] = dw + self.reg * self.params['W2'], db # w/ derivative of the L2 reg
        
        # relu_1 back prop
        dx= relu_backward(dx, cache_relu)

        # FC_1 back prop
        dx, dw, db = affine_backward(dx, cache_fc1)
        grads['W1'], grads['b1'] = dw + self.reg * self.params['W1'], db
```
