# 14장. PyTorch 입문

📖 [◀ 목차](00-목차.md) | [◀ 이전: 13장. 신경망 기초: 퍼셉트론과 역전파](ch13-신경망기초퍼셉트론과역전파.md) | [다음: 15장. 합성곱 신경망(CNN)과 이미지 분류 ▶](ch15-합성곱신경망CNN.md)

---

13장에서 우리는 2층 신경망의 순전파와 역전파를 NumPy로 직접 구현했습니다. 은닉층이 하나뿐인 모델이었는데도 `backward`를 짜려면 미분을 손으로 유도해야 했습니다. 이 장에서는 그 수고를 덜어 주는 딥러닝 프레임워크 **PyTorch**를 배웁니다.

PyTorch는 크게 세 가지를 제공합니다.

1. **텐서(Tensor)**: NumPy 배열과 닮았지만 GPU에서도 계산할 수 있는 다차원 배열.
2. **자동 미분(autograd)**: 순전파 코드만 쓰면 기울기를 알아서 계산해 주는 기능.
3. **`torch.nn`, `torch.optim`, `torch.utils.data`**: 층, 손실 함수, 옵티마이저, 데이터 공급 도구 모음.

이 장의 목표는 PyTorch의 모든 기능을 훑는 것이 아니라, **"표준 학습 루프" 하나를 정확히 이해하는 것**입니다. 이후 CNN(15장)을 비롯한 모든 딥러닝 코드는 이 루프의 변형입니다. 이미지 데이터는 15장에서 다루므로, 이 장에서는 이미 익숙한 `make_moons`와 `load_digits`(8×8 손글씨 숫자) 데이터를 사용합니다.

> **환경.** PyTorch는 `pip install torch`로 설치합니다. GPU가 없어도 CPU 버전으로 이 장의 모든 코드를 실행할 수 있습니다. GPU(CUDA)용 설치 명령은 운영체제와 CUDA 버전에 따라 다르므로 PyTorch 공식 사이트의 설치 안내에서 환경에 맞는 명령을 확인하세요. 이 장의 출력은 CPU 전용 PyTorch로 실제 실행한 결과이며, 시드를 고정했어도 PyTorch나 플랫폼 버전에 따라 소수점 아래 값은 조금 다를 수 있습니다. 앞 장과 마찬가지로 코드 블록은 하나의 파일처럼 이어서 실행하는 것으로 생각하세요.

```python
import torch

print("PyTorch 버전:", torch.__version__)
print("CUDA(GPU) 사용 가능:", torch.cuda.is_available())
```

**실행 결과**

```text
PyTorch 버전: 2.14.1+cpu
CUDA(GPU) 사용 가능: False
```

## 14.1 텐서

### 텐서 만들기와 shape, dtype, device

텐서는 세 가지 속성으로 요약됩니다.

- **shape**: 각 차원의 크기. NumPy의 `shape`와 같습니다.
- **dtype**: 원소의 자료형. 예를 들어 `torch.float32`, `torch.int64`.
- **device**: 텐서가 놓인 장치. `cpu` 또는 `cuda`(GPU) 등.

```python
import numpy as np
import torch

a = torch.tensor([[1, 2, 3], [4, 5, 6]])     # 파이썬 리스트에서 만들기
b = torch.tensor([1.5, 2.5, 3.5])
z = torch.zeros(2, 3)
r = torch.arange(6).reshape(2, 3)
torch.manual_seed(0)                          # 난수 재현을 위한 시드
n = torch.randn(2, 3)                         # 표준정규분포 난수

for name, t in [("a", a), ("b", b), ("z", z), ("r", r), ("n", n)]:
    print(f"{name}: shape={tuple(t.shape)}, dtype={t.dtype}, device={t.device}")
print(n)
```

**실행 결과**

```text
a: shape=(2, 3), dtype=torch.int64, device=cpu
b: shape=(3,), dtype=torch.float32, device=cpu
z: shape=(2, 3), dtype=torch.float32, device=cpu
r: shape=(2, 3), dtype=torch.int64, device=cpu
n: shape=(2, 3), dtype=torch.float32, device=cpu
tensor([[ 1.5410, -0.2934, -2.1788],
        [ 0.5684, -1.0845, -1.3986]])
```

정수 리스트로 만들면 `int64`, 실수 리스트로 만들면 `float32`입니다. NumPy가 실수를 기본 `float64`로 다루는 것과 다른 점입니다. 신경망의 가중치와 입력은 보통 `float32`를 쓰기 때문에 PyTorch의 기본값이 `float32`입니다(정밀도를 조금 양보하고 메모리와 속도를 얻는 선택입니다).

`torch.manual_seed(0)`은 난수 시드를 고정합니다. NumPy의 `np.random.seed`와 별개이므로 PyTorch 난수의 재현성은 이 함수로 챙겨야 합니다.

### 기본 연산

연산자와 메서드의 이름이 NumPy와 거의 같아서 익숙할 것입니다. 한 가지 차이는 축을 지정하는 인자 이름이 `axis`가 아니라 `dim`이라는 점입니다(NumPy의 `axis`도 일부 호환되지만 `dim`을 쓰는 것이 관례입니다).

```python
x = torch.tensor([[1.0, 2.0], [3.0, 4.0]])
y = torch.tensor([[10.0, 20.0], [30.0, 40.0]])

print("x + y =", (x + y).tolist())
print("x * y (원소별) =", (x * y).tolist())
print("x @ y (행렬곱) =", (x @ y).tolist())
print("열 방향 합계 sum(dim=0):", x.sum(dim=0))
print("행 방향 평균 mean(dim=1):", x.mean(dim=1))
print("브로드캐스팅 x + [100, 200]:", (x + torch.tensor([100.0, 200.0])).tolist())
print("reshape(4):", x.reshape(4))
print("인덱싱 x[0, 1] =", x[0, 1].item(), "  x[:, 0] =", x[:, 0])
```

**실행 결과**

```text
x + y = [[11.0, 22.0], [33.0, 44.0]]
x * y (원소별) = [[10.0, 40.0], [90.0, 160.0]]
x @ y (행렬곱) = [[70.0, 100.0], [150.0, 220.0]]
열 방향 합계 sum(dim=0): tensor([4., 6.])
행 방향 평균 mean(dim=1): tensor([1.5000, 3.5000])
브로드캐스팅 x + [100, 200]: [[101.0, 202.0], [103.0, 204.0]]
reshape(4): tensor([1., 2., 3., 4.])
인덱싱 x[0, 1] = 2.0   x[:, 0] = tensor([1., 3.])
```

`*`는 원소별 곱, `@`는 행렬곱입니다. 브로드캐스팅 규칙도 NumPy와 같아서 `(2, 2)` 텐서에 `(2,)` 텐서를 더하면 각 행에 더해집니다.

### dtype 불일치: 가장 흔한 오류

PyTorch를 처음 쓰면 가장 많이 마주치는 오류가 dtype 불일치입니다.

```python
i = torch.tensor([1, 2, 3])
print("정수 리스트 ->", i.dtype)
print("실수 리스트 ->", torch.tensor([1.0, 2.0]).dtype)
print(".float() 변환 ->", i.float().dtype)

try:
    torch.ones(2, 2, dtype=torch.float64) @ torch.ones(2, 2)   # float64 x float32
except RuntimeError as e:
    print("dtype이 다르면 오류:", e)
```

**실행 결과**

