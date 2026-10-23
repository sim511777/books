# 16장. 시퀀스 데이터와 RNN 입문

📖 [◀ 목차](00-목차.md) | [◀ 이전: 15장. 합성곱 신경망(CNN)과 이미지 분류](ch15-합성곱신경망CNN.md) | [다음: 17장. ML 파이프라인과 모델 저장·배포 ▶](ch17-ML파이프라인과모델배포.md)

---

15장에서 다룬 이미지는 "공간"에 구조가 있는 데이터였다. 이웃한 픽셀끼리 의미 있게 묶이기 때문에, 작은 필터를 이미지 위로 훑는 합성곱이 잘 맞았다. 이 장에서는 구조가 "시간" 또는 "순서"에 있는 데이터, 즉 시퀀스(sequence) 데이터를 다룬다. 문장은 단어가 나열된 순서에 따라 뜻이 달라지고, 기온이나 센서 값은 앞선 시점의 값이 뒤따르는 값과 이어져 있다. 이런 데이터를 신경망으로 다루는 가장 기본적인 구조가 순환 신경망(RNN, Recurrent Neural Network)이다.

이 장의 목표는 RNN을 "쓸 수 있게" 되는 것, 그중에서도 특히 입력과 출력의 shape을 정확히 읽을 수 있게 되는 것이다. 먼저 시퀀스 데이터가 일반 표 형태 데이터와 무엇이 다른지 확인하고, RNN의 핵심 아이디어를 NumPy로 직접 계산해 본다. 그다음 PyTorch의 `nn.RNN`, `nn.LSTM`, `nn.GRU`의 shape 규칙을 확인하고, 사인파를 이용한 시계열 예측과 텍스트를 숫자로 바꾸는 방법까지 살펴본다.

## 16.1 시퀀스 데이터의 특징

### 16.1.1 어떤 데이터가 시퀀스인가

시퀀스 데이터는 "원소들이 순서를 가지고 나열된" 데이터다. 대표적인 예는 다음과 같다.

- 텍스트: 단어 또는 문자의 나열. 문장마다 길이가 다르다.
- 시계열: 시간 간격으로 측정한 값의 나열. 기온, 주가, 서버 트래픽, 센서 신호 등이 있다.
- 오디오: 짧은 시간 간격으로 샘플링한 음압 값의 나열.
- 생물학적 서열: DNA 염기 서열처럼 문자로 이루어진 긴 나열.

이런 데이터에는 지금까지 다룬 표 형태 데이터와 구별되는 세 가지 특징이 있다.

첫째, 순서가 곧 정보다. "개가 사람을 물었다"와 "사람이 개를 물었다"는 같은 단어로 이루어져 있지만 뜻이 정반대다. 둘째, 길이가 일정하지 않다. 짧은 문장도 있고 긴 문장도 있는데, 지금까지의 모델은 입력 특성의 개수가 고정되어 있다고 가정했다. 셋째, 문맥이 중요하다. 문장 끝의 "그것"이 무엇을 가리키는지는 앞에 나온 단어를 알아야 판단할 수 있고, 내일의 기온은 오늘과 어제의 기온을 알아야 예측할 수 있다.

### 16.1.2 시계열을 학습 데이터로 바꾸기: 슬라이딩 윈도우

"다음 값 예측" 문제를 지도학습으로 바꾸는 가장 흔한 방법은 슬라이딩 윈도우(sliding window)다. 연속된 `window`개의 값을 입력 `X`로, 바로 다음 값을 정답 `y`로 삼고, 한 칸씩 밀어가며 샘플을 만든다.

```python
import numpy as np

series = np.arange(10, dtype=float)       # 0, 1, 2, ..., 9
window = 4

X = np.array([series[i:i + window] for i in range(len(series) - window)])
y = series[window:]

print("X shape:", X.shape, " y shape:", y.shape)
print(X[:3])
print(y[:3])
```

```
X shape: (6, 4)  y shape: (6,)
[[0. 1. 2. 3.]
 [1. 2. 3. 4.]
 [2. 3. 4. 5.]]
[4. 5. 6.]
```

첫 샘플은 `[0, 1, 2, 3]`을 보고 `4`를 맞히는 문제이고, 다음 샘플은 한 칸 밀려 `[1, 2, 3, 4]`를 보고 `5`를 맞힌다. 이렇게 만든 `X`의 shape은 `(샘플 수, 윈도우 길이)`다. RNN에 넣을 때는 여기에 "시점마다 특성이 몇 개인가"를 나타내는 축을 하나 더 붙여 `(샘플 수, 윈도우 길이, 특성 수)`로 만든다. 값이 하나뿐인 시계열이라면 특성 수는 1이다. 16.5절에서 이 형태를 실제로 사용한다.

### 16.1.3 순서를 버리면 무엇을 잃는가

텍스트를 숫자로 바꾸는 전통적인 방법으로 단어 빈도를 세는 Bag of Words가 있다. scikit-learn의 `CountVectorizer`가 이 방식을 구현한다. 이 방식이 순서를 어떻게 다루는지 확인해 보자.

```python
from sklearn.feature_extraction.text import CountVectorizer

docs = ["dog bites man", "man bites dog"]

vec = CountVectorizer()
bow = vec.fit_transform(docs)

print(vec.get_feature_names_out())
print(bow.toarray())
```

