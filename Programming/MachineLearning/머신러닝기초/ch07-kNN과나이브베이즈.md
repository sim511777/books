# 7장. k-NN과 나이브 베이즈

📖 [◀ 목차](00-목차.md) | [◀ 이전: 6장. 데이터 전처리와 특성 공학](ch06-데이터전처리와특성공학.md) | [다음: 8장. 서포트 벡터 머신 ▶](ch08-서포트벡터머신.md)

---

이번 장에서는 가장 단순하면서도 쓸모가 큰 분류 알고리즘 두 가지를 만납니다. 하나는 "비슷한 것끼리 같은 종류일 것"이라는 상식을 그대로 알고리즘으로 옮긴 **k-최근접 이웃(k-Nearest Neighbors, k-NN)** 이고, 다른 하나는 확률로 분류 문제를 푸는 **나이브 베이즈(Naive Bayes)** 입니다. 둘 다 몇 줄이면 구현할 수 있고, 데이터가 많지 않을 때나 첫 번째 기준선(baseline) 모델이 필요할 때 훌륭한 출발점이 됩니다.

이 장을 마치면 다음을 할 수 있습니다.

- k-NN이 예측을 만드는 과정을 NumPy로 직접 구현하고 `KNeighborsClassifier`와 비교한다.
- 거리 척도, `k`, 스케일링, 차원의 저주가 k-NN에 미치는 영향을 실험으로 확인한다.
- 베이즈 정리와 "나이브한" 독립 가정이 무엇인지 설명하고, `GaussianNB`와 `MultinomialNB`를 사용한다.
- `CountVectorizer`와 `MultinomialNB`로 간단한 스팸 분류기를 만든다.

## 7.1 k-최근접 이웃: 이웃을 보면 정체를 안다

새로 이사 온 동네에서 어떤 가게가 카페인지 식당인지 모르겠다고 해 봅시다. 그 가게 바로 옆 다섯 곳을 살펴보니 네 곳이 카페였다면, 그 가게도 카페일 가능성이 높다고 짐작할 수 있습니다. k-NN은 정확히 이렇게 동작합니다.

1. 예측하고 싶은 새 샘플과 **모든 훈련 샘플 사이의 거리**를 계산한다.
2. 거리가 가까운 순서대로 **k개**를 고른다.
3. 그 k개의 레이블 중 **가장 많은 클래스**(다수결)를 예측값으로 삼는다.

이 알고리즘에는 "학습"이라고 부를 만한 단계가 사실상 없습니다. `fit`은 훈련 데이터를 저장해 둘 뿐이고, 계산은 모두 예측할 때 일어납니다. 그래서 k-NN을 **게으른 학습(lazy learning)** 또는 **사례 기반 학습(instance-based learning)** 이라고도 부릅니다. 지금까지 만난 선형 모델이 데이터에서 가중치를 "압축해서" 배운 뒤 데이터를 버리는 것과 대조됩니다.

### 거리 척도

"가깝다"를 어떻게 정의하느냐가 곧 k-NN의 성격을 결정합니다. 가장 흔한 두 가지는 다음과 같습니다.

- **유클리드 거리**: 두 점을 잇는 직선 거리입니다. 각 특성의 차이를 제곱해 더한 뒤 제곱근을 취합니다.
- **맨해튼 거리**: 격자 모양 도시에서 블록을 따라 걷는 거리입니다. 각 특성의 차이의 절댓값을 더합니다.

두 거리는 모두 **민코프스키 거리**의 특수한 경우입니다. 차이의 `p`제곱을 더한 뒤 `1/p`제곱을 하는 공식에서 `p=2`이면 유클리드, `p=1`이면 맨해튼이 됩니다. 점 두 개로 직접 계산해 보겠습니다.

```python
import numpy as np

a = np.array([1.0, 2.0])
b = np.array([4.0, 6.0])

euclidean = np.sqrt(np.sum((a - b) ** 2))
manhattan = np.sum(np.abs(a - b))
print("유클리드 거리:", euclidean)
print("맨해튼 거리  :", manhattan)
```

```text
유클리드 거리: 5.0
맨해튼 거리  : 7.0
```

(1, 2)와 (4, 6)은 가로로 3, 세로로 4만큼 떨어져 있으므로 직선 거리는 5, 격자를 따라 걸으면 7입니다. 맨해튼 거리는 이상치 하나가 만드는 큰 차이를 제곱하지 않기 때문에 이상치에 덜 민감하고, 특성이 매우 많을 때 유클리드 거리보다 안정적인 경우가 있습니다. 그렇다고 어느 쪽이 항상 낫다고 말할 수는 없으므로, 실제 데이터에서는 두 가지를 모두 시험해 보는 것이 좋습니다.

## 7.2 NumPy로 k-NN 직접 구현하기

위의 세 단계(거리 계산 → 정렬 → 다수결)를 그대로 코드로 옮겨 봅시다. 데이터는 붓꽃(iris) 데이터를 사용하고, 클래스 비율을 유지하도록 `stratify=y`를 지정해 훈련/테스트로 나눕니다.