```text
정수 리스트 -> torch.int64
실수 리스트 -> torch.float32
.float() 변환 -> torch.float32
dtype이 다르면 오류: expected m1 and m2 to have the same dtype, but got: double != float
```

앞으로 이 오류를 자주 보게 될 텐데 원인은 대개 둘 중 하나입니다.

- **NumPy 배열을 그대로 텐서로 바꿨는데 `float64`** 라서 `float32`인 모델 가중치와 충돌합니다. 해결: `.float()`로 변환합니다.
- **분류 레이블이 `float32`** 라서 `CrossEntropyLoss`가 거부합니다. 이 손실 함수는 `int64`(`.long()`) 레이블을 요구합니다(14.4절).

### NumPy와의 변환

scikit-learn으로 전처리한 데이터를 PyTorch로 넘기고, 결과를 다시 NumPy로 받는 일은 아주 흔합니다.

```python
arr = np.array([1.0, 2.0, 3.0])
t_shared = torch.from_numpy(arr)    # 메모리를 공유한다
t_copy = torch.tensor(arr)          # 값을 복사한다

arr[0] = 100.0                      # NumPy 배열을 바꾸면...
print("from_numpy:", t_shared)      # 공유한 텐서도 바뀐다
print("tensor    :", t_copy)        # 복사한 텐서는 그대로다

back = t_shared.numpy()             # 텐서 -> NumPy (CPU 텐서에서만 가능, 메모리 공유)
print(type(back).__name__, back.dtype)
```

**실행 결과**

```text
from_numpy: tensor([100.,   2.,   3.], dtype=torch.float64)
tensor    : tensor([1., 2., 3.], dtype=torch.float64)
ndarray float64
```

세 가지를 기억하세요.

- `torch.from_numpy(arr)`는 **메모리를 공유**합니다. 한쪽을 바꾸면 다른 쪽도 바뀝니다. 복사가 필요하면 `torch.tensor(arr)`를 씁니다.
- `tensor.numpy()`도 메모리를 공유하며, **CPU 텐서에서만** 쓸 수 있습니다. GPU 텐서는 `.cpu()`로 먼저 옮겨야 합니다.
- 기울기를 추적하는(`requires_grad=True`) 텐서는 `.detach().numpy()`처럼 계산 그래프에서 떼어낸 뒤에 변환해야 합니다.

## 14.2 자동 미분

### requires_grad와 backward

텐서를 만들 때 `requires_grad=True`를 주면 PyTorch는 그 텐서가 거쳐 가는 모든 연산을 기록합니다. 마지막에 결과(스칼라)에서 `backward()`를 호출하면 기록을 거꾸로 따라가며 연쇄 법칙을 적용하고, 결과가 `.grad`에 저장됩니다.

`y = x² + 2x`의 `x = 3`에서의 미분은 `2x + 2 = 8`입니다.

```python
x = torch.tensor(3.0, requires_grad=True)   # 기울기를 추적할 텐서
y = x ** 2 + 2 * x                          # y = x^2 + 2x  ->  dy/dx = 2x + 2
y.backward()                                # 역전파
print("y =", y.item(), "  dy/dx =", x.grad.item())

# backward()를 또 부르면 기울기가 "누적"된다
y = x ** 2 + 2 * x
y.backward()
print("누적된 x.grad =", x.grad.item())

x.grad.zero_()                              # 비우기
y = x ** 2 + 2 * x
y.backward()
print("비운 뒤 다시 계산한 x.grad =", x.grad.item())
```

**실행 결과**

```text
y = 15.0   dy/dx = 8.0
누적된 x.grad = 16.0
비운 뒤 다시 계산한 x.grad = 8.0
```

두 가지를 확인할 수 있습니다. 첫째, `y.backward()`만으로 `dy/dx = 8`이 나왔습니다. 미분 공식을 코드에 쓴 적이 없는데도 말입니다. 둘째, `backward()`를 다시 호출하면 기울기가 덮어써지지 않고 **누적**됩니다(8 + 8 = 16). 이 동작은 의도된 것이라서, 매 반복마다 기울기를 직접 비워야 합니다. 14.5절의 학습 루프에서 `optimizer.zero_grad()`가 하는 일이 바로 이것입니다. 이 한 줄을 빼먹으면 학습이 조용히 망가집니다.

### 13장의 손으로 유도한 기울기 검증하기

13장의 `tiny_grad`를 기억하시나요? 입력 2, 은닉 2(시그모이드), 출력 1의 아주 작은 네트워크에 샘플 하나를 넣고, 손으로 유도한 역전파로 기울기를 계산했습니다. 같은 숫자를 PyTorch에 넣고 **순전파만 작성한 뒤 `loss.backward()` 한 줄**로 같은 기울기가 나오는지 확인해 보겠습니다.

```python
# 13장 tiny_grad와 똑같은 숫자 (정밀도를 맞추려고 float64 사용)
x = torch.tensor([[0.5, -1.0]], dtype=torch.float64)
t = torch.tensor([[1.0]], dtype=torch.float64)
W1 = torch.tensor([[0.1, 0.4], [-0.3, 0.2]], dtype=torch.float64, requires_grad=True)
b1 = torch.tensor([0.0, 0.1], dtype=torch.float64, requires_grad=True)
W2 = torch.tensor([[0.7], [-0.5]], dtype=torch.float64, requires_grad=True)
b2 = torch.tensor([0.2], dtype=torch.float64, requires_grad=True)

# 순전파만 작성한다
a1 = torch.sigmoid(x @ W1 + b1)
p = torch.sigmoid(a1 @ W2 + b2)
loss = -(t * torch.log(p) + (1 - t) * torch.log(1 - p)).mean()

loss.backward()          # 13장에서 손으로 유도한 역전파가 이 한 줄이다

print("loss =", round(loss.item(), 6))
params = {"W1": W1, "b1": b1, "W2": W2, "b2": b2}
for name, param in params.items():
    print(f"d{name} =", np.round(param.grad.numpy(), 6).tolist())
```

**실행 결과**

```text
loss = 0.53415
dW1 = [[-0.035124, 0.0258], [0.070247, -0.0516]]
db1 = [-0.070247, 0.0516]
dW2 = [[-0.242762], [-0.217254]]
db2 = [-0.413833]
```

13장에서 출력했던 값과 비교해 보겠습니다.

```python
# 13장의 NumPy 역전파가 출력했던 값
ch13 = {
    "W1": [[-0.035124, 0.0258], [0.070247, -0.0516]],
    "b1": [-0.070247, 0.0516],
    "W2": [[-0.242762], [-0.217254]],
    "b2": [-0.413833],
}
for name, param in params.items():
    same = np.allclose(param.grad.numpy(), np.array(ch13[name]), atol=1e-6)
    print(f"{name}: 13장 손계산 기울기와 일치 = {same}")
```

**실행 결과**

```text
W1: 13장 손계산 기울기와 일치 = True
b1: 13장 손계산 기울기와 일치 = True
W2: 13장 손계산 기울기와 일치 = True
b2: 13장 손계산 기울기와 일치 = True
```

13장에서 `dZ2 = P - t`, `dW2 = A1.T @ dZ2`, `dZ1 = dA1 * f'(Z1)`처럼 손으로 전개했던 내용 전부를, PyTorch는 순전파 코드의 연산 기록만으로 똑같이 재현했습니다. 이것이 자동 미분입니다. 층을 열 개 쌓든, 손실 함수를 바꾸든 `backward()` 호출은 변하지 않습니다.