```
['bites' 'dog' 'man']
[[1 1 1]
 [1 1 1]]
```

두 문장은 뜻이 전혀 다르지만 단어 빈도 벡터는 완전히 같다. 순서 정보가 사라졌기 때문이다. 문서 분류처럼 "어떤 단어가 등장하는가"만으로 충분한 문제에서는 이 방식도 꽤 잘 동작하지만, 순서와 문맥이 중요한 문제에서는 한계가 분명하다. 시퀀스를 앞에서부터 한 원소씩 읽어 가며 문맥을 쌓아 가는 모델, 그것이 RNN이다.

시계열 윈도우를 그냥 펼쳐서 일반 MLP나 선형 모델에 넣는 방법도 가능하지만, 이 경우 윈도우 길이가 고정되어야 하고, 시점마다 완전히 별개의 가중치가 생기며, 시점이 하나 늘 때마다 파라미터도 함께 늘어난다. RNN은 모든 시점에서 같은 가중치를 재사용하는 방식으로 이 문제를 풀어 간다.

## 16.2 RNN의 아이디어: 기억을 다음 시점으로 넘긴다

RNN의 아이디어는 간단하다. 시퀀스를 한 시점씩 읽으면서, 지금까지 읽은 내용을 요약한 벡터를 하나 들고 다닌다. 이 벡터를 은닉 상태(hidden state)라 부르고 `h`로 쓴다. 매 시점 `t`마다 RNN은 현재 입력 `x_t`와 이전 은닉 상태 `h_(t-1)`을 받아서 새 은닉 상태 `h_t`를 만든다.

```
h_t = tanh(W_x · x_t + W_h · h_(t-1) + b)
```

여기서 `W_x`는 입력에 곱하는 가중치, `W_h`는 이전 은닉 상태에 곱하는 가중치, `b`는 편향이고, `tanh`는 값을 -1과 1 사이로 눌러 주는 활성화 함수다. 핵심은 `W_x`, `W_h`, `b`가 모든 시점에서 같다는 점이다. 15장의 CNN이 같은 필터를 이미지 전체에 재사용했다면, RNN은 같은 가중치를 시간 전체에 재사용한다. 덕분에 시퀀스가 길어져도 파라미터 수는 늘지 않고, 길이가 다른 시퀀스도 같은 모델로 처리할 수 있다.

시간 축으로 펼쳐서 그려 보면 구조가 한눈에 보인다.

```
 h_0 ──▶ [ RNN 셀 ] ──▶ h_1 ──▶ [ RNN 셀 ] ──▶ h_2 ──▶ [ RNN 셀 ] ──▶ h_3
              ▲                       ▲                       ▲
             x_1                     x_2                     x_3
        (같은 가중치 W_x, W_h, b를 세 번 모두 재사용)
```

### 16.2.1 NumPy로 RNN 순전파 구현하기

위 식을 그대로 코드로 옮겨 보자. 학습은 하지 않고, 임의의 가중치로 순전파만 계산해서 은닉 상태가 시점마다 어떻게 이어지는지 본다.

```python
import numpy as np

rng = np.random.default_rng(0)

input_size, hidden_size = 2, 3
W_x = rng.normal(0, 0.5, (hidden_size, input_size))
W_h = rng.normal(0, 0.5, (hidden_size, hidden_size))
b = np.zeros(hidden_size)

def rnn_forward(seq):
    """seq: (시점 수, input_size) -> 모든 시점의 은닉 상태 (시점 수, hidden_size)"""
    h = np.zeros(hidden_size)             # 초기 은닉 상태는 0
    states = []
    for x_t in seq:
        h = np.tanh(W_x @ x_t + W_h @ h + b)
        states.append(h)
    return np.array(states)

seq = np.array([[1.0, 0.0],
                [0.0, 1.0],
                [0.5, 0.5],
                [1.0, 1.0]])

states = rnn_forward(seq)
print("states shape:", states.shape)
print(np.round(states, 3))

reversed_states = rnn_forward(seq[::-1])
print("원래 순서의 마지막 h:", np.round(states[-1], 3))
print("뒤집은 순서의 마지막 h:", np.round(reversed_states[-1], 3))
```

```
states shape: (4, 3)
[[ 0.063  0.31  -0.262]
 [ 0.21  -0.089  0.233]
 [ 0.012  0.086 -0.4  ]
 [ 0.183  0.319  0.138]]
원래 순서의 마지막 h: [0.183 0.319 0.138]
뒤집은 순서의 마지막 h: [ 0.099  0.276 -0.341]
```

(난수를 쓰는 예제이므로 NumPy 버전에 따라 소수점 아래 값은 조금 다를 수 있다.)

입력이 4개 시점이므로 은닉 상태도 4개이고, 각 은닉 상태는 `hidden_size=3`차원이다. 같은 네 개의 입력을 순서만 뒤집어 넣었더니 마지막 은닉 상태가 달라졌다. 은닉 상태가 "지금까지 읽은 내용의 요약"이기 때문에 읽은 순서에 따라 요약도 달라지는 것이다. 이 마지막 은닉 상태를 분류기에 연결하면 "시퀀스 전체를 읽고 판단하는" 모델이 되고, 매 시점의 은닉 상태를 각각 출력으로 쓰면 "시점마다 예측하는" 모델이 된다.

## 16.3 기울기 소실과 LSTM, GRU