```python
import numpy as np
from sklearn.datasets import load_iris
from sklearn.model_selection import train_test_split

X, y = load_iris(return_X_y=True)
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.3, stratify=y, random_state=42
)


class MyKNN:
    """NumPy만으로 만든 k-최근접 이웃 분류기."""

    def __init__(self, k=5, p=2):
        self.k = k
        self.p = p  # p=1: 맨해튼 거리, p=2: 유클리드 거리

    def fit(self, X, y):
        # k-NN의 "학습"은 데이터를 그대로 저장하는 것이 전부다.
        self.X_ = np.asarray(X, dtype=float)
        self.y_ = np.asarray(y)
        return self

    def _distances(self, x):
        # 새 샘플 x와 저장된 모든 훈련 샘플 사이의 거리
        diff = np.abs(self.X_ - x)
        return np.sum(diff ** self.p, axis=1) ** (1 / self.p)

    def predict(self, X):
        preds = []
        for x in np.asarray(X, dtype=float):
            dist = self._distances(x)                 # 1) 거리 계산
            nearest = np.argsort(dist)[: self.k]      # 2) 가까운 순으로 정렬해 k개 선택
            votes = np.bincount(self.y_[nearest])     # 3) 클래스별 득표 수 세기
            preds.append(np.argmax(votes))            #    다수결
        return np.array(preds)

    def score(self, X, y):
        return np.mean(self.predict(X) == np.asarray(y))


my_knn = MyKNN(k=5).fit(X_train, y_train)
print("직접 구현 정확도:", round(my_knn.score(X_test, y_test), 4))
```

```text
직접 구현 정확도: 0.9778
```

핵심은 `predict` 안의 세 줄입니다.

- `_distances`: `self.X_ - x`는 NumPy의 **브로드캐스팅** 덕분에 훈련 샘플 전체에 대한 차이를 한 번에 계산합니다. 여기에 `p`제곱, 합, `1/p`제곱을 차례로 적용하면 거리 벡터가 나옵니다.
- `np.argsort(dist)[: self.k]`: 값 자체가 아니라 **정렬된 순서의 인덱스**를 얻어, 가까운 k개 샘플의 위치를 구합니다.
- `np.bincount(...)`와 `np.argmax(...)`: 이웃 레이블이 각각 몇 번 나왔는지 센 뒤 최댓값의 위치(=클래스 번호)를 고릅니다. 득표 수가 같으면 번호가 작은 클래스가 선택됩니다.

테스트 샘플마다 파이썬 `for` 문을 도는 것이 마음에 들지 않는다면, 거리 계산을 통째로 행렬 연산으로 바꿀 수 있습니다.

```python
# 반복문 없이 모든 (테스트, 훈련) 쌍의 거리를 한 번에 계산 (브로드캐스팅)
diff = X_test[:, None, :] - X_train[None, :, :]   # (45, 1, 4) - (1, 105, 4) -> (45, 105, 4)
dist_matrix = np.sqrt((diff ** 2).sum(axis=2))     # (45, 105)
print("거리 행렬 모양:", dist_matrix.shape)

nearest = np.argsort(dist_matrix, axis=1)[:, :5]   # 각 테스트 샘플의 최근접 이웃 5개 인덱스
pred_fast = np.array([np.argmax(np.bincount(y_train[idx])) for idx in nearest])
print("반복문 버전과 같은가?", np.array_equal(pred_fast, my_knn.predict(X_test)))
```

```text
거리 행렬 모양: (45, 105)
반복문 버전과 같은가? True
```

`X_test[:, None, :]`처럼 축을 하나 끼워 넣으면 (45, 1, 4)와 (1, 105, 4) 배열의 뺄셈이 (45, 105, 4)로 브로드캐스팅되어, 모든 (테스트, 훈련) 쌍의 차이가 한꺼번에 계산됩니다. 이 방식은 빠르지만 중간 배열의 크기가 `테스트 수 × 훈련 수 × 특성 수`로 커지므로, 데이터가 크면 메모리를 많이 쓴다는 점에 주의해야 합니다.

### scikit-learn과 비교하기

이제 `KNeighborsClassifier`로 같은 일을 하고 결과를 비교합니다.

```python
from sklearn.neighbors import KNeighborsClassifier

sk_knn = KNeighborsClassifier(n_neighbors=5)  # 기본 거리: 유클리드(minkowski, p=2)
sk_knn.fit(X_train, y_train)

print("scikit-learn 정확도:", round(sk_knn.score(X_test, y_test), 4))
print("예측이 모두 같은가?  ", np.array_equal(sk_knn.predict(X_test), my_knn.predict(X_test)))

# 맨해튼 거리도 같은 방식으로 비교
my_l1 = MyKNN(k=5, p=1).fit(X_train, y_train)
sk_l1 = KNeighborsClassifier(n_neighbors=5, metric="manhattan").fit(X_train, y_train)
print("맨해튼 거리 예측이 같은가?", np.array_equal(sk_l1.predict(X_test), my_l1.predict(X_test)))
```

```text
scikit-learn 정확도: 0.9778
예측이 모두 같은가?   True
맨해튼 거리 예측이 같은가? True
```

직접 만든 구현과 scikit-learn의 예측이 모든 테스트 샘플에서 일치합니다. 거리 척도를 `metric="manhattan"`으로 바꾼 경우에도 마찬가지입니다. k-NN이 어려운 알고리즘이어서가 아니라, 정의가 너무 명확해서 구현마다 다른 답이 나올 여지가 거의 없기 때문입니다. 물론 scikit-learn은 훈련 데이터를 KD 트리나 볼 트리 같은 자료구조에 담아 가까운 이웃을 훨씬 빠르게 찾아 주므로, 실전에서는 직접 구현보다 훨씬 효율적입니다. `algorithm="auto"`(기본값)가 데이터에 맞는 방식을 알아서 고릅니다.

`predict_proba`와 `kneighbors`도 유용합니다.

```python
# 이웃 5명 중 각 클래스의 비율이 곧 predict_proba
print(sk_knn.predict_proba(X_test[:5]))
print("이웃 인덱스:", sk_knn.kneighbors(X_test[:1], return_distance=False))
```

```text
[[0.  0.  1. ]
 [0.  0.8 0.2]
 [0.  0.4 0.6]
 [0.  0.8 0.2]
 [0.  0.4 0.6]]
이웃 인덱스: [[88 15 22  9 82]]
```