### 기울기가 필요 없을 때: no_grad와 detach

학습이 끝난 모델로 예측만 할 때는 기울기도, 연산 기록도 필요 없습니다. 기록을 하면 메모리와 시간만 낭비되므로 `torch.no_grad()` 블록 안에서 계산합니다.

```python
w = torch.tensor([1.0, 2.0], requires_grad=True)

out = (w * 3).sum()
print("추적 중  :", out.requires_grad, out.grad_fn is not None)

with torch.no_grad():                       # 기울기 추적을 끈다 (추론, 평가용)
    out = (w * 3).sum()
print("no_grad  :", out.requires_grad, out.grad_fn is not None)

detached = (w * 3).detach()                 # 계산 그래프에서 떼어 낸 텐서
print("detach() :", detached.requires_grad)
```

**실행 결과**

```text
추적 중  : True True
no_grad  : False False
detach() : False
```

`grad_fn`은 "이 텐서가 어떤 연산으로 만들어졌는지"를 기억하는 속성이고, 이것들이 이어진 것이 계산 그래프입니다. `no_grad` 안에서 만든 텐서에는 `grad_fn`이 없습니다. `detach()`는 이미 만들어진 텐서를 그래프에서 떼어 냅니다. 14.1절에서 `.detach().numpy()`를 쓰라고 한 이유가 이것입니다.

## 14.3 nn.Module로 모델 만들기

### nn.Linear

13장에서는 `W1`, `b1`을 NumPy 배열로 직접 만들고 `X @ W1 + b1`을 썼습니다. PyTorch에서는 이 한 쌍을 `nn.Linear(입력 수, 출력 수)` 층 하나가 담당합니다. 이 층은 가중치와 편향을 `requires_grad=True`인 텐서로 갖고 있고, 입력 수에 맞춰 크기를 조절한 초기화도 자동으로 적용합니다. 13.6절에서 직접 비교했던 초기화 문제를 알아서 처리해 주는 셈입니다.

```python
import torch.nn as nn

torch.manual_seed(0)
lin = nn.Linear(2, 16)
print("weight shape:", tuple(lin.weight.shape), " bias shape:", tuple(lin.bias.shape))

xs = torch.randn(5, 2)
manual = xs @ lin.weight.T + lin.bias       # 13장의 X @ W1 + b1 과 같은 식
print("lin(xs) == xs @ W.T + b :", torch.allclose(lin(xs), manual))
```

**실행 결과**

```text
weight shape: (16, 2)  bias shape: (16,)
lin(xs) == xs @ W.T + b : True
```

주의할 점이 하나 있습니다. **`weight`의 shape이 `(출력 수, 입력 수)`로, 13장의 `W1 (입력 수, 출력 수)`와 전치 관계**입니다. 그래서 식이 `x @ W.T + b`입니다. 13장의 `W1`과 PyTorch의 `weight`를 서로 옮길 일이 있다면 이 전치를 잊지 마세요.

### nn.Module을 상속해 모델 정의하기

PyTorch의 모든 모델은 `nn.Module`을 상속한 클래스입니다. 규칙은 두 가지뿐입니다.

1. `__init__`에서 `super().__init__()`을 먼저 호출한 뒤, **쓸 층을 속성으로 등록**합니다. 이렇게 하면 `net.parameters()`가 모든 학습 대상 매개변수를 알아서 모아 줍니다.
2. `forward`에서 **순전파만** 작성합니다. 역전파는 자동 미분이 책임집니다.

13장의 `TwoLayerNet`(입력 2 → 은닉 16 → 출력)에 해당하는 모델을 만들어 보겠습니다. 이번에는 출력 뉴런을 2개로 둡니다. 이진 분류를 "클래스 두 개 중 하나를 고르는 문제"로 보고, 클래스마다 점수(logit) 하나씩을 내는 방식입니다. 소프트맥스를 씌우면 두 확률의 합이 1이 되고, 13장의 시그모이드 출력 1개와 수학적으로 같은 표현력을 가집니다.

```python
class MoonsNet(nn.Module):
    def __init__(self, n_in=2, n_hidden=16, n_out=2):
        super().__init__()                       # 반드시 먼저 호출
        self.fc1 = nn.Linear(n_in, n_hidden)     # 층을 속성으로 등록
        self.act = nn.ReLU()
        self.fc2 = nn.Linear(n_hidden, n_out)

    def forward(self, x):                        # 순전파만 정의한다
        return self.fc2(self.act(self.fc1(x)))


net = MoonsNet()
print(net)
for name, p in net.named_parameters():
    print(f"{name:12} shape={tuple(p.shape)}")
print("전체 매개변수 수:", sum(p.numel() for p in net.parameters()))
```

**실행 결과**

```text
MoonsNet(
  (fc1): Linear(in_features=2, out_features=16, bias=True)
  (act): ReLU()
  (fc2): Linear(in_features=16, out_features=2, bias=True)
)
fc1.weight   shape=(16, 2)
fc1.bias     shape=(16,)
fc2.weight   shape=(2, 16)
fc2.bias     shape=(2,)
전체 매개변수 수: 82
```

`print(net)`은 모델의 구조를 요약해 보여 줍니다. 매개변수 수는 `fc1`이 `2×16 + 16 = 48`, `fc2`가 `16×2 + 2 = 34`로 합계 82개입니다. 직접 손으로 세어 보면 모델의 규모감이 잡힙니다.

### nn.Sequential

층을 순서대로 이어 붙이기만 하는 단순한 모델이라면 클래스를 따로 만들 필요 없이 `nn.Sequential`로 충분합니다.

```python
seq = nn.Sequential(
    nn.Linear(2, 16),
    nn.ReLU(),
    nn.Linear(16, 2),
)
print(seq)
print("매개변수 이름:", [name for name, _ in seq.named_parameters()])
print("출력 shape:", tuple(seq(torch.randn(3, 2)).shape))
```

**실행 결과**

```text
Sequential(
  (0): Linear(in_features=2, out_features=16, bias=True)
  (1): ReLU()
  (2): Linear(in_features=16, out_features=2, bias=True)
)
매개변수 이름: ['0.weight', '0.bias', '2.weight', '2.bias']
출력 shape: (3, 2)
```

`Sequential`은 이름 대신 번호로 층을 부릅니다(`0.weight`, `2.weight` 등). 활성화 함수는 `nn.ReLU()`, `nn.Tanh()`, `nn.Sigmoid()`처럼 **층으로서** 끼워 넣을 수 있고, `forward` 안에서 `torch.relu(x)`나 `torch.sigmoid(x)` 같은 **함수로** 호출할 수도 있습니다. 둘은 같은 계산이며, 매개변수가 없는 활성화 함수는 취향 문제입니다. 분기, 건너뛰기 연결, 여러 입력처럼 흐름이 단순한 직선이 아닌 모델은 `nn.Module` 클래스로 직접 `forward`를 작성해야 합니다.

13장의 개념과 PyTorch의 대응을 정리하면 다음과 같습니다.

| 13장에서 직접 구현한 것 | PyTorch |
|--------------------------|---------|
| `W1`, `b1`, `X @ W1 + b1` | `nn.Linear(n_in, n_hidden)` |
| `np.maximum(0, z)` | `nn.ReLU()` 또는 `torch.relu` |
| `forward()` 메서드 | `nn.Module.forward()` |
| `backward()` 메서드와 손계산 미분 | `loss.backward()` (자동 미분) |
| 이진 교차 엔트로피 직접 구현 | `nn.CrossEntropyLoss`, `nn.BCEWithLogitsLoss` |
| `W -= lr * dW` | `optimizer.step()` |
| 미니배치 자르기와 섞기 | `DataLoader` |