### 16.3.1 시간을 거슬러 흐르는 기울기

RNN도 신경망이므로 역전파로 학습한다. 시간 축으로 펼친 구조에 역전파를 적용하는 것을 BPTT(Backpropagation Through Time)라 한다. 예를 들어 50번째 시점의 오차가 첫 번째 시점의 입력에 어떤 영향을 주는지 계산하려면, 오차 신호가 50개의 시점을 거슬러 올라오면서 매 시점마다 같은 `W_h`와 `tanh`의 미분값에 곱해져야 한다.

같은 종류의 곱셈을 수십 번 반복하면 결과는 거의 항상 둘 중 하나로 치우친다. 곱하는 값이 1보다 작으면 0에 가깝게 줄어들고(기울기 소실, vanishing gradient), 1보다 크면 급격히 커진다(기울기 폭발, exploding gradient). 예를 들어 0.9를 50번 곱하면 약 0.005이고, 0.5를 50번 곱하면 사실상 0이다. `tanh`의 미분값은 항상 1 이하이기 때문에 기본 RNN은 특히 소실 쪽으로 기울기 쉽다.

이 현상을 직접 숫자로 확인해 보자. 은닉 상태 `h_t`가 첫 은닉 상태 `h_0`에 얼마나 민감한지, 즉 `∂h_t/∂h_0`의 크기를 시점별로 계산한다.

```python
import numpy as np

rng = np.random.default_rng(1)
H = 8
W_h = rng.normal(0, 1.0 / np.sqrt(H), (H, H))
W_x = rng.normal(0, 0.5, (H, 1))
print("W_h의 최대 고유값 크기:", np.abs(np.linalg.eigvals(W_h)).max().round(3))

h = np.zeros(H)
J = np.eye(H)                 # J = d h_t / d h_0, 처음에는 단위 행렬
norms = []
for t in range(1, 51):
    x_t = rng.normal(size=1)
    h = np.tanh(W_x @ x_t + W_h @ h)
    J = np.diag(1 - h ** 2) @ W_h @ J       # 연쇄 법칙: 한 시점마다 곱해진다
    norms.append(np.linalg.norm(J))

for t in [1, 5, 10, 20, 30, 50]:
    print(f"t={t:2d}  ||dh_t/dh_0|| = {norms[t - 1]:.2e}")
```

```
W_h의 최대 고유값 크기: 0.739
t= 1  ||dh_t/dh_0|| = 2.16e+00
t= 5  ||dh_t/dh_0|| = 3.69e-01
t=10  ||dh_t/dh_0|| = 3.80e-02
t=20  ||dh_t/dh_0|| = 3.52e-04
t=30  ||dh_t/dh_0|| = 6.21e-06
t=50  ||dh_t/dh_0|| = 6.49e-10
```

50 시점 전의 정보가 현재에 미치는 영향이 10억 분의 1 수준으로 줄었다. 이 상태에서 학습하면 "먼 과거에 있던 단서"가 오차에 기여한다는 신호가 앞쪽 시점까지 거의 전달되지 않는다. 결과적으로 기본 RNN은 몇 시점 이내의 단기 의존성은 배우지만, 수십 시점 떨어진 관계는 잘 배우지 못한다. 반대로 `W_h`가 크면 기울기가 폭발해 학습이 불안정해지는데, 이는 PyTorch의 `torch.nn.utils.clip_grad_norm_`으로 기울기 크기에 상한을 두는 방법(기울기 클리핑)으로 흔히 대응한다.

### 16.3.2 LSTM과 GRU의 직관

기울기 소실에 대한 대표적인 해법이 LSTM(Long Short-Term Memory)과 GRU(Gated Recurrent Unit)다. 두 구조의 공통된 직관은 "기억을 덮어쓰기만 하지 말고, 무엇을 지우고 무엇을 남길지 게이트(gate)로 조절하자"는 것이다.

LSTM은 은닉 상태 `h_t` 외에 셀 상태(cell state) `c_t`라는 별도의 기억 통로를 가진다. 셀 상태는 컨베이어 벨트에 비유할 수 있다. 벨트 위의 내용은 기본적으로 변하지 않고 그대로 다음 시점으로 흘러가며, 세 개의 게이트가 벨트를 조작한다.

- 망각 게이트(forget gate): 벨트에 실린 기억 중 얼마나 지울지 정한다.
- 입력 게이트(input gate): 현재 입력에서 얻은 새 정보를 얼마나 벨트에 올릴지 정한다.
- 출력 게이트(output gate): 벨트의 기억 중 얼마를 이번 시점의 출력(`h_t`)으로 꺼낼지 정한다.

게이트는 0과 1 사이의 값을 내는 작은 신경망이다. 기억이 덧셈 위주로 갱신되고 "그대로 통과"가 쉬운 구조이기 때문에, 오차 신호가 먼 과거로 거슬러 올라갈 때 매번 `W_h`에 곱해지며 줄어들던 문제가 크게 완화된다.

GRU는 LSTM을 단순화한 구조다. 셀 상태를 따로 두지 않고 은닉 상태 하나만 사용하며, 게이트도 두 개(업데이트 게이트, 리셋 게이트)뿐이다. 파라미터가 적고 계산이 가벼운데도 많은 문제에서 LSTM과 비슷한 성능을 낸다. 어느 쪽이 더 낫다고 일반화하기는 어려워서, 실무에서는 둘 다 시험해 보고 고르는 경우가 많다.