k-NN의 `predict_proba`는 "이웃 k명 중 각 클래스가 차지하는 비율"입니다. 그래서 `k=5`일 때는 0, 0.2, 0.4, ... 같은 거친 값만 나옵니다. 또 `kneighbors`로 어떤 훈련 샘플이 이웃으로 뽑혔는지 직접 확인할 수 있는데, 이는 "왜 이렇게 예측했는가"를 설명하기 쉽다는 k-NN의 장점이기도 합니다.

거리가 가까운 이웃의 의견을 더 존중하고 싶다면 `weights="distance"`를 지정합니다. 이웃의 표를 거리의 역수만큼 가중해서 세기 때문에, 먼 이웃 여러 명보다 아주 가까운 이웃 한 명이 더 큰 영향력을 갖게 됩니다.

## 7.3 k 선택과 편향-분산

k-NN에서 가장 중요한 하이퍼파라미터는 이웃의 수 `k`입니다.

- **k가 매우 작을 때(예: 1)**: 예측이 가장 가까운 점 하나에 좌우됩니다. 훈련 데이터의 잡음이나 이상치까지 그대로 따라가므로 결정 경계가 들쭉날쭉해집니다. 분산이 크고 편향이 작은 **과대적합** 상태입니다.
- **k가 매우 클 때(예: 훈련 샘플 수에 가까움)**: 멀리 있는 점까지 투표에 참여하므로 결정 경계가 지나치게 매끄러워지고, 결국 가장 흔한 클래스만 계속 예측하게 됩니다. 편향이 크고 분산이 작은 **과소적합** 상태입니다.

잡음이 섞인 반달 모양 데이터(`make_moons`)에서 k를 바꿔 가며 확인해 봅시다. 교차 검증(cross-validation)은 훈련 데이터를 여러 조각으로 나눠 번갈아 검증용으로 쓰는 방법으로, 테스트 데이터를 건드리지 않고 일반화 성능을 추정하게 해 줍니다.

```python
from sklearn.datasets import make_moons
from sklearn.model_selection import cross_val_score

Xm, ym = make_moons(n_samples=400, noise=0.35, random_state=0)
Xm_tr, Xm_te, ym_tr, ym_te = train_test_split(
    Xm, ym, test_size=0.3, stratify=ym, random_state=0
)

print(" k | 훈련 정확도 | 테스트 정확도 | 5-겹 교차검증")
for k in [1, 3, 5, 9, 15, 25, 51, 101, 199]:
    knn = KNeighborsClassifier(n_neighbors=k).fit(Xm_tr, ym_tr)
    cv = cross_val_score(KNeighborsClassifier(n_neighbors=k), Xm_tr, ym_tr, cv=5).mean()
    print(f"{k:3d} | {knn.score(Xm_tr, ym_tr):11.3f} | {knn.score(Xm_te, ym_te):13.3f} | {cv:13.3f}")
```

```text
 k | 훈련 정확도 | 테스트 정확도 | 5-겹 교차검증
  1 |       1.000 |         0.833 |         0.832
  3 |       0.911 |         0.875 |         0.846
  5 |       0.904 |         0.892 |         0.857
  9 |       0.896 |         0.892 |         0.868
 15 |       0.893 |         0.875 |         0.879
 25 |       0.882 |         0.867 |         0.879
 51 |       0.875 |         0.883 |         0.861
101 |       0.857 |         0.875 |         0.839
199 |       0.764 |         0.775 |         0.707
```

표에서 세 가지를 읽을 수 있습니다.

1. `k=1`의 훈련 정확도는 정확히 1.000입니다. 훈련 샘플 자신이 자기와 가장 가까운 이웃이기 때문에 항상 정답을 맞힙니다. 그러나 테스트 정확도는 크게 낮아 전형적인 과대적합입니다.
2. `k`를 키우면 훈련 정확도는 꾸준히 내려가는 반면, 교차 검증 점수는 올랐다가 `k=15`~`25` 부근에서 최고(0.879)를 찍고 다시 내려갑니다. 훈련 샘플이 280개인데 `k=199`가 되면 점수가 급락해 과소적합을 보여 줍니다.
3. 테스트 정확도는 `k=5`와 `9`에서 가장 높지만, 테스트 샘플이 120개뿐이라 한 샘플이 약 0.8%p를 좌우합니다. 이런 작은 차이로 k를 고르면 우연에 속기 쉬우므로 **k는 교차 검증 점수를 기준으로 고르는 것**이 안전합니다. 이 방법은 `GridSearchCV`로 자동화할 수 있습니다.

이진 분류에서는 동률 투표를 피하기 위해 홀수 k를 쓰는 것이 관례입니다. 다만 3개 이상의 클래스에서는 홀수여도 동률이 생길 수 있습니다.

## 7.4 스케일링은 필수다

k-NN은 오직 거리로 판단합니다. 따라서 특성마다 값의 크기(단위)가 다르면 **값이 큰 특성이 거리를 독차지**합니다. 와인 데이터로 확인해 봅시다.

```python
from sklearn.datasets import load_wine

Xw, yw = load_wine(return_X_y=True)
wine = load_wine()
for name, col in zip(wine.feature_names[:3] + ["proline"], [0, 1, 2, 12]):
    print(f"{name:12s} 범위: {Xw[:, col].min():8.2f} ~ {Xw[:, col].max():8.2f}")
```

```text
alcohol      범위:    11.03 ~    14.83
malic_acid   범위:     0.74 ~     5.80
ash          범위:     1.36 ~     3.23
proline      범위:   278.00 ~  1680.00
```

`proline`은 수백~천 단위인 반면 `alcohol`은 10 안팎이고 `ash`는 1~3 정도입니다. 이 데이터에 유클리드 거리를 그대로 쓰면 사실상 `proline` 하나로만 이웃을 고르는 셈입니다. 표준화(`StandardScaler`: 평균 0, 표준편차 1로 변환)를 적용한 경우와 비교해 봅시다. 스케일러는 반드시 **훈련 데이터로만 학습**해야 하므로, 교차 검증과 함께 쓸 때는 파이프라인으로 묶습니다.

