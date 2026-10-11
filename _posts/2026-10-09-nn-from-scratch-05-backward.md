---
title: 'Python + NumPy로 신경망 만들기 (5) — 손실에서 거꾸로 올라가며 "가중치를 어떻게 고칠지" 계산하기'
date: 2026-10-09 12:00:00 +0900
permalink: /posts/2026/10/nn-from-scratch-05-backward/
categories:
  - AI
tags:
  - NumPy
  - MNIST
  - Deep Learning
  - From Scratch
  - Python
---

> 이 글은 `notebooks/05_backward.ipynb`에서 한 일을 처음부터 다시 설명합니다.
> [04](/posts/2026/10/nn-from-scratch-04-loss/)에서 만든 손실(숫자 하나)에서 출발해, [03](/posts/2026/10/nn-from-scratch-03-model/)에서 만든 레이어들을 **거꾸로** 통과합니다.

## 이 노트북에서 한 일, 한 줄로

**"모든 레이어에 `backward`를 붙여서, 손실이 어느 가중치 때문에 얼마나 커졌는지를 역으로 계산하는 장치(역전파)를 만들고, 공식이 맞는지 수치로 검증했다."**

이 노트북은 한 번에 끝나지 않았습니다. `Linear.backward`에서 막혀서 01~04를 다시 복습하고 돌아왔고, 두 번째 시도에서 **숫자 하나짜리 레이어부터** 차례로 올라가서 완성했습니다. 그 순서대로 씁니다.

---

## 1. backward는 무엇을 만들기 위한 연산인가

먼저 목적부터. backward는 **"가중치를 어느 방향으로 얼마나 고쳐야 손실이 줄어드나"가 적힌 지시서**를 만듭니다. 그 지시서가 `grad_W`, `grad_b`이고, 가중치와 **같은 모양**입니다. 칸마다 "이 가중치를 올리면 손실이 커진다(+) / 작아진다(−)"가 적혀요.

```
사진 ──▶ Linear ──▶ ReLU ──▶ Linear ──▶ 점수 ──▶ SoftmaxCE ──▶ 손실
         (W1)                (W2)                                │
◀──────────────────── backward ──────────────────────────────────┘
   각 Linear 안에 grad_W, grad_b 가 저장됨 (산출물)
```

backward가 끝나도 **`W`는 아직 안 바뀝니다.** "이렇게 고치면 된다"까지만 하고, 실제로 고치는 건 다음 노트북(06)의 한 줄 `W = W - 학습률 × grad_W`입니다.

### 산출물과 배관

| 역할 | 누가 | 무엇 |
|---|---|---|
| **산출물** | 가중치가 있는 레이어(`Linear`) | `self.grad_W`, `self.grad_b`에 **저장** |
| **배관** | 모든 레이어 | `grad_x`를 **반환**해서 앞 레이어의 입력이 되게 함 |
| 출발점 | 손실 함수 | 인자 없이 첫 `grad`를 만듦 |

`Linear.backward`가 **돌려주는** `grad_x`는 산출물이 아닙니다. 앞 레이어가 자기 지시서를 만들려면 "내 출력이 손실에 얼마나 영향을 줬나"가 필요한데, 그걸 넘겨주는 통로예요. `ReLU`는 가중치가 없으니 배관만 있고, 맨 앞 `Linear`가 돌려주는 `grad_x`는 받을 레이어가 없어서 버려집니다.

---

## 2. 기호 ↔ 직접 만든 코드

처음에 막힌 이유가 "`x`, `w`, `out`, `g`, 손실이 뭐가 뭔지"였습니다. 전부 이미 만든 변수입니다.

| 기호 | 직접 만든 코드에서는 | shape (배치 64) |
|---|---|---|
| `x` | `iterate_batches`가 주는 `x_batch` | `(64, 784)` |
| `W`, `b` | `Linear` 안의 `self.W`, `self.b` | `(784, 10)`, `(10,)` |
| `out` | `layer.forward(x)`가 돌려주는 값 | `(64, 10)` |
| 손실 | `cross_entropy(...)`가 돌려주는 숫자 하나. 처음엔 2.3026 | 숫자 |
| `grad_out` (`g`) | **뒤 레이어의 `backward`가 돌려준 값.** "네 출력이 1 변하면 손실이 이만큼 변해" | `out`과 같음 |