## 16.4 `nn.RNN`, `nn.LSTM`, `nn.GRU`의 입력과 출력 shape

RNN 계열 모듈을 처음 쓸 때 가장 많이 헷갈리는 부분이 shape이다. 작은 예제로 직접 확인해 보자. 배치 크기 `N=4`, 시퀀스 길이 `L=7`, 시점마다 입력 특성 `F=3`개, 은닉 크기 `H=5`인 입력을 만든다.

```python
import torch
from torch import nn

torch.manual_seed(0)

N, L, F, H = 4, 7, 3, 5         # 배치, 길이, 입력 특성 수, 은닉 크기
x = torch.randn(N, L, F)

rnn = nn.RNN(input_size=F, hidden_size=H, batch_first=True)
out, h_n = rnn(x)
print("RNN  out:", tuple(out.shape), " h_n:", tuple(h_n.shape))
print("마지막 시점 출력 == h_n:", torch.allclose(out[:, -1, :], h_n[0]))

lstm = nn.LSTM(F, H, batch_first=True)
out, (h_n, c_n) = lstm(x)
print("LSTM out:", tuple(out.shape), " h_n:", tuple(h_n.shape), " c_n:", tuple(c_n.shape))

gru = nn.GRU(F, H, batch_first=True)
out, h_n = gru(x)
print("GRU  out:", tuple(out.shape), " h_n:", tuple(h_n.shape))
```

```
RNN  out: (4, 7, 5)  h_n: (1, 4, 5)
마지막 시점 출력 == h_n: True
LSTM out: (4, 7, 5)  h_n: (1, 4, 5)  c_n: (1, 4, 5)
GRU  out: (4, 7, 5)  h_n: (1, 4, 5)
```

규칙을 정리하면 다음과 같다.

| 항목 | shape | 의미 |
|------|-------|------|
| 입력 `x` (`batch_first=True`) | `(N, L, F)` | 배치, 시점 수, 시점별 특성 수 |
| `out` | `(N, L, H)` | 모든 시점의 은닉 상태(마지막 층 기준) |
| `h_n` | `(층 수, N, H)` | 마지막 시점의 은닉 상태(층마다 하나) |
| `c_n` (LSTM만) | `(층 수, N, H)` | 마지막 시점의 셀 상태 |

`out`은 "모든 시점"의 은닉 상태이고 `h_n`은 "마지막 시점"의 은닉 상태이므로, 층이 하나인 단방향 모델에서는 `out[:, -1, :]`과 `h_n[0]`이 같은 값이다. 위 출력의 `True`가 이를 확인해 준다. LSTM은 반환값이 `(out, (h_n, c_n))`으로 한 겹 더 묶여 있다는 점도 기억해 두자.

### 16.4.1 `batch_first`의 의미

`nn.RNN`, `nn.LSTM`, `nn.GRU`는 `batch_first` 인자를 `False`로 두는 것이 기본값이다. 이때 입력은 `(L, N, F)`, 즉 "시점 수가 맨 앞"인 shape을 기대한다. 이 책에서는 데이터 shape이 `(샘플, 시점, 특성)`이어서 이해하기 쉬운 `batch_first=True`를 사용한다. 두 설정은 입력과 `out`의 축 순서만 바꾼다. `h_n`과 `c_n`은 `batch_first` 값과 상관없이 항상 `(층 수, N, H)` 순서이므로 주의해야 한다.

### 16.4.2 층 수와 양방향

`num_layers`를 늘려 RNN을 쌓거나 `bidirectional=True`로 시퀀스를 앞뒤 양방향으로 읽게 할 수 있다. 이때 shape은 다음처럼 변한다.

```python
bi = nn.LSTM(F, H, num_layers=2, bidirectional=True, batch_first=True)
out, (h_n, c_n) = bi(x)
print("2층 양방향 out:", tuple(out.shape), " h_n:", tuple(h_n.shape))

rnn_seq_first = nn.RNN(F, H)                 # batch_first=False (기본값)
out, h_n = rnn_seq_first(x.transpose(0, 1))  # (L, N, F)로 바꿔서 입력
print("batch_first=False out:", tuple(out.shape), " h_n:", tuple(h_n.shape))

for name, p in lstm.named_parameters():
    print(name, tuple(p.shape))
```

```
2층 양방향 out: (4, 7, 10)  h_n: (4, 4, 5)
batch_first=False out: (7, 4, 5)  h_n: (1, 4, 5)
weight_ih_l0 (20, 3)
weight_hh_l0 (20, 5)
bias_ih_l0 (20,)
bias_hh_l0 (20,)
```

양방향이면 앞방향과 뒤방향의 은닉 상태가 이어 붙기 때문에 `out`의 마지막 축이 `2 * H = 10`이 되고, `h_n`의 첫 축은 `층 수 × 방향 수 = 2 × 2 = 4`가 된다. 파라미터 이름의 `ih`는 입력(input)에서 은닉(hidden)으로 가는 가중치, `hh`는 은닉에서 은닉으로 가는 가중치를 뜻한다. LSTM의 `weight_ih_l0`가 `(20, 3)`인 이유는 내부적으로 네 묶음(세 개의 게이트와 새 기억 후보)의 가중치가 `4 × H = 20`행으로 이어 붙어 있기 때문이다. 같은 이유로 GRU는 세 묶음이어서 `3 × H`행이 된다.