```python
from sklearn.pipeline import make_pipeline
from sklearn.preprocessing import StandardScaler

raw = KNeighborsClassifier(n_neighbors=5)
scaled = make_pipeline(StandardScaler(), KNeighborsClassifier(n_neighbors=5))

print("스케일링 안 함:", cross_val_score(raw, Xw, yw, cv=5).mean().round(3))
print("StandardScaler:", cross_val_score(scaled, Xw, yw, cv=5).mean().round(3))
```

```text
스케일링 안 함: 0.691
StandardScaler: 0.949
```

스케일링만으로 정확도가 0.69에서 0.95로 뛰었습니다. 알고리즘이나 하이퍼파라미터를 바꾸지 않고 얻은 개선치고는 극적입니다. 거리 기반 알고리즘(k-NN, 8장의 SVM, K-평균 군집화 등)을 쓸 때는 스케일링을 습관처럼 먼저 적용하세요.

## 7.5 차원의 저주

특성 수(차원)가 많아질수록 k-NN은 점점 힘을 잃습니다. 이를 **차원의 저주(curse of dimensionality)** 라고 부릅니다. 직관은 이렇습니다. 고차원 공간은 너무 넓어서, 데이터가 아무리 많아도 점들이 서로 멀리 흩어집니다. 게다가 모든 점이 서로 "비슷하게 멀어져서" 가장 가까운 점과 가장 먼 점의 거리 차이가 별로 나지 않게 됩니다. "가깝다"는 개념이 의미를 잃는 것입니다.

균일하게 뽑은 점 1000개에서 임의의 질의점까지의 거리를 재서, 가장 가까운 점과 가장 먼 점의 거리 비율을 차원별로 확인해 봅시다.

```python
rng = np.random.default_rng(0)

print("차원 | 가장 가까운 점까지 / 가장 먼 점까지의 거리 비율")
for d in [1, 2, 10, 100, 1000]:
    points = rng.random((1000, d))            # [0, 1)^d 안의 균일한 점 1000개
    query = rng.random(d)
    dist = np.sqrt(((points - query) ** 2).sum(axis=1))
    print(f"{d:4d} | {dist.min() / dist.max():.3f}")
```

```text
차원 | 가장 가까운 점까지 / 가장 먼 점까지의 거리 비율
   1 | 0.000
   2 | 0.015
  10 | 0.233
 100 | 0.698
1000 | 0.895
```

1차원에서는 가장 가까운 점이 가장 먼 점보다 훨씬 가깝지만(비율이 0에 가깝습니다), 1000차원에서는 거의 0.9, 즉 가장 가까운 점도 가장 먼 점과 거의 같은 거리에 있습니다. 이 상태에서는 "k개의 가까운 이웃"이 사실상 아무 정보도 주지 못합니다.

실전에서 더 흔한 문제는 **쓸모없는 특성이 섞이는 것**입니다. 붓꽃 데이터에 클래스와 무관한 잡음 특성을 점점 더 붙여 보겠습니다.

```python
# 붓꽃 데이터에 의미 없는 잡음 특성을 추가하면?
rng = np.random.default_rng(0)

print("잡음 특성 수 | 테스트 정확도")
for n_noise in [0, 5, 20, 100, 500]:
    noise_tr = rng.normal(size=(len(X_train), n_noise))
    noise_te = rng.normal(size=(len(X_test), n_noise))
    Xtr = np.hstack([X_train, noise_tr])
    Xte = np.hstack([X_test, noise_te])
    acc = KNeighborsClassifier(n_neighbors=5).fit(Xtr, y_train).score(Xte, y_test)
    print(f"{n_noise:12d} | {acc:.3f}")
```

```text
잡음 특성 수 | 테스트 정확도
           0 | 0.978
           5 | 0.867
          20 | 0.778
         100 | 0.667
         500 | 0.533
```

핵심 정보를 담은 특성은 4개 그대로인데, 잡음 특성이 늘어날수록 정확도가 0.98에서 거의 동전 던지기 수준까지 떨어집니다. 모든 특성이 거리에 똑같이 기여하기 때문에 잡음이 신호를 덮어 버리는 것입니다. 대응책은 다음과 같습니다.

- 의미 있는 특성만 남기는 **특성 선택**
- PCA 같은 **차원 축소**
- 특성 수에 둔감한 다른 알고리즘(트리 기반 모델 등)의 사용

## 7.6 나이브 베이즈: 확률로 분류하기

### 베이즈 정리의 직관

나이브 베이즈는 "증거를 본 뒤에 믿음을 고친다"는 베이즈 정리에서 출발합니다.

```text
                  P(증거 | 클래스) × P(클래스)
P(클래스 | 증거) = ------------------------------
                          P(증거)
```

각 항의 의미는 다음과 같습니다.

- `P(클래스)`: **사전확률**. 증거를 보기 전의 믿음입니다. 예를 들어 전체 메일의 40%가 스팸이라면 0.4입니다.
- `P(증거 | 클래스)`: **가능도**. 그 클래스라면 이런 증거가 얼마나 흔히 나타나는가입니다.
- `P(클래스 | 증거)`: **사후확률**. 증거를 본 뒤 갱신된 믿음이며, 우리가 구하려는 값입니다.

숫자로 확인해 봅시다. 메일의 40%가 스팸이고, 스팸의 50%가 "무료"라는 단어를 포함하며, 정상 메일에서는 5%만 "무료"를 포함한다고 합시다. 어떤 메일에 "무료"가 들어 있다면 스팸일 확률은 얼마일까요?