새로운 건 `grad_out` 하나뿐입니다.

### 모든 레이어의 약속이 둘이 된다

03에서 `Sequential`이 레이어 종류를 몰라도 되는 이유가 "모두 `forward`라는 약속을 지킨다"였죠. 이제 약속이 하나 늘어납니다.

```python
class 모든레이어:
    def forward(self, x):
        self.x = x              # backward에 필요한 것을 메모
        return ...

    def backward(self, grad_out):
        return grad_in          # 뒤에서 받은 기울기 → 내 입력에 대한 기울기
```

---

## 3. "기울기"는 조금 밀면 얼마나 변하나

```python
def numerical_grad(f, x, h=1e-5):
    return (f(x + h) - f(x - h)) / (2 * h)
```

기울기의 정의 그대로입니다. 입력을 `h`만큼 양쪽으로 살짝 밀어 보고, 출력이 얼마나 변했는지 비율을 잽니다. **공식을 몰라도 어떤 함수든** 기울기를 구할 수 있어요. 대신 느립니다. 가중치 10만 개면 10만 번 forward를 해야 하니 학습에는 못 쓰고, **공식으로 짠 `backward`가 맞는지 검증하는 용도**로만 씁니다.

이 노트북의 모든 단계는 "공식을 짜고 → 이걸로 검사하고 → 안 맞으면 공식을 고친다"로 진행했습니다. 실제로 버그 세 개를 이 검사가 잡았습니다.

---

## 4. 1단계: `ReLU.backward` — 마스크

ReLU의 forward는 "양수는 통과, 음수는 0". backward도 **같은 자리에서 같은 일**을 합니다.

```python
class ReLU:
    def forward(self, x):
        self.x = x                       # 어디가 양수였는지 알려면 x가 필요
        return np.maximum(0, x)

    def backward(self, grad_out):
        return grad_out * (self.x > 0)   # 양수였던 자리만 통과
```

```
x        = [-2,    -0.5,  0,     0.5,  2  ]
x > 0    = [False, False, False, True, True]     ← 마스크
grad_out = [1,     1,     1,     1,    1  ]
결과      = [0,     0,     0,     1,    1  ]
```

### 왜 음수 자리는 0인가

음수 자리는 forward에서 출력이 **0으로 고정**됐습니다. 입력을 조금 바꿔도 출력은 0 그대로이니 손실도 안 변하고, 기울기는 0입니다. 양수 자리는 출력 = 입력이니 기울기가 그대로 전달됩니다. `x = 0`은 관례상 0(막음)으로 둡니다.

### 왜 `self.x`를 메모하나

`backward`는 "어느 자리가 양수였나"를 알아야 하는데, 그 정보는 forward에 들어온 `x`에 있고, `x`는 함수가 끝나면 사라집니다. 그래서 `self`에 적어 둡니다.

> **직접 발견한 것:** ReLU는 출력만 봐도 양수 자리를 알 수 있습니다(`out > 0`과 `x > 0`이 항상 같음). 그래서 `self.out`을 저장해도 됩니다. PyTorch는 메모리를 아끼려고 그렇게 합니다. 반면 `Linear`는 `out = x @ W + b`에서 `x`를 되돌릴 수 없어서 **반드시 입력을 저장**해야 합니다. 규칙은 "backward에 필요한 것을 메모한다"이고, 그게 무엇인지는 레이어마다 다릅니다.

### 테스트에서 `np.ones(5)`는 뭔가

`grad_out`은 원래 뒤 레이어가 넘겨주는 값인데, 레이어 하나만 떼어 테스트할 때는 뒤가 없습니다. 그래서 **가짜로** 넣습니다. 1로 채우면 "손실 = 출력의 합"이라고 가정한 것이고, `grad_out × 마스크`에서 결과가 **마스크 그 자체**로 나와 보기 편합니다.