## 16.5 실습: 사인파의 다음 값 예측하기

이제 RNN으로 간단한 시계열 예측을 해 보자. 잡음이 약간 섞인 사인파를 만들고, 직전 20개 시점의 값을 보고 다음 값 하나를 예측하는 LSTM을 학습한다. 데이터가 단순해서 CPU에서 1초 안에 학습이 끝난다.

### 16.5.1 데이터 만들기와 분할

```python
import numpy as np
import torch
from torch import nn

torch.manual_seed(0)
rng = np.random.default_rng(0)

# 사인파 + 약간의 잡음 (0.1 간격으로 600개 점)
t = np.arange(0, 60, 0.1)
series = (np.sin(t) + rng.normal(0, 0.05, size=t.shape)).astype(np.float32)

# 슬라이딩 윈도우: (샘플, 20, 1)과 정답 (샘플, 1)
window = 20
X = np.stack([series[i:i + window] for i in range(len(series) - window)])
y = series[window:]
X = X[:, :, None]
y = y[:, None]
print("X:", X.shape, "y:", y.shape)

# 시간 순서대로 앞 80%는 학습, 뒤 20%는 테스트
split = int(len(X) * 0.8)
X_train, X_test = torch.from_numpy(X[:split]), torch.from_numpy(X[split:])
y_train, y_test = torch.from_numpy(y[:split]), torch.from_numpy(y[split:])
print("train:", tuple(X_train.shape), "test:", tuple(X_test.shape))
```

```
X: (580, 20, 1) y: (580, 1)
train: (464, 20, 1) test: (116, 20, 1)
```

여기서 중요한 점은 학습/테스트 분할에 `train_test_split`처럼 샘플을 무작위로 섞는 방식을 쓰지 않았다는 것이다. 시계열에서는 "과거로 학습하고 미래로 평가"하는 것이 실제 사용 상황과 같다. 무작위로 섞으면 테스트 샘플의 바로 앞뒤 값이 학습 데이터에 들어가 있어서, 모델이 미래를 예측하는 것이 아니라 이미 본 값 사이를 메우는 문제가 되어 성능이 실제보다 좋게 보인다.

### 16.5.2 모델 정의와 학습

모델은 LSTM 한 층과 선형층 하나다. LSTM이 20개 시점을 읽으면서 은닉 상태를 갱신하고, 마지막 시점의 은닉 상태를 선형층에 넣어 다음 값 하나를 출력한다.

```python
class SeqRegressor(nn.Module):
    def __init__(self, hidden_size=32):
        super().__init__()
        self.lstm = nn.LSTM(input_size=1, hidden_size=hidden_size, batch_first=True)
        self.head = nn.Linear(hidden_size, 1)

    def forward(self, x):                     # x: (N, 20, 1)
        out, (h_n, c_n) = self.lstm(x)        # out: (N, 20, hidden)
        return self.head(out[:, -1, :])       # 마지막 시점만 사용 -> (N, 1)

model = SeqRegressor()
loss_fn = nn.MSELoss()
optimizer = torch.optim.Adam(model.parameters(), lr=0.01)

for epoch in range(1, 151):
    model.train()
    optimizer.zero_grad()
    loss = loss_fn(model(X_train), y_train)   # 데이터가 작아 전체를 한 번에 사용
    loss.backward()
    optimizer.step()

    if epoch % 30 == 0:
        model.eval()
        with torch.no_grad():
            test_loss = loss_fn(model(X_test), y_test).item()
        print(f"epoch {epoch:3d}  train MSE {loss.item():.4f}  test MSE {test_loss:.4f}")
```

```
epoch  30  train MSE 0.0083  test MSE 0.0051
epoch  60  train MSE 0.0049  test MSE 0.0037
epoch  90  train MSE 0.0045  test MSE 0.0034
epoch 120  train MSE 0.0043  test MSE 0.0032
epoch 150  train MSE 0.0040  test MSE 0.0030
```

학습 데이터가 464개뿐이라 미니배치로 나누지 않고 매 에폭마다 전체를 한 번에 넣었다. 13~14장의 학습 루프에서 `DataLoader`로 배치를 나누던 부분이 빠졌을 뿐, `zero_grad → 순전파 → loss → backward → step`의 흐름은 그대로다.

### 16.5.3 얼마나 잘 맞히는가

MSE 숫자만으로는 좋은지 나쁜지 알기 어렵다. 기준선(baseline)과 비교해 보자. 시계열에서 가장 단순한 기준선은 "직전 값을 그대로 다음 값으로 예측"하는 것이다.

```python
model.eval()
with torch.no_grad():
    pred = model(X_test)

mse_lstm = loss_fn(pred, y_test).item()
mse_last = loss_fn(X_test[:, -1, :], y_test).item()       # 직전 값 복사
print(f"LSTM MSE: {mse_lstm:.4f}   직전 값 복사 MSE: {mse_last:.4f}")
print(f"잡음 분산: {0.05 ** 2:.4f}")

for i in range(0, 10, 3):
    print(f"실제 {y_test[i].item():+.3f}  예측 {pred[i].item():+.3f}")
```