```python
# P(스팸) = 0.4, P("무료" | 스팸) = 0.5, P("무료" | 정상) = 0.05
p_spam = 0.4
p_word_given_spam = 0.5
p_word_given_ham = 0.05

evidence = p_word_given_spam * p_spam + p_word_given_ham * (1 - p_spam)  # P("무료")
posterior = p_word_given_spam * p_spam / evidence                       # P(스팸 | "무료")
print(f"P(스팸 | '무료') = {posterior:.3f}")
```

```text
P(스팸 | '무료') = 0.870
```

스팸 메일 비율은 40%였지만 "무료"라는 증거를 보고 나자 87%로 올랐습니다. 분류기는 이렇게 클래스마다 사후확률을 계산한 뒤 가장 큰 클래스를 고릅니다. 클래스를 비교할 때는 모든 클래스에 공통인 분모 `P(증거)`가 필요 없으므로, 분자만 계산해서 비교해도 결과는 같습니다.

### "나이브"한 독립 가정

문제는 증거가 단어 하나가 아니라 **특성 수십~수천 개의 조합**이라는 점입니다. "무료", "쿠폰", "지금"이 동시에 등장할 확률을 정확히 추정하려면, 이 세 단어의 모든 조합이 데이터에 충분히 나타나야 합니다. 특성이 늘어날수록 현실적으로 불가능해집니다.

나이브 베이즈는 과감한 단순화로 이 문제를 풉니다. **클래스가 정해졌다면 특성들은 서로 독립이다**라고 가정하는 것입니다. 그러면 결합 확률이 각 특성의 확률의 곱으로 쪼개집니다.

```text
P(x1, x2, ..., xn | 클래스) = P(x1 | 클래스) × P(x2 | 클래스) × ... × P(xn | 클래스)
```

현실에서 특성들이 정말로 독립인 경우는 드뭅니다. 문서에서 "쿠폰"이 나오면 "무료"도 나올 가능성이 높은 것처럼 말이죠. 그래서 "순진한(naive)"이라는 이름이 붙었습니다. 그런데도 이 알고리즘은 놀랄 만큼 잘 동작하는 경우가 많습니다. 분류에서는 확률의 **정확한 값**보다 **어느 클래스가 더 큰가**라는 순서만 맞으면 되기 때문입니다.

실제 구현에서는 작은 확률을 수천 번 곱하면 컴퓨터가 0으로 인식해 버리는(언더플로) 문제가 있어서, 로그를 취해 **곱셈을 덧셈으로** 바꿉니다. 이는 아래 코드에서 확인할 수 있습니다.

### GaussianNB: 연속형 특성

특성이 연속적인 숫자일 때는 "클래스별로, 각 특성은 정규분포를 따른다"고 가정하는 `GaussianNB`를 씁니다. 학습은 단순히 클래스별·특성별 평균과 분산을 계산하는 것이 전부입니다.

```python
from sklearn.naive_bayes import GaussianNB

gnb = GaussianNB().fit(X_train, y_train)
print("GaussianNB 테스트 정확도:", round(gnb.score(X_test, y_test), 4))
print("클래스 사전확률:", gnb.class_prior_.round(3))
print("클래스별 평균 (행=클래스, 열=특성):")
print(gnb.theta_.round(2))
print("클래스별 분산:")
print(gnb.var_.round(3))
```

```text
GaussianNB 테스트 정확도: 0.9111
클래스 사전확률: [0.333 0.333 0.333]
클래스별 평균 (행=클래스, 열=특성):
[[4.99 3.43 1.49 0.24]
 [5.95 2.73 4.24 1.31]
 [6.68 3.01 5.63 2.07]]
클래스별 분산:
[[0.103 0.174 0.023 0.009]
 [0.241 0.086 0.211 0.036]
 [0.425 0.117 0.323 0.064]]
```

`theta_`는 클래스별 평균, `var_`는 클래스별 분산, `class_prior_`는 사전확률입니다. 붓꽃 데이터는 클래스당 50개씩 균형이 맞아서 사전확률이 모두 1/3입니다. 평균 표를 보면 꽃잎 길이(세 번째 열)가 1.49, 4.24, 5.63으로 클래스마다 뚜렷하게 달라서 분류에 큰 도움이 될 것임을 짐작할 수 있습니다.

이제 `predict_proba`가 내부에서 하는 일을 직접 계산해 확인해 봅시다. 사전확률의 로그에, 특성별 로그 가능도를 **그냥 더하면** 됩니다.

```python
from scipy.stats import norm

x = X_test[3]   # 모델이 다소 망설이는 샘플
log_post = []
for c in range(3):
    # 특성별 로그 가능도를 "그냥 더한다" = 특성들이 독립이라는 나이브 가정
    log_lik = norm.logpdf(x, loc=gnb.theta_[c], scale=np.sqrt(gnb.var_[c])).sum()
    log_post.append(np.log(gnb.class_prior_[c]) + log_lik)

log_post = np.array(log_post)
post = np.exp(log_post - log_post.max())
post /= post.sum()                      # 정규화 -> 사후확률
print("직접 계산:", post.round(4))
print("predict_proba:", gnb.predict_proba(x.reshape(1, -1))[0].round(4))
```

```text
직접 계산: [0.    0.668 0.332]
predict_proba: [0.    0.668 0.332]
```

직접 계산한 값이 `predict_proba`와 일치합니다. 이 샘플은 두 번째와 세 번째 클래스 사이에서 모델이 다소 망설이는 경우입니다. 이 정도가 나이브 베이즈의 전부입니다. 평균과 분산 표, 그리고 로그 확률의 덧셈이 모델의 거의 전부이므로 학습도 예측도 매우 빠릅니다.

붓꽃에서 `GaussianNB`의 정확도(0.911)는 같은 데이터에서 k-NN(0.978)보다 낮습니다. 붓꽃의 꽃잎 길이와 폭은 서로 강하게 연관되어 있어 "독립" 가정이 잘 맞지 않기 때문으로 볼 수 있습니다. 같은 방법이 데이터에 따라 잘 맞을 수도, 덜 맞을 수도 있다는 점을 기억해 두세요.

