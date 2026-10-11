---
title: 'Python + NumPy로 신경망 만들기 (6) — 지시서대로 가중치를 고치기, 학습 루프'
date: 2026-10-10 12:00:00 +0900
permalink: /posts/2026/10/nn-from-scratch-06-train/
categories:
  - AI
tags:
  - NumPy
  - MNIST
  - Deep Learning
  - From Scratch
  - Python
---

> 이 글은 `notebooks/06_train.ipynb`와 그걸 옮긴 `train.py`, `mnist/optim.py`에서 한 일을 처음부터 다시 설명합니다.
> [05](/posts/2026/10/nn-from-scratch-05-backward/)에서 만든 지시서(`grad_W`, `grad_b`)를 드디어 **실제로 씁니다.**

## 이 노트북에서 한 일, 한 줄로

**"지금까지 만든 다섯 조각을 반복문 하나에 넣고, 배치마다 `W -= lr × grad_W`로 가중치를 고쳤더니, 3바퀴 만에 테스트 사진 1만 장 중 95.43%를 맞췄다."**

01~05는 전부 **준비**였습니다. 데이터를 읽고, 점수를 내고, 채점하고, 어디를 고칠지 계산하는 것까지. 이 노트북에서 처음으로 `W`가 **바뀝니다.** 03에서 "지금은 낙서지만 학습이 악보로 바꾼다"고 했던 그 학습이 여기예요.

---

## 1. 다섯 조각이 한 반복문에 들어간다

```python
for xb, yb in iterate_batches(x_train, y_train, 64):   # 02  배치 하나 꺼냄
    loss = loss_fn.forward(model.forward(xb), yb)      # 03, 04  점수 → 손실
    model.backward(loss_fn.backward())                 # 05  지시서 계산
    optimizer.step()                                   # 06  지시서대로 고침  ← 새로 만든 것
```

새로 만든 건 마지막 줄 하나입니다. 나머지는 전부 앞 노트북에서 가져온 거예요.

```
         ┌───────────────────────────────────────────────┐
         ▼                                               │
사진 ──▶ 모델 ──▶ 점수 ──▶ 손실 ──▶ backward ──▶ 지시서 ──▶ W 수정 ─┘
                                                      (반복)
```

---

## 2. 학습 전 손실: 2.3025

반복문을 짜기 전에, 랜덤 모델의 손실이 **2.3026 근처**인지 먼저 확인했습니다.

```python
np.random.seed(0)
model = Sequential([Linear(784, 128), ReLU(), Linear(128, 10)])
loss_fn = SoftmaxCrossEntropy()

xb, yb = next(iterate_batches(x_train, y_train, 64))
print(loss_fn.forward(model.forward(xb), yb))    # 2.302507...
```

04에서 "10개에 전부 0.1을 주면 손실은 `log(10) = 2.3026`"이라고 기억해 둔 숫자입니다. 랜덤 가중치 모델은 10개 숫자에 거의 똑같은 확률을 주니 정확히 그 값이 나와요. **이 숫자가 안 나오면 어딘가 연결이 틀어진 것**이고, 나오면 02~04가 제대로 이어졌다는 뜻입니다. 학습을 시작하기 전에 보는 첫 번째 정상 신호예요.

---

## 3. 한 걸음: `W -= lr × grad_W`

반복문 전에 **배치 하나로 딱 한 번만** 고쳐 봤습니다.

```python
lr = 0.1

loss_before = loss_fn.forward(model.forward(xb), yb)
model.backward(loss_fn.backward())                  # grad_W, grad_b 저장

for layer in model.layers:
    if hasattr(layer, "W"):                         # 가중치 있는 레이어만
        layer.W -= lr * layer.grad_W
        layer.b -= lr * layer.grad_b

loss_after = loss_fn.forward(model.forward(xb), yb)
print(loss_before, "->", loss_after)                # 2.3025 -> 2.2982
```

**손실이 줄었습니다.** 0.004밖에 안 줄었지만, 처음으로 `W`가 바뀌어서 모델이 조금 덜 틀리게 된 순간입니다.

### 왜 `-=`인가