### 수치 검사

```
-0.5 → 0.0      0.5 → 1.0      2.0 → 1.0
```

공식(`backward`)과 일치합니다. `0.99999999`처럼 미세하게 벗어난 건 float 반올림 오차라 `np.allclose`로 비교합니다.

---

## 5. 2단계: `SoftmaxCrossEntropy.backward` — `p - onehot`

softmax와 cross-entropy를 **따로** 미분하면 식이 복잡합니다(softmax는 10×10 미분표가 나옵니다). 그런데 **한 덩어리로** 미분하면 거의 다 약분되고 이것만 남습니다.

```
손실을 점수 z로 미분한 값 = p - onehot(y)     (그리고 평균이니 N으로 나눔)
```

```python
class SoftmaxCrossEntropy:
    def forward(self, z, y):
        self.p = softmax(z)              # 메모
        self.y = y                       # 메모
        return cross_entropy(self.p, y)

    def backward(self):
        N = len(self.y)
        onehot = np.zeros_like(self.p)           # p와 같은 모양의 0
        onehot[np.arange(N), self.y] = 1         # 정답 자리에만 1 (팬시 인덱싱으로 대입)
        return (self.p - onehot) / N
```

### onehot은 어떻게 만들어지나

04의 `p[np.arange(N), y]`가 값을 **꺼내는** 팬시 인덱싱이었다면, 여기서는 `= 1`을 붙여 값을 **넣습니다.** `y = [0, 2]`면 좌표 `(0,0)`, `(1,2)`에 1이 들어갑니다.

```
onehot = [[1, 0, 0],      ← 사진 0의 정답은 0번
          [0, 0, 1]]      ← 사진 1의 정답은 2번
```

정답 레이블을 "확률처럼 생긴" 형태(정답 100%, 나머지 0%)로 바꾼 것이고, 그래야 `p`와 같은 모양이 되어 뺄 수 있습니다.

### 손 계산

```
z = [[2, 1, -1], [0, 5, 0]],  y = [0, 2]

p          = [[0.705, 0.259, 0.035],
              [0.007, 0.987, 0.007]]
p - onehot = [[-0.295, 0.259, 0.035],
              [ 0.007, 0.987, -0.993]]
÷ 2        = [[-0.147, 0.130, 0.018],
              [ 0.004, 0.494, -0.497]]
```

### 부호의 뜻

| 자리 | 값 | 부호 | 뜻 |
|---|---|---|---|
| 정답 | `p - 1` | 항상 음수 | 이 점수를 올리면 손실이 **준다** → **올려라** |
| 오답 | `p - 0` | 항상 양수 | 이 점수를 올리면 손실이 **커진다** → **내려라** |

크기도 보세요. 사진 1은 정답(2번)에 0.007밖에 안 줬으니 `-0.497`로 **세게 올려라**, 1번 자리에 0.987이나 줬으니 `+0.494`로 **세게 내려라**. **틀린 만큼 비례**해서 신호가 나옵니다. 04에서 "`-log`는 틀릴수록 기울기가 또렷하다"고 한 게 이 숫자들입니다.

### 왜 `backward()`에 `grad_out`이 없나

이 레이어는 **사슬의 맨 끝**입니다. 뒤에 레이어가 없고, 출력이 손실 그 자체입니다. "손실이 1 변하면 손실이 얼마나 변하나"는 당연히 1이니 받을 필요가 없습니다. 역전파는 **항상 손실 함수에서 시작**합니다.

### 배열용 수치 검사

```python
def numerical_grad_array(f, x, h=1e-5):
    grad = np.zeros_like(x)
    for idx in np.ndindex(x.shape):        # 모든 좌표를 순회
        old = x[idx]
        x[idx] = old + h;  f_plus = f()
        x[idx] = old - h;  f_minus = f()
        x[idx] = old                       # 복구
        grad[idx] = (f_plus - f_minus) / (2 * h)
    return grad
```