### MultinomialNB: 텍스트와 개수 데이터

문서를 분류할 때 특성은 보통 "각 단어가 몇 번 나왔는가"입니다. 이런 **개수(count) 데이터**에는 `MultinomialNB`가 어울립니다. 클래스별로 "단어 w가 나올 확률"을 추정하는데, 개수에 라플라스 스무딩 `alpha`를 더해 계산합니다.

```
P(w | 클래스) = (그 클래스 문서들에서 w가 나온 횟수 + alpha) / (그 클래스의 전체 단어 수 + alpha × 어휘 수)
```

`alpha`가 없다면 훈련 중 스팸에서 한 번도 나오지 않은 단어가 하나만 포함되어도 스팸 확률이 통째로 0이 되어 버립니다(곱셈에 0이 끼어들기 때문입니다). `alpha`는 "모든 단어를 최소 `alpha`번은 본 것으로 치자"는 가정으로 이 문제를 막아 줍니다. 기본값은 1.0입니다.

#### 스팸 분류 예제

작은 예제 문장을 직접 만들어 사용합니다. 스팸 8문장, 정상 8문장입니다. 텍스트를 숫자로 바꾸는 일은 `CountVectorizer`가 합니다. 문장을 단어로 쪼개고, 어휘 사전을 만들고, 문장마다 단어가 몇 번 나왔는지 센 행렬을 돌려줍니다.

```python
spam = [
    "지금 무료 쿠폰 받아가세요",
    "당첨 축하합니다 무료 상품권 지급",
    "대출 승인 완료 지금 바로 신청하세요",
    "한정 특가 할인 지금 클릭하세요",
    "무료 체험 신청하면 상품권 증정",
    "긴급 당첨 확인 링크 클릭",
    "저금리 대출 상담 무료 신청",
    "이벤트 당첨 쿠폰 지금 확인",
]
ham = [
    "내일 회의 시간 오후 세 시로 변경됐어요",
    "점심 같이 먹을래요",
    "보고서 초안 메일로 보냈어요 확인 부탁해요",
    "주말에 등산 갈 사람 있어요",
    "오늘 저녁 회식 장소 공지합니다",
    "다음 주 프로젝트 일정 공유드립니다",
    "회의 자료 확인 부탁드립니다",
    "생일 축하해요 저녁에 만나요",
]
texts = spam + ham
labels = [1] * len(spam) + [0] * len(ham)   # 1: 스팸, 0: 정상

from sklearn.feature_extraction.text import CountVectorizer

vec = CountVectorizer()
X_counts = vec.fit_transform(texts)
print("행렬 모양:", X_counts.shape, "(문장 수, 어휘 수)")
print("처음 8개 어휘:", list(vec.get_feature_names_out()[:8]))
vocab = vec.get_feature_names_out()
print("첫 문장:", texts[0])
print("첫 문장의 단어 수:", {vocab[j]: int(X_counts[0, j]) for j in X_counts[0].nonzero()[1]})
```

```text
행렬 모양: (16, 61) (문장 수, 어휘 수)
처음 8개 어휘: ['같이', '공유드립니다', '공지합니다', '긴급', '내일', '다음', '당첨', '대출']
첫 문장: 지금 무료 쿠폰 받아가세요
첫 문장의 단어 수: {'지금': 1, '무료': 1, '쿠폰': 1, '받아가세요': 1}
```

16개의 문장에서 61개의 서로 다른 단어가 추출되었습니다. 각 문장은 61차원 개수 벡터가 되며, 대부분의 값이 0이므로 `CountVectorizer`는 희소 행렬(sparse matrix)로 저장합니다.

> **참고**: `CountVectorizer`는 기본적으로 공백과 구두점을 기준으로 단어를 나누고, 글자 수가 2 미만인 토큰("세", "시" 같은 한 글자)은 버립니다. 한국어는 "신청", "신청하세요"처럼 조사와 어미가 붙은 형태가 서로 다른 단어로 취급됩니다. 이 장의 예제는 원리를 보여 주기 위해 이 단순한 방식을 그대로 쓰지만, 실제 한국어 텍스트 분류에서는 보통 형태소 분석기(예: KoNLPy)로 먼저 어간을 추출합니다.

이제 분류기를 학습하고, 훈련에 쓰지 않은 새 문장으로 예측합니다. 새 문장은 반드시 같은 `vec`의 `transform`으로 변환해야 합니다. 훈련 때 보지 못한 단어는 어휘 사전에 없으므로 무시됩니다.

```python
from sklearn.naive_bayes import MultinomialNB

nb = MultinomialNB(alpha=1.0).fit(X_counts, labels)

tests = [
    "무료 쿠폰 지금 신청하세요",
    "내일 점심 회의 확인 부탁해요",
    "당첨 확인 부탁드립니다",
    "프로젝트 일정 공유 부탁드립니다",
    "축하합니다 상품권 당첨",
]
proba = nb.predict_proba(vec.transform(tests))
for sentence, p in zip(tests, proba):
    label = "스팸" if p[1] > 0.5 else "정상"
    print(f"{label} (스팸 확률 {p[1]:.3f})  <- {sentence}")
```

```text
스팸 (스팸 확률 0.992)  <- 무료 쿠폰 지금 신청하세요
정상 (스팸 확률 0.033)  <- 내일 점심 회의 확인 부탁해요
스팸 (스팸 확률 0.639)  <- 당첨 확인 부탁드립니다
정상 (스팸 확률 0.100)  <- 프로젝트 일정 공유 부탁드립니다
스팸 (스팸 확률 0.955)  <- 축하합니다 상품권 당첨
```