`grad_W[i, j]`는 "이 가중치를 올리면 손실이 이만큼 **커진다**"였습니다(05).

| `grad_W[i, j]` 부호 | 뜻 | 해야 할 일 |
|---|---|---|
| 양수 | 올리면 손실이 커짐 | **내려라** |
| 음수 | 올리면 손실이 작아짐 | **올려라** |

둘 다 "기울기의 **반대** 방향"이고, 그게 `-=` 하나로 표현됩니다. 산비탈에서 기울기가 가리키는 반대쪽(내리막)으로 한 걸음 내딛는 것이라 **경사 하강법(gradient descent)**이라고 부릅니다.

### `lr`은 걸음 크기

`lr`(학습률, learning rate)은 그 한 걸음을 **얼마나 크게** 내딛을지입니다. 0.1이면 "기울기의 10분의 1만큼". 너무 작으면 하염없이 느리고, 너무 크면 골짜기를 건너뛰어 반대편 비탈로 튀어 오릅니다. 04에서 "학습률이 조금만 커도 점수가 1000까지 가서 `exp`가 터진다"고 한 게 그 경우예요.

### `hasattr`

`hasattr(layer, "W")`는 "이 객체에 `W`라는 속성이 있나"를 묻는 Python 내장 함수입니다. `ReLU`에는 가중치가 없으니 건너뜁니다.

---

## 4. SGD라는 이름

위의 한 줄에는 이름이 있습니다. **SGD**, 세 단어이고 각각 이미 아는 것입니다.

| 단어 | 뜻 | 어디서 했나 |
|---|---|---|
| **G**radient (기울기) | `grad_W` | 05 |
| **D**escent (하강) | 기울기 반대로 내려감. `-=` | 06 |
| **S**tochastic (확률적) | 6만 장 전체가 아니라 **무작위 64장**으로 기울기를 구함 | 02의 미니배치, 셔플 |

6만 장 전체로 기울기를 구하면 한 바퀴에 딱 한 걸음입니다. 64장씩 뽑으면 한 바퀴에 938걸음이죠. 64장으로 구한 기울기는 전체와 **조금 다르지만**(그래서 "확률적"), 걸음 수가 압도적으로 많아서 훨씬 빨리 내려갑니다.

"기울기를 받아서 가중치를 **어떻게** 고칠지 정하는 규칙"을 **옵티마이저(optimizer)**라고 부르고, SGD는 그중 가장 단순한 규칙입니다. Adam처럼 가중치마다 학습률을 자동 조절하는 똑똑한 규칙도 있는데, 그건 확장 실험감으로 남겨 둡니다.

---

## 5. 반복문: 938걸음 × 3바퀴

한 걸음이 되니 반복문으로 감쌉니다. 바깥은 "전체 데이터를 몇 바퀴 돌지"(**epoch**), 안쪽은 "배치마다".

```python
for epoch in range(3):
    losses = []
    for xb, yb in iterate_batches(x_train, y_train, 64):
        loss = loss_fn.forward(model.forward(xb), yb)
        model.backward(loss_fn.backward())
        for layer in model.layers:
            if hasattr(layer, "W"):
                layer.W -= lr * layer.grad_W
                layer.b -= lr * layer.grad_b
        losses.append(loss)
    print(f"epoch {epoch}: loss = {np.mean(losses):.4f}")
```

```
epoch 0: loss = 0.5473
epoch 1: loss = 0.2454
epoch 2: loss = 0.1816
```

### 숫자 읽기

- **epoch 0의 평균이 2.30이 아닌 이유:** 938번 고치는 동안 손실이 계속 내려가서, 첫 배치의 2.30과 마지막 배치의 낮은 값이 **평균**된 겁니다. 938개 배치 손실을 다 찍으면 너무 많아서 epoch마다 평균만 찍었어요.
- **줄어드는 폭이 0.30 → 0.06으로 작아지는 것:** 처음에는 크게 틀려서 크게 고치고, 갈수록 고칠 게 줄어듭니다. 더 돌리면 더 내려가지만 점점 천천히요.

---