3절의 함수를 **원소 하나씩** 적용하는 반복문입니다. 결과는 공식과 소수점 8자리까지 일치했습니다(`True`).

---

## 6. 3단계: `Linear.backward` — 네 번에 나눠 올라가기

여기서 처음에 막혔습니다. 행렬 식 `x.T @ grad_out`을 바로 보여 주니 "`.T`가 왜 생기는지"가 안 보였어요. 두 번째 시도에서는 **숫자 하나**부터 시작했습니다.

### 6-a. 숫자 하나 (`TinyLinear`)

```
out = x * w + b     (전부 숫자 하나)
```

| 대상 | 1 올리면 `out`은 | 그래서 기울기 |
|---|---|---|
| `w` | `x`만큼 변함 (`w`에 `x`가 곱해져 있으니) | `x` |
| `b` | 1만큼 변함 (그냥 더해져 있으니) | `1` |
| `x` | `w`만큼 변함 (`x`에 `w`가 곱해져 있으니) | `w` |

여기에 뒤에서 온 `grad_out`을 곱하면 **손실**에 대한 기울기가 됩니다. "`w`를 1 올리면 `out`이 `x` 변하고, `out`이 1 변하면 손실이 `grad_out` 변하니, `w`를 1 올리면 손실은 `x × grad_out` 변한다." 이 **곱하기**가 연쇄 법칙(Chain Rule)입니다.

```python
def backward(self, grad_out):
    self.grad_w = self.x * grad_out     # w에 곱해져 있던 것
    self.grad_b = 1 * grad_out          # 그냥 더해졌으니 1
    return self.w * grad_out            # x에 곱해져 있던 것
```

`w=2, b=0.5, x=3, grad_out=1`로 확인: `3.0 1.0 2.0`, 수치 검사와 일치.

**버그 1 (수치 검사가 잡음):** 처음에 `return x * grad_out`이라고 써서 `grad_x`가 3으로 나왔습니다. 수치 검사는 2라고 했죠. `x`의 기울기는 `x`가 아니라 **`x`에 곱해져 있던 `w`**입니다. 게다가 그 `x`는 `self.x`가 아니라 셀의 전역 변수를 가리키고 있었습니다. 메서드 안에서는 반드시 `self.`로 꺼내야 합니다.

### 6-b. 입력 여러 개 (`VecLinear`)

```
out = x[0]·w[0] + x[1]·w[1] + x[2]·w[2] + b = x @ w + b     (x, w는 벡터, out은 숫자)
```

`w[0]`에 곱해져 있는 건 `x[0]`뿐, `w[1]`에는 `x[1]`. 규칙이 **원소마다 그대로** 적용되고, **코드는 6-a와 똑같습니다.** `*`가 원소별 곱으로 바뀐 것뿐이에요.

```
x = [1, 2, 3], w = [0.5, -1, 2], b = 0.5
out = 5.0
grad_w = [1, 2, 3]  (= x)     grad_b = 1     grad_x = [0.5, -1, 2]  (= w)
```

### 6-c. 출력 여러 개 (`MultiLinear`)

출력이 m개가 되면 `w`가 행렬 `W (n, m)`이 됩니다. 뉴런이 m개인 거예요.

**`grad_W`:** `W[i, j]`는 `x[i]`에 곱해져서 `out[j]`로 갑니다. 그러니 `grad_W[i, j] = x[i] * grad_out[j]`. "행은 `x`를 따라, 열은 `grad_out`을 따라" 가는 표이고, 이걸 만드는 함수가 `np.outer`입니다.

**`grad_b`:** `b[j]`는 `out[j]`에만 더해지니 `grad_out` 그대로.

**`grad_x`:** `x[i]`는 **m개 출력 전부에** 쓰입니다. 그러니 기울기는 m개 경로에서 온 것을 **전부 더한 것**: `W @ grad_out`. 03에서 심어 둔 씨앗이 여기서 나옵니다. **하나의 값이 여러 곳으로 퍼졌으면, 기울기는 되돌아올 때 합친다.**