"무료 쿠폰 지금 신청하세요"와 "축하합니다 상품권 당첨"은 스팸으로, 회의나 일정이 들어간 문장은 정상으로 분류되었습니다. 세 번째 문장 "당첨 확인 부탁드립니다"는 스팸 확률이 0.639로 애매합니다. "당첨"은 스팸 문장에서만, "부탁드립니다"는 정상 문장에서만 나왔고 "확인"은 양쪽에 두 번씩 나왔기 때문에 증거가 서로 엇갈립니다. 분류기가 확률로 자신의 망설임을 표현해 준다는 점이 나이브 베이즈의 매력입니다. 다만 단어가 서로 독립이라는 가정 때문에, 이 확률값을 그대로 믿기보다는 "대략적인 순위 점수" 정도로 보는 편이 안전합니다.

모델이 무엇을 근거로 판단했는지도 들여다볼 수 있습니다. `feature_log_prob_`는 클래스별 `log P(단어 | 클래스)`입니다. 스팸 쪽에서 뺀 정상 쪽 값을 비교하면 어떤 단어가 어느 쪽을 강하게 시사하는지 알 수 있습니다.

```python
vocab = vec.get_feature_names_out()
log_ratio = nb.feature_log_prob_[1] - nb.feature_log_prob_[0]   # log P(w|스팸) - log P(w|정상)
order = np.argsort(log_ratio)
print("스팸 쪽 단어:", [(vocab[j], round(float(log_ratio[j]), 2)) for j in order[::-1][:4]])
print("정상 쪽 단어:", [(vocab[j], round(float(log_ratio[j]), 2)) for j in order[:4]])
```

```text
스팸 쪽 단어: [('지금', 1.57), ('무료', 1.57), ('당첨', 1.35), ('쿠폰', 1.06)]
정상 쪽 단어: [('회의', -1.14), ('같이', -0.73), ('공지합니다', -0.73), ('공유드립니다', -0.73)]
```

"지금", "무료", "당첨", "쿠폰"이 스팸의 신호로, "회의" 등이 정상 쪽 신호로 학습되었습니다. 정상 쪽 단어 중 `-0.73`인 단어들은 정상 문장에서 딱 한 번만 나온 단어들이라 값이 같고, 순서는 의미가 없습니다. 직접 만든 데이터가 작기 때문에 생기는 현상입니다.

마지막으로, 스무딩 공식이 정말로 위와 같은지 한 단어("무료")로 확인해 봅시다.

```python
# 라플라스 스무딩: P(w | 클래스) = (클래스 내 w의 횟수 + alpha) / (클래스 내 전체 단어 수 + alpha * 어휘 수)
counts = X_counts.toarray()
labels_arr = np.array(labels)
alpha = 1.0
j = list(vocab).index("무료")

spam_counts = counts[labels_arr == 1].sum(axis=0)
manual = (spam_counts[j] + alpha) / (spam_counts.sum() + alpha * len(vocab))
print("직접 계산한 P('무료' | 스팸):", round(manual, 4))
print("모델 내부 값               :", round(float(np.exp(nb.feature_log_prob_[1][j])), 4))
```

```text
직접 계산한 P('무료' | 스팸): 0.0495
모델 내부 값               : 0.0495
```

직접 계산한 값과 모델 내부 값이 일치합니다. 실전에서는 `CountVectorizer`와 분류기를 `make_pipeline`으로 묶으면 원문 문자열을 그대로 `fit`과 `predict`에 넣을 수 있어 편리합니다.

```python
from sklearn.pipeline import make_pipeline

clf = make_pipeline(CountVectorizer(), MultinomialNB())
clf.fit(texts, labels)                       # 원문 문자열을 그대로 넣는다
print(clf.predict(["무료 상품권 지금 클릭", "회의 자료 공유드립니다"]))
```

```text
[1 0]
```

`1`은 스팸, `0`은 정상입니다. 두 문장 모두 기대대로 분류되었습니다.

### 그 밖의 나이브 베이즈 변형

scikit-learn에는 데이터의 성격에 따라 고를 수 있는 여러 변형이 있습니다.

| 클래스 | 적합한 특성 |
| --- | --- |
| `GaussianNB` | 연속형 숫자 (정규분포 가정) |
| `MultinomialNB` | 단어 개수, 빈도 (TF-IDF 값도 흔히 사용) |
| `BernoulliNB` | 단어가 "있는가/없는가"(0/1) |
| `CategoricalNB` | 정수로 인코딩된 범주형 특성 |

## 7.7 k-NN과 나이브 베이즈 비교

같은 와인 데이터에서 두 알고리즘을 층화 5-겹 교차 검증으로 비교해 봅시다.

```python
from sklearn.model_selection import StratifiedKFold

cv = StratifiedKFold(n_splits=5, shuffle=True, random_state=0)
models = {
    "k-NN (스케일링 없음)": KNeighborsClassifier(n_neighbors=5),
    "k-NN (표준화)": make_pipeline(StandardScaler(), KNeighborsClassifier(n_neighbors=5)),
    "GaussianNB": GaussianNB(),
}
for name, model in models.items():
    scores = cross_val_score(model, Xw, yw, cv=cv)
    print(f"{name:18s} 평균 {scores.mean():.3f} (표준편차 {scores.std():.3f})")
```

```text
k-NN (스케일링 없음)     평균 0.663 (표준편차 0.036)
k-NN (표준화)         평균 0.961 (표준편차 0.013)
GaussianNB         평균 0.972 (표준편차 0.000)
```

(앞 절의 0.691, 0.949와 숫자가 조금 다른 것은 교차 검증의 분할 방식이 다르기 때문입니다. 이 절에서는 분할 전에 데이터를 섞었습니다.) 스케일링을 한 k-NN과 `GaussianNB`가 모두 높은 점수를 내고, 스케일링을 빼먹은 k-NN만 크게 뒤처집니다. 한편 `GaussianNB`는 스케일링이 아예 필요 없습니다. 특성별로 따로 평균과 분산을 계산해 정규분포로 다루기 때문에, 단위가 달라도 영향을 받지 않습니다.