## 6. 손실 말고 "몇 장 맞췄나": 정확도

손실 0.18은 사람이 읽기 어려운 숫자입니다. 결과를 보고할 때는 **정확도**(accuracy), "1만 장 중 몇 장을 맞췄나"로 합니다.

```python
x_test = preprocess(load_idx("data/t10k-images-idx3-ubyte.gz"))
y_test = load_idx("data/t10k-labels-idx1-ubyte.gz")

def accuracy(model, x, y):
    z = model.forward(x)             # (10000, 10) 점수
    pred = z.argmax(axis=1)          # 사진마다 점수가 가장 높은 자리 → (10000,)
    return (pred == y).mean()        # 맞은 비율

print(accuracy(model, x_test, y_test))    # 0.9543
```

**95.43%.** 1만 장 중 9,543장.

### 왜 테스트 데이터로 재나

학습에 쓴 6만 장으로 재면 "외운 건지 이해한 건지" 알 수 없습니다. 01에서 공부용(train)과 시험용(t10k)을 나눈 이유가 여기서 쓰입니다. 모델은 `t10k` 사진을 학습 중에 **한 번도 본 적이 없습니다.**

### `argmax(axis=1)`

`argmax`는 `max`의 사촌으로, 가장 큰 **값** 대신 가장 큰 값의 **자리 번호**를 돌려줍니다. `[0.1, 0.7, 0.2]`면 `1`. 점수가 가장 높은 자리 번호가 곧 모델의 답(0~9)입니다. `axis=1`은 04에서 익힌 그 규칙, "사진마다 하나"니까 10이 지워져야 합니다.

softmax를 거칠 필요가 없는 건 softmax가 **순서를 바꾸지 않기** 때문입니다. 점수가 가장 큰 자리는 확률도 가장 큽니다.

### 손실 0.18인데 정확도 95%?

`-log(p) = 0.18`이면 정답 확률 `p ≈ 0.83`입니다. 그런데 정확도는 95%로 더 높죠. 정확도는 "정답 자리가 **1등**이기만 하면" 맞은 걸로 치기 때문입니다. 정답 확률이 0.4여도 나머지가 더 작으면 맞은 거예요. 손실은 "얼마나 확신했나"까지 보고, 정확도는 "1등이 맞았나"만 봅니다.

---

## 7. 정해진 값과 정한 값

`Linear(784, 128)`의 128은 어디서 왔을까요?

| 숫자 | 어디서 왔나 | 바꿀 수 있나 |
|---|---|---|
| 784 | 입력 크기. 28×28 | ❌ 데이터가 정함 |
| 10 | 출력 크기. 숫자 0~9 | ❌ 문제가 정함 |
| **128** | 중간 뉴런 수 | ✅ **설계자가 정함** |
| **0.1** | 학습률 | ✅ |
| **64** | 배치 크기 | ✅ |
| **3** | epoch 수 | ✅ |

학습으로 배우는 게 아니라 사람이 미리 정하는 값을 **하이퍼파라미터**라고 합니다. 128은 03의 "본보기 그림을 128장 두겠다"는 뜻이고, 32로 줄이면 표현력이 부족할 수 있고 512로 늘리면 느려지고 외워 버릴 위험이 커집니다. 2의 거듭제곱인 건 관례일 뿐이에요. 이 값들을 바꿔 가며 정확도를 비교하는 게 **확장 실험**입니다.

---

## 8. `.py`로 옮기기

### `mnist/optim.py` — `SGD` 클래스

3절의 코드를 "준비"와 "한 걸음"으로 나눕니다.

```python
class SGD:
    def __init__(self, layers, lr):
        self.layers = [layer for layer in layers if hasattr(layer, "W")]   # 준비: Linear만 골라 둠
        self.lr = lr

    def step(self):
        for layer in self.layers:                                         # 한 걸음
            layer.W -= self.lr * layer.grad_W
            layer.b -= self.lr * layer.grad_b
```