```python
def backward(self, grad_out):
    self.grad_W = np.outer(self.x, grad_out)    # (n, m)
    self.grad_b = 1 * grad_out                  # (m,)
    return self.W @ grad_out                    # (n,)
```

```
x = [1, 2, 3], W = [[0.5, 1], [-1, 0], [2, -0.5]], grad_out = [1, 1]
grad_W = [[1, 1], [2, 2], [3, 3]]     grad_b = [1, 1]     grad_x = [1.5, -1, 1.5]
```

**버그 2 (shape 에러가 잡음):** 처음에 `self.x @ grad_out`이라고 썼습니다. `(3,) @ (2,)`는 안쪽이 안 맞아 에러가 났고, 설령 맞았어도 `@`는 "짝지어 곱해서 **더함**"이라 숫자 하나가 됩니다. 원하는 건 "모든 (i, j) 짝을 곱하되 **더하지 않는**" 표라서 `np.outer`입니다.

### 6-d. 사진 여러 장 — 드디어 `.T`

사진이 N장이면 `x (N, n)`, `grad_out (N, m)`. `W`와 `b`는 **모든 사진이 공유**하니 그대로입니다.

**핵심 생각:** 사진마다 6-c를 하고, 결과를 **더한다.** `W`는 N장이 공유하니 N장의 신호를 전부 더해야 하죠.

```
grad_W = outer(x[0], g[0]) + outer(x[1], g[1]) + ... + outer(x[N-1], g[N-1])
```

이 "outer를 N번 구해서 더하기"를 **한 번에** 해 주는 게 `x.T @ grad_out`입니다. 행렬 곱의 안쪽 `N`이 **사진에 대해 더하는 자리**이고, `.T`는 `x`를 `(n, N)`으로 세워서 그 더하기가 `N`을 따라 일어나게 하려고 붙습니다.

노트북 셀 11에서 반복문과 행렬 곱이 같은지 확인했습니다.

| 기울기 | 6-c (1장) | 6-d (N장) | 확인 |
|---|---|---|---|
| `grad_W` | `np.outer(x, g)` | `x.T @ grad_out` | 반복문으로 outer를 더한 것과 `True` |
| `grad_b` | `g` | `grad_out.sum(axis=0)` | `g[0]+g[1]+...`과 `True` |
| `grad_x` | `W @ g` | `grad_out @ W.T` | 반복문으로 행마다 `W @ g[k]`한 것과 `True` |

`grad_b`의 `sum(axis=0)`은 03의 "`b`를 N행에 **복사**해서 더했으니 되돌아올 때 **합친다**"입니다.

```python
class Linear:
    def forward(self, x):
        self.x = x                              # grad_W 계산에 필요
        return x @ self.W + self.b

    def backward(self, grad_out):
        self.grad_W = self.x.T @ grad_out       # (in, out)
        self.grad_b = grad_out.sum(axis=0)      # (out,)
        return grad_out @ self.W.T              # (N, in)
```

**버그 3 (에러 메시지의 숫자가 잡음):** `return self.W.T @ grad_out`이라고 써서 `(2,3) @ (4,2)` 에러가 났습니다. 메시지의 "size 4 is different from 3"에서 어느 축이 안 맞는지 역추적했습니다.

### 곱 순서는 목표 shape에서 역산한다

| 만들 것 | 목표 shape | 재료 | 안쪽이 맞는 유일한 배치 |
|---|---|---|---|
| `grad_W` | `(3, 2)` = `W` | `x (4,3)`, `grad_out (4,2)` | `x.T (3,4) @ grad_out (4,2)` |
| `grad_x` | `(4, 3)` = `x` | `grad_out (4,2)`, `W (3,2)` | `grad_out (4,2) @ W.T (2,3)` |

"목표 shape의 첫 숫자를 가진 재료가 앞에 온다." 외울 건 이것 하나입니다.