## 14.4 손실 함수와 옵티마이저

### 손실 함수

`torch.nn`에는 흔히 쓰는 손실 함수가 준비되어 있습니다. 회귀에는 `nn.MSELoss`(평균 제곱 오차), 다중 클래스 분류에는 `nn.CrossEntropyLoss`를 씁니다.

```python
mse = nn.MSELoss()
pred = torch.tensor([2.5, 0.0, 2.0])
target = torch.tensor([3.0, -0.5, 2.0])
print("MSELoss   :", round(mse(pred, target).item(), 4),
      " 직접 계산:", round(((pred - target) ** 2).mean().item(), 4))

# CrossEntropyLoss: 소프트맥스를 넣기 "전"의 점수(logits)와 정수 클래스 번호를 받는다
logits = torch.tensor([[2.0, 0.5, -1.0],
                       [0.1, 0.2, 3.0]])
labels = torch.tensor([0, 2])                  # 샘플별 정답 클래스 (dtype=int64)
ce = nn.CrossEntropyLoss()

probs = torch.softmax(logits, dim=1)
manual = -torch.log(probs[torch.arange(2), labels]).mean()
print("CrossEntropyLoss:", round(ce(logits, labels).item(), 4),
      " 직접 계산:", round(manual.item(), 4))
print("소프트맥스 확률:\n", probs.round(decimals=3))
```

**실행 결과**

```text
MSELoss   : 0.1667  직접 계산: 0.1667
CrossEntropyLoss: 0.1755  직접 계산: 0.1755
소프트맥스 확률:
 tensor([[0.7860, 0.1750, 0.0390],
        [0.0490, 0.0540, 0.8960]])
```

`nn.CrossEntropyLoss`는 사용법에서 실수가 가장 많은 곳이라 다음 세 가지를 꼭 기억해야 합니다.

1. **입력은 소프트맥스를 통과시키기 전의 점수(logits)** 입니다. 소프트맥스(`log-softmax`)가 손실 함수 안에 이미 들어 있습니다. 모델 마지막에 `nn.Softmax`를 붙이면 소프트맥스가 두 번 적용되어 학습이 느려지고 결과가 이상해집니다. 확률이 필요한 것은 예측을 확인할 때뿐이며 그때는 `torch.softmax`를 따로 호출합니다.
2. **정답은 원-핫이 아니라 클래스 번호**(`int64`)입니다. 위의 `labels = [0, 2]`처럼 한 샘플에 정수 하나입니다.
3. 입력 shape은 `(배치, 클래스 수)`, 정답 shape은 `(배치,)`입니다.

출력이 하나인 이진 분류(13장의 방식)에는 `nn.BCEWithLogitsLoss`를 쓸 수 있습니다. 이름 그대로 시그모이드를 안에 포함한 이진 교차 엔트로피이고, 정답을 `float`으로 받는다는 점이 다릅니다(연습문제 4).

### 옵티마이저

옵티마이저는 기울기를 보고 매개변수를 갱신하는 객체입니다. 생성할 때 **갱신할 매개변수 목록**(`model.parameters()`)과 학습률을 넘기고, 이후 `optimizer.zero_grad()`로 기울기를 비우고 `optimizer.step()`으로 한 걸음 내딛습니다.

- **`torch.optim.SGD`**: 13장의 `W -= lr * dW`와 같은 확률적 경사 하강법입니다. `momentum=0.9`를 주면 이전 진행 방향의 관성을 더해 골짜기에서 덜 흔들리고 더 빨리 내려갑니다.
- **`torch.optim.Adam`**: 매개변수마다 기울기의 크기에 맞춰 보폭을 자동으로 조절합니다. 학습률에 덜 민감해서 처음 시도할 때의 기본 선택으로 많이 쓰입니다. 학습률의 기본값은 0.001입니다.

두 옵티마이저의 차이는 곧 실험으로 확인합니다.

## 14.5 표준 학습 루프

### 다섯 단계

PyTorch의 학습은 항상 같은 다섯 단계를 반복합니다.

```text
1. optimizer.zero_grad()   이전 기울기를 비운다
2. pred = model(x)         순전파
3. loss = loss_fn(pred, y) 손실 계산
4. loss.backward()         역전파 (기울기 계산)
5. optimizer.step()        가중치 갱신
```

가장 단순한 예제로 연습해 보겠습니다. `y = 2x + 1`에 잡음이 낀 데이터를 `nn.Linear(1, 1)` 하나로 학습시킵니다.

```python
torch.manual_seed(0)
X_reg = torch.linspace(-1, 1, 50).unsqueeze(1)           # (50, 1)
y_reg = 2 * X_reg + 1 + 0.1 * torch.randn_like(X_reg)    # y = 2x + 1 + 잡음

model_reg = nn.Linear(1, 1)
loss_fn = nn.MSELoss()
optimizer = torch.optim.SGD(model_reg.parameters(), lr=0.1)

for epoch in range(1, 101):
    optimizer.zero_grad()                  # 1) 이전 기울기 비우기
    pred = model_reg(X_reg)                # 2) 순전파
    loss = loss_fn(pred, y_reg)            # 3) 손실 계산
    loss.backward()                        # 4) 역전파
    optimizer.step()                       # 5) 가중치 갱신
    if epoch in (1, 10, 50, 100):
        print(f"에폭 {epoch:>3}: 손실 = {loss.item():.4f}")

print("학습된 가중치 =", round(model_reg.weight.item(), 3),
      " 편향 =", round(model_reg.bias.item(), 3), " (정답: 2, 1)")
```

**실행 결과**

```text
에폭   1: 손실 = 1.9837
에폭  10: 손실 = 0.3988
에폭  50: 손실 = 0.0125
에폭 100: 손실 = 0.0113
학습된 가중치 = 2.027  편향 = 1.005  (정답: 2, 1)
```

손실이 줄고, 학습된 가중치와 편향이 정답(2, 1)에 가깝게 나왔습니다. 이 다섯 단계는 모델이 선형 층 하나이든 수억 개의 매개변수를 가진 모델이든 똑같습니다.

### make_moons 분류 학습

이제 13장에서 쓴 `make_moons` 문제를 PyTorch로 풀어 보겠습니다. 데이터 분할과 전처리는 13장과 완전히 똑같이 하고, 마지막에 텐서로 바꿉니다. 특성은 `float32`, 분류 레이블은 `int64`로 맞추는 것이 핵심입니다.

```python
from sklearn.datasets import make_moons
from sklearn.model_selection import train_test_split
from sklearn.preprocessing import StandardScaler

# 13장과 똑같은 데이터와 분할
X_all, y_all = make_moons(n_samples=400, noise=0.25, random_state=42)
X_tr, X_te, y_tr, y_te = train_test_split(
    X_all, y_all, test_size=0.25, random_state=42, stratify=y_all)
scaler = StandardScaler().fit(X_tr)
X_tr, X_te = scaler.transform(X_tr), scaler.transform(X_te)

# NumPy -> 텐서: 특성은 float32, 분류 레이블은 int64(long)
X_tr_t = torch.from_numpy(X_tr).float()
X_te_t = torch.from_numpy(X_te).float()
y_tr_t = torch.from_numpy(y_tr).long()
y_te_t = torch.from_numpy(y_te).long()

print("X_tr_t:", tuple(X_tr_t.shape), X_tr_t.dtype, " y_tr_t:", tuple(y_tr_t.shape), y_tr_t.dtype)
```