`hasattr`를 `__init__`에서 한 번만 하면, 938번 도는 `step()`에서는 `if`가 필요 없습니다. 쓰는 쪽에서는 네 줄이 `optimizer.step()` 한 줄이 되고, 나중에 `Adam`을 만들면 `SGD(...)`를 `Adam(...)`으로 바꿔 끼우기만 하면 됩니다. `Sequential`이 레이어 종류를 몰라도 되는 것과 같은 설계예요.

### `train.py` — 프로젝트 루트

노트북 셀을 순서대로 옮긴 것입니다. 전체가 이것입니다.

```python
import numpy as np
from mnist.data import load_idx, preprocess, iterate_batches
from mnist.layers import Linear, ReLU, Sequential
from mnist.losses import SoftmaxCrossEntropy
from mnist.optim import SGD


def accuracy(model, x, y):
    z = model.forward(x)
    pred = z.argmax(axis=1)
    return (pred == y).mean()


def main():
    x_train = preprocess(load_idx("data/train-images-idx3-ubyte.gz"))
    y_train = load_idx("data/train-labels-idx1-ubyte.gz")
    x_test = preprocess(load_idx("data/t10k-images-idx3-ubyte.gz"))
    y_test = load_idx("data/t10k-labels-idx1-ubyte.gz")

    np.random.seed(0)
    model = Sequential([Linear(784, 128), ReLU(), Linear(128, 10)])
    loss_fn = SoftmaxCrossEntropy()
    optimizer = SGD(model.layers, lr=0.1)

    for epoch in range(3):
        losses = []
        for xb, yb in iterate_batches(x_train, y_train, 64):
            loss = loss_fn.forward(model.forward(xb), yb)
            model.backward(loss_fn.backward())
            optimizer.step()
            losses.append(loss)
        print(f"epoch {epoch}: loss = {np.mean(losses):.4f}")

    print("test accuracy:", accuracy(model, x_test, y_test))


if __name__ == "__main__":
    main()
```

`python train.py`로 실행하면 노트북과 **똑같은 숫자**(0.5473, 0.2454, 0.1816, 0.9543)가 나옵니다. 같은 시드를 썼으니까요. 숫자가 같다는 게 "제대로 옮겼다"의 확인입니다.

`train.py`를 `mnist/` 안이 아니라 루트에 두는 이유: `mnist/`는 **부품**이고 `train.py`는 **부품을 조립해 실행하는 것**입니다. 부품이 조립 코드를 import하면 안 됩니다.

```python
if __name__ == "__main__":
    main()
```

이 두 줄은 "이 파일을 직접 실행했을 때만 `main()`을 돌려라"입니다. 다른 파일이 `from train import accuracy`로 함수만 가져다 써도 학습이 자동으로 시작되지 않아요.

---

## 9. 만들면서 겪은 버그: `lr`을 무시함

`SGD.step()`을 처음 썼을 때 이렇게 됐습니다.

```python
layer.W -= layer.grad_W          # self.lr 이 빠짐
```

`self.lr`을 저장만 하고 **안 쓴** 거예요. 그런데 **에러가 안 납니다.** 방향(`-=`)은 맞으니 손실도 줄어들어요. 다만 걸음 크기가 10배(학습률 1.0)라서, 큰 모델에서는 발산할 수 있고, 확장 실험에서 "학습률을 바꿔도 결과가 똑같네?" 하고 헤맸을 겁니다.

이걸 잡으려고 테스트를 하나 더 넣었습니다.

```python
def test_sgd_uses_lr():
    # lr=0 이면 W 가 한 칸도 변하면 안 되고, lr 을 2배로 하면 변화량도 정확히 2배여야 한다
```

"한 걸음 뒤 손실이 줄어든다"는 테스트만으로는 이 버그를 **못 잡습니다.** 실제로 버그 버전에서도 통과했어요. `lr=0.0`으로 `step()`했는데 `W`가 변했다면 `lr`을 안 쓴 것이고, 그 검사가 있어야 잡힙니다. "테스트가 통과한다"와 "테스트가 그 버그를 검증한다"는 다릅니다. 03, 04에 이어 세 번째로 같은 교훈이에요.

---

## 10. 이 노트북에서 배운 것