두 알고리즘의 특징을 정리하면 다음과 같습니다.

| 항목 | k-NN | 나이브 베이즈 |
| --- | --- | --- |
| 접근 방식 | 가까운 사례의 다수결 | 확률(베이즈 정리 + 독립 가정) |
| 학습 | 데이터를 저장할 뿐 | 평균, 분산, 단어 빈도 계산 (매우 빠름) |
| 예측 속도 | 느림 (훈련 데이터 전체와 거리 계산) | 매우 빠름 |
| 주요 하이퍼파라미터 | `k`, 거리 척도, 가중치 | `alpha`, `var_smoothing` 등 거의 없음 |
| 스케일링 | 필수 | 불필요 (`GaussianNB`) |
| 고차원/희소 데이터 | 약함 (차원의 저주) | 강함 (텍스트 분류의 고전) |
| 결정 경계 | 매우 유연 (비선형) | 단순한 편 |
| 가정 | 비슷한 점은 같은 클래스 | 특성 간 조건부 독립 |
| 확률 출력 | 이웃 비율 (거침) | 있으나 과신하는 경향 |
| 해석 | 이웃을 직접 보여 줄 수 있음 | 단어별 기여도를 볼 수 있음 |

어느 알고리즘을 먼저 써 볼까요? 데이터가 작고 특성이 적으며 복잡한 경계가 예상된다면 k-NN이, 텍스트처럼 특성이 매우 많고 빠른 기준선이 필요하다면 나이브 베이즈가 좋은 첫 선택입니다. 둘 다 구현이 단순해서, 더 복잡한 모델을 시도하기 전에 "이 정도는 이미 나오는구나" 하는 기준을 세워 주는 역할을 해 줍니다.

## 요약

- k-NN은 새 샘플과 가장 가까운 k개의 훈련 샘플을 찾아 다수결로 분류합니다. 학습은 데이터 저장이 전부이고 계산은 예측 때 일어납니다.
- 거리는 유클리드(`p=2`)와 맨해튼(`p=1`)이 대표적이며, NumPy로 "거리 계산 → `argsort` → `bincount`/`argmax`" 세 단계만으로 구현할 수 있습니다. 직접 만든 구현은 `KNeighborsClassifier`와 같은 예측을 냅니다.
- `k`가 작으면 과대적합(분산↑), 크면 과소적합(편향↑)입니다. 교차 검증으로 `k`를 고르세요.
- k-NN은 거리에 의존하므로 스케일링이 필수이며(와인 데이터에서 0.69 → 0.95), 특성이 많거나 잡음 특성이 섞이면 성능이 급격히 떨어집니다(차원의 저주).
- 나이브 베이즈는 베이즈 정리에 "클래스가 주어지면 특성들은 서로 독립"이라는 가정을 더해, 확률을 특성별 확률의 곱(로그에서는 합)으로 계산합니다.
- `GaussianNB`는 연속형 특성, `MultinomialNB`는 단어 개수 같은 이산형 개수 특성에 씁니다. `alpha`는 한 번도 못 본 단어 때문에 확률이 0이 되는 것을 막는 스무딩입니다.
- `CountVectorizer` + `MultinomialNB` 파이프라인으로 몇 줄 만에 텍스트 분류기를 만들 수 있습니다.

## 연습문제

1. **거리 가중 투표.** `MyKNN`에 `weights="distance"`에 해당하는 기능을 추가해 보세요. 이웃 k개의 득표를 `1 / (거리 + 1e-9)`로 가중해서 더하도록 `predict`를 고치고(`np.bincount`의 `weights` 인자를 활용할 수 있습니다), `KNeighborsClassifier(weights="distance")`와 예측이 같은지 확인하세요.

2. **k와 스케일링 실험.** `sklearn.datasets.load_breast_cancer`로 유방암 데이터를 불러와 (a) 스케일링 없이, (b) `StandardScaler`를 적용해서 `k = 1, 3, 5, ..., 21`에 대해 5-겹 교차 검증 정확도를 구하세요. 어떤 `k`가 가장 좋은가요? 스케일링의 효과는 `k`에 따라 어떻게 달라지나요?

3. **스무딩 관찰.** 7.6절의 스팸 분류기에서 `alpha`를 `0.01`, `1`, `10`으로 바꿔 가며 "당첨 확인 부탁드립니다"의 스팸 확률을 비교하세요. `alpha`가 커질수록 확률이 어느 쪽으로 움직이는지, 그 이유가 무엇인지 설명해 보세요.

4. **나만의 스팸 필터.** 예제에 정상/스팸 문장을 각각 5개 이상 더 추가하고, 일부러 헷갈리는 문장("무료 회의 자료 확인")을 만들어 확률이 어떻게 나오는지 살펴보세요. 단어가 하나도 어휘 사전에 없는 문장을 넣으면 어떤 일이 일어나는지도 확인하고, 왜 그런 결과가 나오는지 설명하세요.

5. **나이브 가정의 한계.** `make_classification`이나 `numpy`로 두 특성이 거의 똑같은 값(강한 상관)을 갖도록 데이터를 만든 뒤, 같은 특성을 하나만 쓴 경우와 여러 번 복사해서 쓴 경우의 `GaussianNB` 확률 출력이 어떻게 달라지는지 비교해 보세요. 독립 가정이 깨지면 확률이 어느 방향으로 왜곡되는지 설명해 보세요.

---

📖 [◀ 목차](00-목차.md) | [◀ 이전: 6장. 데이터 전처리와 특성 공학](ch06-데이터전처리와특성공학.md) | [다음: 8장. 서포트 벡터 머신 ▶](ch08-서포트벡터머신.md)