**실행 결과**

```text
X_tr_t: (300, 2) torch.float32  y_tr_t: (300,) torch.int64
```

평가에 쓸 함수도 하나 만들어 두겠습니다. 평가는 항상 `model.eval()`로 평가 모드를 켜고 `torch.no_grad()` 안에서 계산합니다. `argmax(dim=1)`은 클래스별 점수 중 가장 큰 쪽의 번호를 돌려줍니다.

```python
def accuracy(model, X, y):
    """평가용 함수: eval 모드 + 기울기 추적 끄기."""
    device = next(model.parameters()).device     # 모델이 있는 장치로 데이터를 보낸다
    model.eval()
    with torch.no_grad():
        pred = model(X.to(device)).argmax(dim=1)   # 가장 큰 점수의 클래스
    return (pred == y.to(device)).float().mean().item()
```

학습 루프입니다. 여기서는 데이터 전체를 한 번에 넣는 배치 학습이고, 미니배치는 14.6절에서 다룹니다.

```python
torch.manual_seed(0)
model = nn.Sequential(nn.Linear(2, 16), nn.ReLU(), nn.Linear(16, 2))
loss_fn = nn.CrossEntropyLoss()
optimizer = torch.optim.Adam(model.parameters(), lr=0.01)

for epoch in range(1, 201):
    model.train()                              # 학습 모드
    optimizer.zero_grad()
    logits = model(X_tr_t)                     # 전체 데이터를 한 번에 (배치 = 전체)
    loss = loss_fn(logits, y_tr_t)
    loss.backward()
    optimizer.step()
    if epoch in (1, 10, 50, 100, 200):
        print(f"에폭 {epoch:>3}: 학습 손실 {loss.item():.4f}  "
              f"학습 정확도 {accuracy(model, X_tr_t, y_tr_t):.3f}  "
              f"테스트 정확도 {accuracy(model, X_te_t, y_te_t):.3f}")
```

**실행 결과**

```text
에폭   1: 학습 손실 0.7740  학습 정확도 0.480  테스트 정확도 0.470
에폭  10: 학습 손실 0.5936  학습 정확도 0.727  테스트 정확도 0.830
에폭  50: 학습 손실 0.3555  학습 정확도 0.857  테스트 정확도 0.950
에폭 100: 학습 손실 0.2842  학습 정확도 0.887  테스트 정확도 0.960
에폭 200: 학습 손실 0.1662  학습 정확도 0.933  테스트 정확도 0.940
```

13장에서 NumPy로 구현한 모델과 정확도가 비슷한 수준입니다. 달라진 점은 `backward()`를 직접 짜지 않고 `loss.backward()`를 호출한 것뿐이라는 점에 주목하세요. 학습 정확도와 테스트 정확도가 비슷한 수준이고, 테스트 데이터가 100개뿐이라 0.94~0.96 사이의 오르내림은 우연의 영향이 큽니다.

### train()과 eval(), no_grad의 차이

코드 중간의 `model.train()`과 `model.eval()`, 그리고 `torch.no_grad()`는 이름이 비슷해서 혼동하기 쉽지만 하는 일이 전혀 다릅니다.

| | 하는 일 |
|---|---------|
| `model.train()` / `model.eval()` | **층의 동작 방식**을 바꿉니다. `Dropout`이나 `BatchNorm`처럼 학습과 평가에서 다르게 동작하는 층에만 영향이 있습니다. |
| `torch.no_grad()` | **기울기 추적**을 끕니다. 메모리와 속도 이득만 있고 층의 동작은 바꾸지 않습니다. |

그래서 평가할 때는 보통 **둘 다** 씁니다. 이 모델에는 해당 층이 없어서 `eval()`이 아무 영향이 없지만, 습관으로 항상 챙겨야 합니다. `Dropout`이 어떻게 달라지는지 직접 보겠습니다.

```python
drop = nn.Dropout(p=0.5)
ones = torch.ones(1, 10)

torch.manual_seed(0)
drop.train()                                   # 학습 모드: 무작위로 꺼 버리고 나머지는 2배
print("train 모드:", drop(ones))

drop.eval()                                    # 평가 모드: 아무것도 하지 않는다
print("eval  모드:", drop(ones))
print("model.training 플래그:", drop.training)
```

**실행 결과**

```text
train 모드: tensor([[0., 0., 2., 0., 0., 0., 2., 2., 0., 2.]])
eval  모드: tensor([[1., 1., 1., 1., 1., 1., 1., 1., 1., 1.]])
model.training 플래그: False
```

`Dropout(p=0.5)`은 학습 중에 뉴런의 출력을 무작위로 절반 정도 0으로 꺼 버려서 모델이 특정 뉴런에 과하게 의존하지 못하게 하는 규제 기법입니다. 살아남은 값은 기댓값을 맞추기 위해 `1/(1-p) = 2`배가 됩니다. 평가할 때는 이런 무작위성이 없어야 하므로 `eval()` 모드에서는 입력을 그대로 통과시킵니다. `model.eval()`을 빼먹고 평가하면 같은 입력에도 예측이 매번 달라지는 문제가 생깁니다.

### 옵티마이저 비교

14.4절에서 설명한 옵티마이저 세 가지를 같은 초기 가중치, 같은 데이터, 같은 100에폭으로 비교해 보겠습니다.

```python
def run(make_opt, epochs=100):
    torch.manual_seed(0)                       # 모든 실험이 같은 초기 가중치에서 시작
    m = nn.Sequential(nn.Linear(2, 16), nn.ReLU(), nn.Linear(16, 2))
    opt = make_opt(m.parameters())
    for _ in range(epochs):
        m.train()
        opt.zero_grad()
        loss = loss_fn(m(X_tr_t), y_tr_t)
        loss.backward()
        opt.step()
    return loss.item(), accuracy(m, X_te_t, y_te_t)

configs = [
    ("SGD(lr=0.1)",               lambda p: torch.optim.SGD(p, lr=0.1)),
    ("SGD(lr=0.1, momentum=0.9)", lambda p: torch.optim.SGD(p, lr=0.1, momentum=0.9)),
    ("Adam(lr=0.01)",             lambda p: torch.optim.Adam(p, lr=0.01)),
]
print(f"{'옵티마이저':<28}{'100에폭 후 손실':>16}{'테스트 정확도':>14}")
for name, make_opt in configs:
    loss_value, acc = run(make_opt)
    print(f"{name:<28}{loss_value:>16.4f}{acc:>14.3f}")
```

**실행 결과**

```text
옵티마이저                             100에폭 후 손실       테스트 정확도
SGD(lr=0.1)                           0.3773         0.920
SGD(lr=0.1, momentum=0.9)             0.2263         0.960
Adam(lr=0.01)                         0.2842         0.960
```

순수한 SGD(학습률 0.1)가 가장 느리게 줄었고, 모멘텀을 더한 SGD와 Adam이 더 빨리 내려갔습니다. 13장의 `lr_exp` 실험에서 고생했던 "학습률 고르기"가 옵티마이저에 따라 훨씬 수월해질 수 있다는 뜻입니다. 단, 이 숫자는 한 번의 시드로 얻은 작은 문제의 결과이므로 "Adam이 항상 가장 좋다"는 결론으로 이어서는 안 됩니다. 실무에서는 Adam(또는 변형인 AdamW)으로 시작해 보는 경우가 많다는 정도로 기억하세요.