---

## 7. `Sequential.backward` — forward의 거울

```python
def forward(self, x):
    for layer in self.layers:               # 앞에서부터
        x = layer.forward(x)
    return x

def backward(self, grad):
    for layer in reversed(self.layers):     # 뒤에서부터
        grad = layer.backward(grad)         # 이 레이어가 돌려준 grad가 앞 레이어의 입력
    return grad
```

| | forward | backward |
|---|---|---|
| 도는 순서 | `self.layers` | `reversed(self.layers)` |
| 시키는 일 | `layer.forward(x)` | `layer.backward(grad)` |
| 넘기는 변수 | `x` | `grad` |

반복문이 끝나면 각 `Linear`의 `grad_W`, `grad_b`에 **산출물이 저장**되어 있습니다. 마지막 `return grad`는 맨 앞 레이어의 `grad_x`라 보통 쓰이지 않습니다.

---

## 8. 전체 사슬 검사

```python
model = Sequential([Linear(3, 4), ReLU(), Linear(4, 2)])
loss_fn = SoftmaxCrossEntropy()

loss_fn.forward(model.forward(x), y)
model.backward(loss_fn.backward())          # 손실 → Linear2 → ReLU → Linear1

f = lambda: loss_fn.forward(model.forward(x), y)
for i in [0, 2]:
    print(np.allclose(model.layers[i].grad_W, numerical_grad_array(f, model.layers[i].W)))
    print(np.allclose(model.layers[i].grad_b, numerical_grad_array(f, model.layers[i].b)))
```

`True` 네 번. 손실 함수에서 출발한 기울기가 세 레이어를 거꾸로 통과해 `grad_W` 네 개에 도착했고, 전부 수치 미분과 일치했습니다. 역전파의 핵심인 **연쇄 법칙(Chain Rule)**이 이 셀 하나에 들어 있습니다.

### shape 표 (배치 64)

backward 열을 아래에서 위로 읽으면 forward 열을 거꾸로 쓴 것과 같습니다.

| 단계 | 들어가는 것 | 나오는 것 |
|---|---|---|
| `Linear(784,128)` forward | `(64, 784)` | `(64, 128)` |
| `ReLU` forward | `(64, 128)` | `(64, 128)` |
| `Linear(128,10)` forward | `(64, 128)` | `(64, 10)` |
| `SoftmaxCE` forward | `(64, 10)`, `(64,)` | 숫자 |
| `SoftmaxCE` backward | — | `(64, 10)` |
| `Linear(128,10)` backward | `(64, 10)` | `(64, 128)`; 안에 `grad_W (128,10)`, `grad_b (10,)` |
| `ReLU` backward | `(64, 128)` | `(64, 128)` |
| `Linear(784,128)` backward | `(64, 128)` | `(64, 784)`; 안에 `grad_W (784,128)`, `grad_b (128,)` |

---

## 9. shape 디버깅 도구 상자

"모든 shape을 외워야 디버깅이 되나?"라는 걱정이 있었습니다. 외우는 게 아니라 **도구와 습관**입니다. 이 노트북에서 버그 세 개를 전부 이걸로 잡았습니다.

1. **기울기의 shape은 원본과 같다.** `grad_W.shape == W.shape`, `grad_x.shape == x.shape`. 이 `assert`만으로 곱 순서·전치 실수의 대부분이 걸립니다. 테스트에 넣어 두었습니다.
2. **줄마다 shape을 주석으로 적는다.** `# (n, m)`.
3. **에러 메시지의 숫자를 읽는다.** "size 4 is different from 3"이면 4와 3이 어느 변수의 어느 축인지 찾는다.
4. **작은 숫자로 테스트한다.** 784가 아니라 3, 2, 4. 그래야 손으로 세고 기대값을 적을 수 있다.
5. **막히면 `print(x.shape)`.** 전문가들이 가장 많이 쓰는 방법이다.
6. **수치 미분이 꺾이는 점에서는 믿을 수 없다.** ReLU의 `x = 0`에서 수치 미분은 0.5가 나온다. 테스트에서 0을 피한 이유.