```
LSTM MSE: 0.0030   직전 값 복사 MSE: 0.0093
잡음 분산: 0.0025
실제 -0.992  예측 -0.937
실제 -1.017  예측 -1.039
실제 -0.993  예측 -0.939
실제 -0.856  예측 -0.815
```

LSTM의 MSE(0.0030)는 직전 값 복사(0.0093)의 약 3분의 1이다. 또 우리가 데이터에 섞은 잡음의 분산이 0.0025이므로, 예측하는 값 자체에 이만큼의 잡음이 있어서 어떤 모델도 MSE를 이보다 크게 낮출 수 없다. 0.0030은 그 한계에 꽤 가까운 값이다. 직전 값 복사가 의외로 나쁘지 않은 기준선이라는 점도 기억해 두자. 실제 문제에서 복잡한 모델이 이 단순한 기준선조차 이기지 못한다면 모델 설계나 데이터 처리에 문제가 있다는 신호다.

### 16.5.4 여러 걸음 앞 예측하기

학습한 모델은 한 걸음 앞만 예측하지만, 예측값을 다시 입력 윈도우의 끝에 붙여 가며 반복하면 여러 걸음 앞까지 이어서 예측할 수 있다. 이를 자기회귀(autoregressive) 방식의 예측이라고 한다.

```python
with torch.no_grad():
    w = X_test[0:1].clone()                   # 첫 테스트 윈도우 (1, 20, 1)
    preds = []
    for _ in range(30):
        nxt = model(w)                        # (1, 1)
        preds.append(nxt.item())
        # 가장 오래된 값을 버리고 새 예측값을 끝에 붙인다
        w = torch.cat([w[:, 1:, :], nxt.reshape(1, 1, 1)], dim=1)

true_clean = np.sin(t[split + window: split + window + 30])    # 잡음 없는 정답
mse_30 = float(np.mean((np.array(preds) - true_clean) ** 2))
print(f"30걸음 자기회귀 예측 MSE (잡음 없는 사인파 기준): {mse_30:.4f}")
```

```
30걸음 자기회귀 예측 MSE (잡음 없는 사인파 기준): 0.0020
```

예측값이 다음 입력으로 되먹임되기 때문에 오차가 쌓일 수 있지만, 이 예제는 사인파라는 규칙적인 신호여서 30걸음을 가도 파형이 유지된다. 실제 주가나 날씨처럼 불규칙한 시계열에서는 멀리 갈수록 오차가 빠르게 커진다.

## 16.6 텍스트를 숫자로 바꾸기: 토큰화와 임베딩

시계열은 이미 숫자지만 텍스트는 그렇지 않다. 텍스트를 RNN에 넣으려면 다음 단계를 거친다.

1. 토큰화(tokenization): 문장을 토큰(단어, 문자, 부분 단어)의 나열로 쪼갠다.
2. 어휘 사전(vocabulary): 토큰마다 정수 ID를 부여한다.
3. 패딩(padding): 길이가 다른 문장을 배치로 묶기 위해 짧은 문장 뒤를 특수 값으로 채운다.
4. 임베딩(embedding): 정수 ID를 학습 가능한 실수 벡터로 바꾼다.

### 16.6.1 토큰화, 어휘 사전, 패딩

가장 단순한 공백 기준 토큰화로 작은 예제를 만들어 보자.

```python
import torch
from torch import nn
from torch.nn.utils.rnn import pad_sequence

sentences = [
    "i love this movie",
    "this movie is boring",
    "i love it",
    "boring boring boring plot but i love it",
]

# 1) 토큰화: 공백 기준
tokenized = [s.split() for s in sentences]
print(tokenized[1])

# 2) 어휘 사전: 0번은 패딩용으로 예약해 둔다
vocab = {"<pad>": 0}
for tokens in tokenized:
    for tok in tokens:
        if tok not in vocab:
            vocab[tok] = len(vocab)
print("어휘 크기:", len(vocab))
print(vocab)

# 3) 정수 시퀀스로 변환하고, 가장 긴 문장에 맞춰 0으로 패딩
ids = [torch.tensor([vocab[tok] for tok in tokens]) for tokens in tokenized]
lengths = torch.tensor([len(t) for t in ids])
batch = pad_sequence(ids, batch_first=True, padding_value=0)

print("batch shape:", tuple(batch.shape))
print(batch)
print("lengths:", lengths.tolist())
```

```
['this', 'movie', 'is', 'boring']
어휘 크기: 10
{'<pad>': 0, 'i': 1, 'love': 2, 'this': 3, 'movie': 4, 'is': 5, 'boring': 6, 'it': 7, 'plot': 8, 'but': 9}
batch shape: (4, 8)
tensor([[1, 2, 3, 4, 0, 0, 0, 0],
        [3, 4, 5, 6, 0, 0, 0, 0],
        [1, 2, 7, 0, 0, 0, 0, 0],
        [6, 6, 6, 8, 9, 1, 2, 7]])
lengths: [4, 4, 3, 8]
```

네 문장이 `(4, 8)` 크기의 정수 행렬 하나로 묶였다. 0은 "내용 없음"을 뜻하는 패딩 값이므로, 실제 길이 정보를 `lengths`에 따로 보관해 두었다. `pad_sequence`는 `torch.nn.utils.rnn`에 있는 함수로, 길이가 다른 텐서 리스트를 가장 긴 것에 맞춰 채워 준다.