## 14.6 Dataset과 DataLoader

### 미니배치를 자동으로

13장에서는 `rng.permutation`으로 순서를 섞고 `range(0, N, batch_size)`로 잘라서 미니배치를 직접 만들었습니다. PyTorch는 이 일을 두 클래스로 나눠서 맡습니다.

- **`Dataset`**: "i번째 샘플이 무엇인지"만 알려 주는 객체. `__len__`과 `__getitem__` 두 메서드만 구현하면 됩니다.
- **`DataLoader`**: `Dataset`에서 샘플을 모아 **배치로 묶고, 에폭마다 섞어서** 순서대로 내주는 반복 가능한 객체.

```python
from torch.utils.data import Dataset, DataLoader, TensorDataset

class ArrayDataset(Dataset):
    """NumPy 배열 (X, y)를 텐서로 바꿔 한 샘플씩 꺼내 주는 Dataset."""

    def __init__(self, X, y):
        self.X = torch.from_numpy(X).float()
        self.y = torch.from_numpy(y).long()

    def __len__(self):                         # 샘플 수
        return len(self.y)

    def __getitem__(self, idx):                # idx번째 (입력, 정답) 한 쌍
        return self.X[idx], self.y[idx]


train_ds = ArrayDataset(X_tr, y_tr)
print("샘플 수:", len(train_ds))
print("첫 샘플:", train_ds[0])

train_loader = DataLoader(train_ds, batch_size=32, shuffle=True,
                          generator=torch.Generator().manual_seed(0))
xb, yb = next(iter(train_loader))
print("한 배치:", tuple(xb.shape), xb.dtype, tuple(yb.shape), yb.dtype)
print("에폭당 배치 수:", len(train_loader))

# 텐서가 이미 있다면 TensorDataset이 같은 일을 해 준다
ds2 = TensorDataset(X_tr_t, y_tr_t)
print("TensorDataset 첫 샘플이 같은가:", torch.equal(ds2[0][0], train_ds[0][0]))
```

**실행 결과**

```text
샘플 수: 300
첫 샘플: (tensor([-0.9990,  0.6789]), tensor(1))
한 배치: (32, 2) torch.float32 (32,) torch.int64
에폭당 배치 수: 10
TensorDataset 첫 샘플이 같은가: True
```

300개 샘플을 배치 크기 32로 나누면 10개의 배치(9개는 32개, 마지막은 12개)가 됩니다. `shuffle=True`는 에폭마다 순서를 섞고, `generator`에 시드를 준 이유는 섞는 순서까지 재현하기 위해서입니다. 메모리에 다 올라가는 작은 데이터라면 `ArrayDataset`을 따로 만들 필요 없이 `TensorDataset`이 같은 일을 해 줍니다. 파일에서 이미지를 한 장씩 읽어 오는 것처럼 데이터가 크다면, `__getitem__` 안에서 그때그때 읽도록 `Dataset`을 직접 만듭니다. 15장에서 이 구조를 다시 만나게 됩니다.

`DataLoader`에는 마지막의 작은 배치를 버리는 `drop_last=True`, 데이터 읽기를 여러 프로세스로 나누는 `num_workers` 같은 옵션도 있습니다(Windows에서 `num_workers`를 0보다 크게 쓰려면 스크립트 본문을 `if __name__ == "__main__":` 아래에 두어야 합니다).

### 미니배치 학습 함수

앞으로 계속 쓸 수 있도록 에폭 단위 학습 과정을 함수로 묶겠습니다. 안쪽 `for` 문이 한 배치씩 처리하는 표준 루프입니다.

```python
def fit(model, loader, optimizer, loss_fn, epochs, X_val, y_val, log_every=10):
    device = next(model.parameters()).device
    for epoch in range(1, epochs + 1):
        model.train()
        total_loss = 0.0
        for xb, yb in loader:                      # 미니배치 반복
            xb, yb = xb.to(device), yb.to(device)  # 장치 이동 (14.8절 참고)
            optimizer.zero_grad()
            loss = loss_fn(model(xb), yb)
            loss.backward()
            optimizer.step()
            total_loss += loss.item() * len(xb)    # 배치 크기로 가중해 합산
        if epoch == 1 or epoch % log_every == 0:
            val_acc = accuracy(model, X_val, y_val)
            print(f"에폭 {epoch:>3}  학습 손실 {total_loss / len(loader.dataset):.4f}"
                  f"  검증 정확도 {val_acc:.3f}")
```

`xb.to(device)`는 14.8절의 GPU 사용을 위한 줄입니다. 지금은 CPU뿐이므로 아무 일도 하지 않습니다.

```python
torch.manual_seed(0)
model = nn.Sequential(nn.Linear(2, 16), nn.ReLU(), nn.Linear(16, 2))
optimizer = torch.optim.Adam(model.parameters(), lr=0.01)

fit(model, train_loader, optimizer, nn.CrossEntropyLoss(),
    epochs=100, X_val=X_te_t, y_val=y_te_t, log_every=20)
print(f"최종 테스트 정확도: {accuracy(model, X_te_t, y_te_t):.3f}")
```

**실행 결과**

```text
에폭   1  학습 손실 0.6832  검증 정확도 0.770
에폭  20  학습 손실 0.2596  검증 정확도 0.960
에폭  40  학습 손실 0.1609  검증 정확도 0.950
에폭  60  학습 손실 0.1446  검증 정확도 0.900
에폭  80  학습 손실 0.1335  검증 정확도 0.910
에폭 100  학습 손실 0.1310  검증 정확도 0.910
최종 테스트 정확도: 0.910
```

배치 학습(14.5절)보다 한 에폭에 가중치를 10번(`300 / 32`) 갱신하므로 훨씬 빨리 수렴해서 20에폭에 이미 검증 정확도가 0.96에 도달했습니다. 이후 학습 손실은 계속 줄지만 검증 정확도는 0.96에서 0.91로 내려갔습니다. 학습 데이터에는 점점 더 잘 맞는데 새 데이터에는 오히려 약해지는 **과적합(overfitting)** 의 전형적인 징후입니다(테스트 샘플이 100개뿐이라 그 폭이 과장된 면도 있습니다). 이럴 때는 에폭 수를 줄이거나 규제를 더하거나 검증 성능이 나빠지기 시작하는 지점에서 학습을 멈추는 방법(조기 종료)을 씁니다.

## 14.7 손글씨 숫자 분류 (load_digits)

이번에는 더 현실적인 다중 클래스 문제입니다. `load_digits`는 8×8 흑백 손글씨 숫자 1,797장이 담긴 데이터로, 각 이미지를 64개의 숫자로 펼친 표 형태입니다. 다운로드가 필요 없어서 연습용으로 좋습니다. (이미지의 2차원 구조를 살리는 방법은 15장 CNN에서 배웁니다. 여기서는 64개 픽셀을 그저 64개의 특성으로 보고 `nn.Linear`로 처리합니다.)

```python
from sklearn.datasets import load_digits

digits = load_digits()
Xd = digits.data / 16.0            # 픽셀값 0~16을 0~1로 맞춘다
yd = digits.target
Xd_tr, Xd_te, yd_tr, yd_te = train_test_split(
    Xd, yd, test_size=0.25, random_state=42, stratify=yd)

digits_train = ArrayDataset(Xd_tr, yd_tr)
digits_loader = DataLoader(digits_train, batch_size=64, shuffle=True,
                           generator=torch.Generator().manual_seed(0))
Xd_te_t = torch.from_numpy(Xd_te).float()
yd_te_t = torch.from_numpy(yd_te).long()

print("학습/테스트:", len(Xd_tr), "/", len(Xd_te), " 특성 수:", Xd.shape[1],
      " 클래스 수:", len(np.unique(yd)))
```