| 용어 | 뜻 |
|---|---|
| 학습 루프 | 배치 꺼내기 → forward → 손실 → backward → `step()` 반복 |
| 경사 하강법 | 기울기의 반대 방향으로 가중치를 조금 이동. `W -= lr × grad_W` |
| 학습률 `lr` | 한 걸음의 크기 |
| SGD | 미니배치로 구한 기울기로 경사 하강. 가장 단순한 옵티마이저 |
| 옵티마이저 | 기울기를 받아 가중치를 어떻게 고칠지 정하는 규칙 |
| epoch | 전체 데이터를 한 바퀴 |
| 2.3026 | 랜덤 모델의 첫 손실. `log(10)`. 정상 신호 |
| `argmax(axis=1)` | 사진마다 점수가 가장 높은 자리 번호 |
| 정확도 | 테스트 사진 중 1등 자리가 정답인 비율 |
| 하이퍼파라미터 | 학습이 아니라 사람이 정하는 값. 128, 0.1, 64, 3 |
| `hasattr` | 객체에 그 속성이 있는지 |
| `if __name__ == "__main__"` | 직접 실행했을 때만 돌리기 |

---

## 스스로 확인해 보기

1. 학습 전 손실이 2.30이 아니라 4.6이 나왔다면 무엇을 의심해야 할까요? (힌트: `-log(0.01)`)
2. `W += lr * grad_W`로 쓰면 손실은 어떻게 될까요?
3. `lr = 0.1`로 한 걸음 걸었을 때 손실이 2.3025 → 2.2982였습니다. `lr = 0.2`면 대략 얼마가 될까요? 항상 그럴까요?
4. epoch 0의 평균 손실 0.547은 첫 배치 손실 2.30과 마지막 배치 손실 사이의 값입니다. 마지막 배치 손실은 0.547보다 클까요, 작을까요?
5. 학습 데이터로 정확도를 재면 95.43%보다 높을까요, 낮을까요? 그 숫자를 보고하면 왜 안 되나요?
6. `SGD(model.layers, lr=0.1)`에서 `model.layers`에는 `ReLU`도 들어 있는데, `step()`에서 `ReLU.W`를 찾다 에러가 안 나는 이유는?
7. `lr`을 무시하는 버그는 왜 "한 걸음 뒤 손실이 줄어든다" 테스트로 못 잡나요?

<details markdown="1">
<summary>답</summary>

1. 모델이 10개 중 하나에 거의 확신하는데 틀린 상태. 보통 초기 가중치가 너무 크거나(`* 0.01`이 빠짐) 전처리(`/255`)가 빠져서 점수가 크게 나온 것.
2. 기울기 방향(오르막)으로 가니 손실이 **커진다.** `test_sgd_step_decreases_loss`가 잡는다.
3. 대략 2.294 (두 배인 0.0086 감소). 하지만 항상은 아니다. 걸음이 너무 크면 골짜기를 건너뛰어 오히려 커질 수 있다.
4. 작다. 손실이 계속 내려가니 마지막 배치가 가장 낮고, 평균은 그 위에 있다.
5. 높다 (학습 중에 본 사진이니). 그 숫자는 "외운 정도"를 포함해서 실제 성능을 과대평가하므로 보고하면 안 된다.
6. `__init__`에서 `hasattr(layer, "W")`로 Linear만 골라 `self.layers`에 담았기 때문. `step()`은 그 목록만 돈다.
7. 방향(`-=`)만 맞으면 걸음 크기가 얼마든 손실은 줄기 때문. `lr=0`일 때 `W`가 안 변하는지를 따로 확인해야 한다.

</details>

---

**이전 글:** [05. 손실에서 거꾸로 올라가며 "가중치를 어떻게 고칠지" 계산하기](/posts/2026/10/nn-from-scratch-05-backward/)

**다음:** 확장 실험 — 뉴런 수, 학습률, epoch, 옵티마이저(Adam), 초기화, 활성화 함수(Sigmoid), 손실 함수(MSE)를 바꿔 가며 정확도 표 만들기

**시리즈 소개:** [00. 시작하며 (전체 목차)](/posts/2026/10/nn-from-scratch-00-intro/)