실제 프로젝트에서는 공백 기준 토큰화가 늘 적절하지는 않다. 한국어는 "영화를", "영화가", "영화는"처럼 조사가 붙어 같은 단어가 여러 모양으로 나타나기 때문에 형태소 분석기를 쓰거나, 단어를 더 작은 조각으로 쪼개는 서브워드(subword) 방식(BPE 등)을 사용한다. 이 책에서는 구조를 이해하는 것이 목적이므로 공백 기준으로 충분하다.

### 16.6.2 `nn.Embedding`

정수 ID를 그대로 모델에 넣으면 "ID 4가 ID 2의 두 배"처럼 숫자의 크기에 아무 근거 없는 의미가 생긴다. 이를 피하는 한 방법은 원-핫 인코딩이지만, 어휘가 수만 개라면 벡터가 너무 크고 희소해진다. 임베딩은 어휘의 각 토큰에 짧고 촘촘한 실수 벡터를 하나씩 대응시킨 표(룩업 테이블)다. 이 벡터들은 모델의 파라미터이므로 학습 과정에서 함께 조정되고, 비슷하게 쓰이는 단어는 비슷한 벡터를 갖게 되는 경향이 있다.

```python
torch.manual_seed(0)
embedding = nn.Embedding(num_embeddings=len(vocab), embedding_dim=4, padding_idx=0)

print("embedding.weight shape:", tuple(embedding.weight.shape))
vecs = embedding(batch)
print("embedded shape:", tuple(vecs.shape))
print("pad 벡터:", embedding.weight[0].detach())
```

```
embedding.weight shape: (10, 4)
embedded shape: (4, 8, 4)
pad 벡터: tensor([0., 0., 0., 0.])
```

`nn.Embedding`의 가중치는 `(어휘 크기, 임베딩 차원)`이고, `(N, L)` 크기의 정수 텐서를 넣으면 `(N, L, 임베딩 차원)`의 실수 텐서가 나온다. 이 shape은 앞에서 본 RNN의 입력 `(N, L, F)`와 정확히 일치한다. 임베딩 차원이 곧 RNN 입장에서의 "시점별 특성 수 `F`"다. `padding_idx=0`을 지정하면 0번 행은 0 벡터로 고정되고 학습 중에도 갱신되지 않는다.

임베딩을 처음부터 학습하는 대신 대량의 텍스트로 미리 학습된 단어 벡터(Word2Vec, GloVe 등)를 초기값으로 가져다 쓰는 방법도 있다. 이 책에서는 이름만 소개한다.

### 16.6.3 임베딩, LSTM, 선형층을 이어 붙이기

세 부품을 이어 붙이면 문장 분류 모델의 뼈대가 완성된다. 이때 패딩 때문에 한 가지 주의할 점이 있다. 짧은 문장은 뒤에 패딩이 붙어 있어서, 그냥 마지막 시점의 은닉 상태(`out[:, -1]` 또는 `h_n`)를 쓰면 "패딩 토큰을 읽은 뒤의 상태"가 되어 버린다. 그래서 `lengths`를 이용해 각 문장의 "진짜 마지막 시점"을 골라낸다.

```python
class TextClassifier(nn.Module):
    def __init__(self, vocab_size, emb_dim=8, hidden_size=16, num_classes=2):
        super().__init__()
        self.embedding = nn.Embedding(vocab_size, emb_dim, padding_idx=0)
        self.lstm = nn.LSTM(emb_dim, hidden_size, batch_first=True)
        self.fc = nn.Linear(hidden_size, num_classes)

    def forward(self, x, lengths):            # x: (N, L) 정수, lengths: (N,)
        e = self.embedding(x)                 # (N, L, emb_dim)
        out, _ = self.lstm(e)                 # (N, L, hidden)
        rows = torch.arange(x.size(0))
        last = out[rows, lengths - 1]         # 각 문장의 진짜 마지막 시점 (N, hidden)
        return self.fc(last)                  # (N, num_classes)

torch.manual_seed(0)
model = TextClassifier(len(vocab))
logits = model(batch, lengths)
print("logits shape:", tuple(logits.shape))

# 패딩의 영향이 없는지 확인: 3번째 문장(길이 3)을 패딩 없이 단독으로 넣어 본다
solo = model(batch[2:3, :3], torch.tensor([3]))
print("패딩 유무와 무관하게 같은 결과:", torch.allclose(solo, logits[2:3], atol=1e-6))
```

```
logits shape: (4, 2)
패딩 유무와 무관하게 같은 결과: True
```

`out[rows, lengths - 1]`은 "i번째 문장에서 `lengths[i] - 1`번째 시점의 은닉 상태"를 한 번에 골라 주는 인덱싱이다. 이렇게 하면 패딩 뒤에 있던 시점은 결과에 영향을 주지 않는다. LSTM은 앞에서 뒤로 읽는 단방향이므로, 어떤 시점의 출력은 그 시점 이전의 입력에만 의존하기 때문이다. PyTorch에는 이 문제를 더 정식으로 다루는 `pack_padded_sequence`도 있지만, 이 책에서는 이름만 언급하고 넘어간다. 여기서 만든 모델은 구조만 보인 것이며, 실제 분류기로 쓰려면 라벨이 있는 충분한 양의 문장 데이터로 학습해야 한다.