**실행 결과**

```text
학습/테스트: 1347 / 450  특성 수: 64  클래스 수: 10
```

픽셀 값(0~16)을 16으로 나누어 0~1 범위로 맞췄습니다. 입력의 크기를 정리해 주는 것은 13장에서 `StandardScaler`를 쓴 것과 같은 이유입니다. 모델은 64 → 64 → 10으로 만들고 중간에 `Dropout`을 넣겠습니다. 출력 10개는 숫자 0~9에 해당하는 점수이고, 손실 함수는 `CrossEntropyLoss`입니다.

```python
def make_digits_model():
    return nn.Sequential(
        nn.Linear(64, 64),
        nn.ReLU(),
        nn.Dropout(0.2),          # 학습할 때만 동작: model.train() / eval()이 중요한 이유
        nn.Linear(64, 10),        # 숫자 0~9에 해당하는 10개의 점수(logits)
    )

torch.manual_seed(0)
digits_model = make_digits_model()
optimizer = torch.optim.Adam(digits_model.parameters(), lr=0.001)

fit(digits_model, digits_loader, optimizer, nn.CrossEntropyLoss(),
    epochs=30, X_val=Xd_te_t, y_val=yd_te_t, log_every=5)
print(f"최종 테스트 정확도: {accuracy(digits_model, Xd_te_t, yd_te_t):.3f}")
```

**실행 결과**

```text
에폭   1  학습 손실 2.2472  검증 정확도 0.536
에폭   5  학습 손실 1.4119  검증 정확도 0.829
에폭  10  학습 손실 0.6918  검증 정확도 0.896
에폭  15  학습 손실 0.4637  검증 정확도 0.920
에폭  20  학습 손실 0.3505  검증 정확도 0.920
에폭  25  학습 손실 0.2798  검증 정확도 0.938
에폭  30  학습 손실 0.2383  검증 정확도 0.944
최종 테스트 정확도: 0.944
```

30에폭 만에 테스트 정확도 0.94를 얻었습니다. 정확도가 에폭 1의 0.536에서 시작해 빠르게 올라가는 것을 볼 수 있습니다. 이 모델은 합성곱이 없는 단순한 모델이라 정확도에 한계가 있지만, 손실이 계속 줄고 있어서 에폭을 더 늘리면 조금 더 오를 여지가 있습니다. 예측과 정답을 몇 개 확인해 보겠습니다. 점수를 확률로 바꾸려면 `torch.softmax`를 씁니다.

```python
digits_model.eval()
with torch.no_grad():
    probs = torch.softmax(digits_model(Xd_te_t[:6]), dim=1)

print("예측 :", probs.argmax(dim=1).tolist())
print("정답 :", yd_te_t[:6].tolist())
print("확신도:", [round(v, 3) for v in probs.max(dim=1).values.tolist()])
```

**실행 결과**

```text
예측 : [1, 0, 9, 1, 5, 6]
정답 : [1, 0, 9, 1, 5, 6]
확신도: [0.254, 0.972, 0.964, 0.858, 0.95, 0.955]
```

처음 여섯 개를 모두 맞혔습니다. 첫 샘플은 정답 `1`을 맞혔지만 확신도가 0.254로 낮습니다. 정확도뿐 아니라 이런 확신도까지 봐야 모델의 상태를 제대로 파악할 수 있습니다.

### 모델 저장과 불러오기

학습된 모델을 파일로 저장했다가 나중에 불러와 쓰는 방법도 알아 둡시다. 권장 방식은 모델 객체 전체가 아니라 매개변수를 담은 `state_dict`만 저장하는 것입니다. 불러올 때는 **같은 구조의 모델을 먼저 만든 뒤** 가중치를 채웁니다. 아래 예제는 임시 폴더에 저장하고 자동으로 지웁니다.

```python
import os
import tempfile

with tempfile.TemporaryDirectory() as tmp:          # 임시 폴더에 저장했다가 자동 삭제
    path = os.path.join(tmp, "digits_model.pt")
    torch.save(digits_model.state_dict(), path)      # 가중치(state_dict)만 저장
    print("저장된 항목:", list(digits_model.state_dict().keys()))

    restored = make_digits_model()                   # 같은 구조의 모델을 먼저 만들고
    restored.load_state_dict(torch.load(path))       # 가중치를 채워 넣는다

print(f"복원한 모델의 테스트 정확도: {accuracy(restored, Xd_te_t, yd_te_t):.3f}")
```

**실행 결과**

```text
저장된 항목: ['0.weight', '0.bias', '3.weight', '3.bias']
복원한 모델의 테스트 정확도: 0.944
```

복원한 모델의 정확도가 원본과 같습니다. `torch.load`는 내부적으로 파이썬의 직렬화 기능을 쓰므로 출처를 알 수 없는 모델 파일은 열지 않는 것이 안전합니다.

## 14.8 GPU 사용하기

신경망의 핵심 계산은 거대한 행렬곱이고, GPU는 이런 계산을 한꺼번에 처리하는 데 특화되어 있습니다. 큰 모델과 큰 데이터에서는 GPU가 CPU보다 수십 배 이상 빠를 수 있습니다. PyTorch에서 GPU를 쓰는 규칙은 단순합니다. **모델과 입력 데이터가 같은 장치에 있어야 한다**는 것입니다.

```python
device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
model = model.to(device)          # 모델의 모든 매개변수를 GPU로
x, y = x.to(device), y.to(device) # 배치마다 데이터도 GPU로
```

GPU가 있는 컴퓨터에서는 `device`가 `cuda`가 되고, 없으면 `cpu`가 되어 같은 코드가 어디서든 동작합니다. 애플 실리콘 맥에서는 `"mps"` 장치를 쓸 수 있고, GPU가 여러 개라면 `"cuda:0"`, `"cuda:1"`처럼 번호를 지정합니다. 이 책을 쓴 환경에는 GPU가 없어서 아래 예제는 CPU에서 실행한 결과입니다. 코드의 구조 자체는 GPU 환경에서도 같습니다.

```python
device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
print("사용 장치:", device)

gpu_model = make_digits_model().to(device)           # 모델의 매개변수를 장치로 이동
xb = Xd_te_t[:5].to(device)                          # 입력도 같은 장치로 이동
print("입력 장치:", xb.device, " 매개변수 장치:", next(gpu_model.parameters()).device)

gpu_model.eval()
with torch.no_grad():
    pred = gpu_model(xb).argmax(dim=1)
pred_np = pred.cpu().numpy()                         # NumPy로 바꾸려면 CPU로 되돌려야 한다
print("예측 결과(NumPy):", pred_np)
```

**실행 결과**

```text
사용 장치: cpu
입력 장치: cpu  매개변수 장치: cpu
예측 결과(NumPy): [7 0 0 7 0]
```