---

## 10. 이 노트북에서 배운 것

| 용어 | 뜻 |
|---|---|
| 기울기 | 조금 밀면 얼마나 변하나 |
| `grad_out` | 뒤 레이어가 넘겨준 "네 출력이 손실에 미치는 영향" |
| 연쇄 법칙 | (내 레이어에서의 변화율) × (뒤에서 온 기울기) |
| 산출물 / 배관 | `grad_W`, `grad_b` 저장 / `grad_x` 반환 |
| 메모 | forward 때 backward에 필요한 것을 `self`에 저장 |
| 마스크 | 통과 1, 차단 0인 배열을 곱해 거르기 |
| `p - onehot` | softmax + cross-entropy를 합쳐 미분한 결과 |
| `np.outer` | 모든 (i, j) 짝을 곱하되 더하지 않는 표 |
| `x.T @ grad_out` | 사진마다 outer를 구해 더한 것을 한 번에 |
| `grad_out.sum(axis=0)` | 복사해서 퍼뜨린 `b`는 되돌아올 때 합친다 |
| `reversed` | 리스트를 뒤에서부터 |
| 수치 기울기 검사 | 공식 없이 밀어 본 값과 공식을 비교. 역전파 검증의 유일한 방법 |
| `np.random.seed` | 난수를 고정해 테스트를 재현 가능하게 |

---

## 스스로 확인해 보기

1. `out = x * w + b`에서 `x = 5`, 뒤에서 온 기울기가 `3`이면 `w`의 기울기는?
2. ReLU에 `[-1, 2, -3]`이 들어갔고 `grad_out = [10, 10, 10]`이면 `backward`의 결과는?
3. `SoftmaxCrossEntropy.backward()`에서 정답 자리의 값이 `-0.9`라면, 모델은 정답에 확률을 얼마나 줬던 걸까요? (N = 1일 때)
4. `grad_W`가 `(784, 128)`이어야 하는데 코드가 `(128, 784)`를 만들었다면, 어떤 실수일 가능성이 높나요?
5. `Linear(784, 128)`의 `grad_b`는 왜 `(64, 128)`이 아니라 `(128,)`인가요?
6. `Sequential.backward`에서 `reversed`를 빼면 무슨 일이 생기나요?
7. backward가 끝난 직후 `W`는 바뀌어 있나요?

<details markdown="1">
<summary>답</summary>

1. `5 × 3 = 15`
2. `[0, 10, 0]`
3. `p - 1 = -0.9`이니 `p = 0.1`. 정답에 10%만 줬다.
4. `x.T @ grad_out` 대신 `grad_out.T @ x`처럼 순서가 뒤집힘. "목표 shape의 첫 숫자(784)를 가진 재료(`x`)가 앞"이어야 한다.
5. `b`는 64장이 공유하는 하나의 벡터라 지시서도 하나. 64행에서 온 기울기를 `sum(axis=0)`으로 합친다.
6. `Linear1.backward`가 `(64,10)`짜리 `grad`를 받게 되어 shape 에러. 기울기는 반드시 손실 쪽(뒤)에서 입력 쪽(앞)으로 흘러야 한다.
7. 아니요. 지시서(`grad_W`, `grad_b`)만 저장됨. 실제 수정은 06의 옵티마이저가 한다.

</details>

---

**이전 글:** [04. 점수 10개를 확률로 바꾸고, "얼마나 틀렸나"를 숫자 하나로 만들기](/posts/2026/10/nn-from-scratch-04-loss/)

**다음 글:** [06. `W = W - 학습률 × grad_W`로 지시서대로 가중치를 고치는 학습 루프. 첫 손실 2.3026이 내려가는 것을 처음으로 보게 됩니다](/posts/2026/10/nn-from-scratch-06-train/)

**시리즈 소개:** [00. 시작하며 (전체 목차)](/posts/2026/10/nn-from-scratch-00-intro/)