## 16.7 이 책의 범위 너머: 트랜스포머와 LLM

RNN은 긴 시퀀스에서 앞쪽 정보를 붙잡기 어렵고, 시점을 하나씩 순서대로 계산해야 해서 병렬 처리가 어렵다는 한계가 있다. 이 문제를 어텐션(attention)이라는 다른 방식으로 푼 구조가 트랜스포머(Transformer)이며, 오늘날의 대규모 언어 모델(LLM, Large Language Model)은 대부분 트랜스포머를 기반으로 한다.

이 주제들은 이 책의 범위를 벗어나며, 따로 공부해야 할 별도의 학습 주제다. 이 책의 13~16장에서 익힌 텐서, `nn.Module`, 학습 루프, 임베딩, 시퀀스 shape에 대한 감각이 그 출발점이 된다. 관심이 있다면 PyTorch의 `nn.MultiheadAttention`, `nn.Transformer` 모듈의 문서와, 사전 학습된 언어 모델을 쉽게 불러 쓸 수 있게 해 주는 Hugging Face의 Transformers 라이브러리부터 살펴보면 된다.

## 요약

- 시퀀스 데이터(텍스트, 시계열 등)는 순서가 정보이고, 길이가 일정하지 않으며, 앞선 문맥이 뒤의 해석을 좌우한다. Bag of Words는 순서를 버리므로 "dog bites man"과 "man bites dog"를 구별하지 못한다.
- 시계열은 슬라이딩 윈도우로 `(샘플 수, 윈도우 길이, 특성 수)` 형태의 지도학습 데이터로 바꿀 수 있다. 시계열을 분할할 때는 섞지 않고 시간 순서를 지킨다.
- RNN은 `h_t = tanh(W_x · x_t + W_h · h_(t-1) + b)`처럼 은닉 상태를 다음 시점으로 넘기며 시퀀스를 읽고, 모든 시점에서 같은 가중치를 재사용한다.
- 시간을 거슬러 역전파할 때 같은 행렬이 반복해서 곱해지므로 기울기가 소실되거나 폭발한다. 기본 RNN은 먼 과거의 정보를 배우기 어렵고, 기울기 폭발에는 기울기 클리핑을 쓴다.
- LSTM은 셀 상태와 세 개의 게이트(망각, 입력, 출력)로, GRU는 두 개의 게이트로 기억을 조절해 이 문제를 완화한다.
- `batch_first=True`일 때 RNN 계열의 입력은 `(N, L, F)`, `out`은 `(N, L, H)`(양방향이면 `2H`)이고, `h_n`과 `c_n`은 `batch_first`와 상관없이 `(층 수 × 방향 수, N, H)`다. LSTM은 `(out, (h_n, c_n))`을 반환한다.
- 사인파 예측 실습에서 LSTM은 "직전 값 복사" 기준선보다 낮은 MSE를 보였다. 복잡한 모델은 항상 단순한 기준선과 비교해서 평가한다.
- 텍스트는 토큰화 → 어휘 사전 → 정수 ID → 패딩 → `nn.Embedding` 순서로 `(N, L, 임베딩 차원)` 텐서가 되어 RNN에 들어간다. 패딩된 배치에서는 문장마다 진짜 마지막 시점을 골라야 한다.
- 트랜스포머, 어텐션, LLM은 이 책의 범위를 벗어난 별도의 학습 주제다.

## 연습문제

1. 슬라이딩 윈도우로 만든 시계열 데이터를 `train_test_split`으로 무작위로 섞어 분할하면 왜 평가 결과가 실제보다 좋게 나올 수 있는지 설명하라.
2. `nn.LSTM(input_size=6, hidden_size=10, num_layers=3, bidirectional=True, batch_first=True)`에 `(16, 25, 6)` 크기의 입력을 넣었을 때 `out`, `h_n`, `c_n`의 shape을 각각 쓰고, 그렇게 되는 이유를 설명하라.
3. 16.3.1절의 기울기 소실 실험에서 `W_h`의 초기화 크기를 `1.0 / np.sqrt(H)`에서 `3.0 / np.sqrt(H)`로 바꿔 실행해 보고, `||dh_t/dh_0||`가 어떻게 변하는지 관찰하라. 결과가 소실, 폭발 중 어느 쪽에 가까운지, `tanh`의 미분값 `1 - h^2`이 결과에 어떤 영향을 주는지 함께 서술하라.
4. 16.5절의 `SeqRegressor`에서 `window`를 5로 줄이거나 60으로 늘려 학습해 보고, 테스트 MSE와 30걸음 자기회귀 예측 MSE가 어떻게 달라지는지 관찰하라. 사인파의 주기(약 62.8 시점)와 연결지어 이유를 추측해 보라.
5. 16.6.3절의 `TextClassifier.forward`에서 `out[rows, lengths - 1]` 대신 `out[:, -1]`을 쓰면 짧은 문장의 결과가 패딩 유무에 따라 달라지는지 직접 실험해 확인하라. 달라진다면 왜 그런지 설명하라.

---

📖 [◀ 목차](00-목차.md) | [◀ 이전: 15장. 합성곱 신경망(CNN)과 이미지 분류](ch15-합성곱신경망CNN.md) | [다음: 17장. ML 파이프라인과 모델 저장·배포 ▶](ch17-ML파이프라인과모델배포.md)