모델의 `.to(device)`는 매개변수를 이동시키고 모델 자신을 반환하므로 `make_digits_model().to(device)`처럼 이어 쓸 수 있습니다. 반면 텐서의 `.to(device)`는 **새 텐서를 반환**하므로 `x = x.to(device)`처럼 다시 대입해야 합니다. 마지막 `pred.cpu().numpy()`는 GPU 텐서를 NumPy로 바꾸려면 CPU로 되돌려야 한다는 14.1절의 규칙입니다. 위 모델은 아직 학습하지 않은 모델이어서 예측값 자체에는 의미가 없습니다.

14.6절에서 만든 `fit`에는 이미 `xb.to(device)`가 들어 있어서 장치만 바꿔서 그대로 쓸 수 있습니다.

```python
# 같은 fit()이 장치만 바꿔서 그대로 동작한다 (여기서는 사용 가능한 장치로 3에폭만 시험)
torch.manual_seed(0)
m = make_digits_model().to(device)
opt = torch.optim.Adam(m.parameters(), lr=0.001)
fit(m, digits_loader, opt, nn.CrossEntropyLoss(), epochs=3,
    X_val=Xd_te_t, y_val=yd_te_t, log_every=1)
```

**실행 결과**

```text
에폭   1  학습 손실 2.2439  검증 정확도 0.584
에폭   2  학습 손실 2.0816  검증 정확도 0.767
에폭   3  학습 손실 1.8798  검증 정확도 0.813
```

GPU가 항상 더 빠른 것은 아닙니다. 위의 숫자 분류 모델처럼 작은 모델에서는 데이터를 GPU로 옮기는 시간이 계산 시간보다 커서 CPU가 오히려 빠를 수 있습니다. GPU는 모델과 배치가 클 때 효과가 큽니다.

### 학습 중 자주 하는 실수

| 증상 | 원인 | 해결 |
|------|------|------|
| 손실이 이상하게 줄지 않거나 불안정 | `optimizer.zero_grad()` 누락으로 기울기가 누적 | 반복마다 `zero_grad()` 호출 |
| `expected ... same dtype` 오류 | 입력이 `float64`이거나 레이블이 `float` | 입력 `.float()`, 분류 레이블 `.long()` |
| 평가 정확도가 매번 달라짐 | `Dropout` 등이 있는데 `model.eval()` 누락 | 평가 전에 `model.eval()` |
| 평가가 느리고 메모리가 급증 | `torch.no_grad()` 누락 | 평가는 `with torch.no_grad():` |
| `Expected all tensors to be on the same device` | 모델과 데이터의 장치 불일치 | 모두 같은 `device`로 `.to(device)` |
| 분류 손실이 이상함 | `CrossEntropyLoss` 앞에 `Softmax`를 붙임 | 모델은 logits를 출력하게 둠 |

## 요약

- **텐서**는 shape, dtype, device를 가진 다차원 배열입니다. 실수 텐서의 기본은 `float32`이고 분류 레이블은 `int64`입니다. dtype 불일치가 가장 흔한 오류입니다.
- `torch.from_numpy`와 `.numpy()`는 메모리를 공유하고, `torch.tensor`는 복사합니다. 기울기를 추적하는 텐서는 `.detach()`가 필요하고 GPU 텐서는 `.cpu()`가 필요합니다.
- **자동 미분**은 순전파의 연산을 기록해 두었다가 `backward()`로 연쇄 법칙을 거꾸로 적용합니다. 13장에서 손으로 유도한 기울기와 소수점 여섯째 자리까지 같음을 확인했습니다. 기울기는 누적되므로 반복마다 비워야 합니다.
- 모델은 `nn.Module`을 상속해 `__init__`에서 층을 등록하고 `forward`에서 순전파만 작성합니다. 단순한 직선 구조는 `nn.Sequential`로 충분합니다. `nn.Linear`의 `weight`는 `(출력, 입력)` 모양입니다.
- 손실은 회귀에 `MSELoss`, 다중 분류에 `CrossEntropyLoss`를 씁니다. `CrossEntropyLoss`는 소프트맥스 이전의 logits와 정수 클래스 번호를 받습니다.
- 옵티마이저는 `SGD`(모멘텀 선택), `Adam` 등이 있고, **표준 학습 루프**는 `zero_grad() → forward → loss → backward() → step()` 다섯 단계입니다.
- `model.train()`/`model.eval()`은 층의 동작 방식을, `torch.no_grad()`는 기울기 추적을 제어합니다. 평가에서는 보통 둘 다 씁니다.
- `Dataset`은 샘플 하나를, `DataLoader`는 배치 묶기와 섞기를 담당합니다.
- GPU를 쓰려면 모델과 데이터를 같은 장치로 `.to(device)` 합니다. 작은 모델에서는 CPU가 더 빠를 수도 있습니다.

## 연습문제

1. **텐서와 dtype.** `np.random.default_rng(0).normal(size=(4, 3))`으로 NumPy 배열을 만들고 텐서로 바꾸세요. `nn.Linear(3, 2)`에 바로 넣으면 어떤 오류가 나는지 확인한 뒤 올바르게 고치세요. 그리고 `torch.from_numpy`로 만든 텐서와 `torch.tensor`로 만든 텐서 중 어느 쪽이 원래 배열의 변경에 영향을 받는지 직접 실험해 확인하세요.

2. **자동 미분 검증.** `f(x, y) = x² · y + y³`에 대해 `(x, y) = (2, 3)`에서의 편미분 `∂f/∂x`와 `∂f/∂y`를 손으로 계산하세요. 그런 다음 `requires_grad=True`인 텐서로 `f`를 계산하고 `backward()`로 얻은 `.grad`와 비교하세요. `.grad`를 비우지 않고 `backward()`를 두 번 호출하면 `.grad`가 어떻게 되는지도 확인해 보세요.

3. **모델 정의.** `nn.Module`을 상속해 입력 2, 은닉 16, 은닉 16, 출력 2의 네트워크를 만들고 활성화 함수는 `tanh`를 쓰세요. 먼저 매개변수 개수를 손으로 계산한 뒤, `sum(p.numel() for p in model.parameters())`로 확인하세요. 같은 모델을 `nn.Sequential`로도 만들어서 `named_parameters()`의 이름이 어떻게 다른지 비교하세요.

4. **이진 분류의 다른 방식.** 14.5절의 `make_moons` 학습을 출력 뉴런 1개와 `nn.BCEWithLogitsLoss`로 바꿔 보세요. 정답은 `y.float().unsqueeze(1)`처럼 `(N, 1)` 모양의 `float`로 바꾸어야 합니다. 예측은 `logits > 0`(시그모이드 후 0.5 초과와 같은 조건)으로 판정하세요. 정확도가 `CrossEntropyLoss` 방식과 비슷하게 나오는지 확인하세요.

5. **load_digits 실험.** 14.7절의 모델에서 다음을 하나씩 바꾸고 테스트 정확도를 비교하세요. (a) `Dropout`을 제거하거나 0.5로 키우기, (b) 은닉 뉴런을 64에서 128로 늘리기, (c) `Adam` 대신 `SGD(lr=0.1, momentum=0.9)` 사용하기. 또한 평가 함수에서 `model.eval()`을 일부러 빼고 `Dropout`이 켜진 상태로 같은 테스트 데이터를 여러 번 평가해서 결과가 매번 달라지는지 관찰해 보세요.

---

📖 [◀ 목차](00-목차.md) | [◀ 이전: 13장. 신경망 기초: 퍼셉트론과 역전파](ch13-신경망기초퍼셉트론과역전파.md) | [다음: 15장. 합성곱 신경망(CNN)과 이미지 분류 ▶](ch15-합성곱신경망CNN.md)
