# 17장. ML 파이프라인과 모델 저장·배포

📖 [◀ 목차](00-목차.md) | [◀ 이전: 16장. 시퀀스 데이터와 RNN 입문](ch16-시퀀스데이터와RNN입문.md) | [다음: 18장. 실전 프로젝트: 데이터 수집부터 배포까지 엔드투엔드 ▶](ch18-실전프로젝트엔드투엔드.md)

---

지금까지 우리는 노트북이나 스크립트 안에서 모델을 만들고, 점수를 확인하고, 그것으로 끝냈습니다. 그런데 모델이 진짜로 쓸모를 갖는 순간은 **다른 사람이나 다른 프로그램이 그 모델의 예측을 받아 쓰는 순간**입니다. 쇼핑몰의 추천 목록, 은행의 사기 거래 탐지, 공장의 불량 예측이 모두 그렇습니다. 학습을 마친 모델을 그런 자리까지 옮기려면 몇 가지 현실적인 문제를 풀어야 합니다.

- 학습할 때 쓴 전처리(결측치 채우기, 스케일링, 범주형 인코딩)를 **예측할 때도 똑같이** 적용해야 합니다. 전처리 코드를 따로 두면 한쪽만 고치고 다른 쪽을 잊기 쉽습니다.
- 학습한 모델을 파일로 저장하고, 다른 프로세스에서 다시 불러와야 합니다. 이때 파일 형식의 **보안과 버전 호환성**을 알아 두어야 합니다.
- 모델을 호출 가능한 서비스(API)로 감싸야 합니다.
- 서비스를 시작한 뒤에도 세상은 변합니다. 모델의 성능이 서서히 떨어지는 것을 **알아차리고 다시 학습**할 방법이 필요합니다.

이 장에서는 scikit-learn의 `Pipeline`과 `ColumnTransformer`로 전처리와 모델을 한 덩어리로 묶는 방법부터 시작해, `joblib`으로 저장하고, 웹 API로 서빙하는 코드의 모양을 살펴보고, 배포 후에 데이터가 변하는 현상(드리프트)을 감지하는 방법까지 다룹니다. 마지막으로 PyTorch 모델을 저장하는 방법을 간단히 소개합니다.

> 이 장의 코드 블록은 하나의 파일에 이어서 쓰는 것으로 생각하세요. 뒤쪽 블록은 앞 블록에서 정의한 변수와 함수(`df`, `y`, `model`, `search` 등)를 그대로 사용합니다. 출력은 Python 3.14, NumPy 2.5, scikit-learn 1.9 환경에서 실제로 실행한 결과입니다. 난수를 쓰는 부분은 시드를 고정했지만 라이브러리 버전이 다르면 소수점 아래 값이 조금 달라질 수 있습니다. 단, 17.5절의 웹 서버 코드와 17.7절의 PyTorch 코드는 이 책이 사용한 실행 환경에서 직접 실행하지 못했으므로 출력을 싣지 않았습니다. 해당 라이브러리를 설치해 직접 확인해 보시기 바랍니다.

## 17.1 왜 파이프라인으로 묶는가

### 데이터 누수: 보이지 않는 반칙

머신러닝에서 가장 조용하게 일어나는 실수는 **데이터 누수(data leakage)** 입니다. 테스트(또는 검증) 데이터의 정보가 학습 과정에 새어 들어가서, 실제보다 훨씬 좋은 점수가 나오는 현상입니다. 점수가 좋아 보이기 때문에 실수를 알아차리기도 어렵습니다. 5장의 5.3절에서 이 문제를 처음 만났습니다. 그때는 "먼저 분리하고, 훈련 데이터로만 특성을 고른다"는 규칙을 손으로 지켰다면, 이 장에서는 그 규칙을 **코드의 구조로 강제하는 도구**를 배웁니다.

대표적인 사례가 "전체 데이터로 전처리를 먼저 하고, 그다음에 훈련/검증으로 나누는" 순서입니다. 평균과 표준편차를 구하거나, 쓸모 있는 특성을 고르거나, 결측치를 채우는 값을 계산할 때 이미 검증 데이터를 들여다본 셈이 되기 때문입니다. 효과가 얼마나 큰지 극단적인 실험으로 확인해 봅시다. 특성 2000개와 정답이 **모두 순수한 난수**인 데이터입니다. 이 데이터에서 알아낼 수 있는 패턴은 처음부터 없으므로 정확도는 50% 근처여야 합니다.

```python
import numpy as np
from sklearn.feature_selection import SelectKBest, f_classif
from sklearn.linear_model import LogisticRegression
from sklearn.model_selection import StratifiedKFold, cross_val_score
from sklearn.pipeline import make_pipeline

rng = np.random.default_rng(0)
X = rng.normal(size=(100, 2000))      # 순수한 잡음 특성 2000개
y = rng.integers(0, 2, size=100)      # 정답도 무작위

cv = StratifiedKFold(n_splits=5, shuffle=True, random_state=0)

# 잘못된 방법: 전체 데이터로 특성을 먼저 고른 뒤 교차검증
X_sel = SelectKBest(f_classif, k=20).fit_transform(X, y)
leaky = cross_val_score(LogisticRegression(max_iter=1000), X_sel, y, cv=cv)

# 올바른 방법: 특성 선택을 파이프라인 안에 넣는다
pipe = make_pipeline(SelectKBest(f_classif, k=20), LogisticRegression(max_iter=1000))
honest = cross_val_score(pipe, X, y, cv=cv)

print(f"누수가 있는 평가 정확도: {leaky.mean():.3f}")
print(f"파이프라인 평가 정확도: {honest.mean():.3f}")
```

**실행 결과**

```text
누수가 있는 평가 정확도: 0.810
파이프라인 평가 정확도: 0.520
```

정답과 아무 관계도 없는 데이터에서 81%의 정확도가 나왔습니다. 잘못된 방법에서는 `SelectKBest`가 **정답 `y`까지 보면서** 2000개 중 우연히 정답과 상관이 높게 나온 20개를 골랐고, 이 선택에 검증 데이터의 정답이 이미 반영되어 있었기 때문입니다. 올바른 방법에서는 교차검증의 매 번(fold)마다 특성 선택을 **그 회차의 훈련 데이터로만** 수행하므로 정확도가 0.52, 즉 찍는 수준으로 나옵니다. 이것이 정직한 답입니다.

### 파이프라인이 해 주는 일

`Pipeline`은 여러 변환 단계와 마지막 모델 하나를 **하나의 추정기(estimator)** 처럼 다루게 해 줍니다. `fit`을 호출하면 각 단계가 앞에서부터 차례로 `fit_transform`을 수행하고, `predict`를 호출하면 같은 변환을 `transform`으로 적용한 뒤 마지막에 예측합니다. 파이프라인으로 묶으면 얻는 이점은 세 가지입니다.

1. **누수 방지.** `cross_val_score`나 `GridSearchCV`에 파이프라인을 통째로 넘기면 매 fold의 훈련 데이터로만 전처리를 `fit`합니다. 위 실험이 보여 준 그대로입니다.
2. **재현성과 일관성.** 전처리와 모델이 한 객체이므로, 학습 때와 예측 때 서로 다른 전처리를 적용하는 실수가 구조적으로 사라집니다. 저장할 파일도 하나입니다.
3. **하이퍼파라미터 탐색의 범위 확장.** 모델의 `C`뿐 아니라 "결측치를 평균으로 채울지 중앙값으로 채울지" 같은 전처리 선택까지 한꺼번에 탐색할 수 있습니다(17.3절).

## 17.2 Pipeline과 ColumnTransformer

실제 데이터는 숫자 열과 범주 열이 섞여 있고, 열마다 필요한 전처리가 다릅니다. 숫자 열에는 결측치 채우기와 스케일링을, 범주 열에는 결측치 채우기와 원-핫 인코딩을 적용해야 합니다. 열마다 다른 변환을 적용하는 도구가 **`ColumnTransformer`** 입니다. 파이프라인 안에 `ColumnTransformer`를 넣으면 "열별 전처리 + 모델"이 하나로 묶입니다. 6장의 6.7절에서 `ColumnTransformer`와 `make_pipeline`을 맛보기로 썼는데, 여기서는 단계에 이름을 붙여 `Pipeline`으로 정식으로 구성하고, 하이퍼파라미터 탐색과 저장·배포까지 이어 갑니다.

### 예제 데이터

설명을 위해 가상의 통신사 고객 데이터를 만들어 봅니다. 나이, 월 소득(만원), 가입 기간(개월) 같은 숫자 열과 요금제, 지역 같은 범주 열이 있고, 일부 값이 비어 있으며, 정답은 이탈(해지) 여부입니다.

```python
import pandas as pd

def make_customers(n, seed, drift=False):
    """가상의 통신사 고객 데이터를 만든다. drift=True면 시장 환경이 바뀐 상황."""
    rng = np.random.default_rng(seed)
    age = rng.integers(18, 75, n)
    income = rng.normal(350, 90, n).clip(80)          # 월 소득(만원)
    tenure = rng.integers(1, 72, n)                   # 가입 기간(개월)
    plan = rng.choice(["basic", "standard", "premium"], size=n, p=[0.5, 0.3, 0.2])
    region = rng.choice(["seoul", "gyeonggi", "busan", "etc"], size=n)

    if drift:
        income = income * 1.3                         # 공변량 변화: 소득 분포 이동

    z = (0.4 - 0.07 * tenure + 1.2 * (plan == "basic") - 1.0 * (plan == "premium")
         - 0.008 * (income - 350) + 0.03 * (40 - age))
    if drift:
        z = z + 3.0 * (plan == "premium")             # 개념 변화: 프리미엄 고객 이탈 급증
    churn = rng.random(n) < 1 / (1 + np.exp(-z))

    df = pd.DataFrame({"age": age, "income": income, "tenure_months": tenure,
                       "plan": plan, "region": region})
    df.loc[rng.random(n) < 0.08, "income"] = np.nan   # 결측치 8%
    df.loc[rng.random(n) < 0.05, "region"] = np.nan   # 결측치 5%
    return df, churn.astype(int)

df, y = make_customers(2000, seed=1)
print(df.head(4).to_string())
print("\n이탈 비율:", y.mean().round(3))
print("\n결측치 개수:\n" + df.isna().sum().to_string())
print("\n자료형:\n" + df.dtypes.to_string())
```

`drift=True` 옵션은 17.6절에서 "세상이 변한 뒤의 데이터"를 흉내 낼 때 사용합니다. 지금은 `drift=False`만 쓰므로 무시하셔도 됩니다.

**실행 결과**

```text
   age      income  tenure_months      plan    region
0   44  281.511064             62     basic     seoul
1   47  258.985029             66     basic     busan
2   61  429.331295             37  standard  gyeonggi
3   72  374.390895             63     basic       etc

이탈 비율: 0.22

결측치 개수:
age                0
income           171
tenure_months      0
plan               0
region           121

자료형:
age                int64
income           float64
tenure_months      int64
plan                 str
region               str
```

> 참고: 문자열 열의 자료형이 `object`가 아니라 `str`로 나오는 것은 pandas 3 버전의 기본 동작입니다. pandas 2 이하에서는 `object`로 보입니다. 아래 코드는 두 경우 모두 동일하게 동작합니다.

### 열별 전처리를 하나로 묶기

```python
from sklearn.compose import ColumnTransformer
from sklearn.impute import SimpleImputer
from sklearn.model_selection import train_test_split
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import OneHotEncoder, StandardScaler

numeric_features = ["age", "income", "tenure_months"]
categorical_features = ["plan", "region"]

numeric_pipe = Pipeline([
    ("imputer", SimpleImputer(strategy="median")),
    ("scaler", StandardScaler()),
])
categorical_pipe = Pipeline([
    ("imputer", SimpleImputer(strategy="most_frequent")),
    ("onehot", OneHotEncoder(handle_unknown="ignore")),
])
preprocess = ColumnTransformer([
    ("num", numeric_pipe, numeric_features),
    ("cat", categorical_pipe, categorical_features),
])
model = Pipeline([
    ("preprocess", preprocess),
    ("clf", LogisticRegression(max_iter=1000)),
])

X_train, X_test, y_train, y_test = train_test_split(
    df, y, test_size=0.25, stratify=y, random_state=0)

model.fit(X_train, y_train)
print("테스트 정확도:", round(model.score(X_test, y_test), 3))
print("단계 이름:", list(model.named_steps))
names = model[:-1].get_feature_names_out()
print("전처리 후 특성 수:", len(names))
print(names)
```

**실행 결과**

```text
테스트 정확도: 0.808
단계 이름: ['preprocess', 'clf']
전처리 후 특성 수: 10
['num__age' 'num__income' 'num__tenure_months' 'cat__plan_basic'
 'cat__plan_premium' 'cat__plan_standard' 'cat__region_busan'
 'cat__region_etc' 'cat__region_gyeonggi' 'cat__region_seoul']
```

코드의 구조를 짚어 봅시다.

- `Pipeline`의 인자는 `(이름, 객체)` 튜플의 리스트입니다. 이름은 나중에 `model.named_steps["clf"]`처럼 단계를 꺼내거나, 17.3절에서 하이퍼파라미터를 지정할 때 씁니다. 이름을 일일이 짓기 귀찮으면 `make_pipeline(...)`이 클래스 이름을 소문자로 바꿔 자동으로 이름을 붙입니다.
- `ColumnTransformer`의 인자는 `(이름, 변환기, 적용할 열)` 튜플의 리스트입니다. 여기에 언급하지 않은 열은 기본적으로 **버려집니다**(`remainder="drop"`). 그대로 통과시키려면 `remainder="passthrough"`를 지정합니다.
- 범주 열에 `handle_unknown="ignore"`를 준 것이 중요합니다. 이 옵션이 없으면 학습 때 보지 못한 범주가 예측 때 들어오는 순간 오류가 납니다. 운영 환경에서는 거의 반드시 일어나는 일입니다.
- `model[:-1]`처럼 파이프라인을 슬라이스하면 "마지막 모델을 뺀 전처리 부분"만 꺼낼 수 있습니다. `get_feature_names_out()`은 전처리 후의 열 이름을 알려 주므로, 계수나 특성 중요도를 해석할 때 유용합니다. 원-핫 인코딩으로 열이 5개에서 10개로 늘어난 것을 볼 수 있습니다.

핵심은 결측치가 있는 **원본 DataFrame을 그대로** `fit`과 `predict`에 넣었다는 점입니다. 전처리 코드를 따로 호출할 필요가 없습니다. 학습 때 보지 못한 지역(`"jeju"`)과 결측 소득이 들어와도 예측이 됩니다.

```python
new_customer = pd.DataFrame([{"age": 34, "income": np.nan, "tenure_months": 5,
                              "plan": "basic", "region": "jeju"}])
print("이탈 확률:", model.predict_proba(new_customer)[0, 1].round(3))
```

**실행 결과**

```text
이탈 확률: 0.818
```

## 17.3 파이프라인 전체를 GridSearchCV에 넣기

파이프라인 단계 안의 하이퍼파라미터는 **`단계이름__파라미터이름`** 형식(밑줄 두 개)으로 가리킵니다. 단계가 중첩되어 있으면 밑줄 두 개로 계속 이어 갑니다. 예를 들어 `preprocess` 단계 안의 `num` 변환기 안의 `imputer`의 `strategy`는 `preprocess__num__imputer__strategy`입니다.

또 하나의 요령은 `param_grid`에 **딕셔너리 리스트**를 주는 것입니다. 리스트의 각 딕셔너리는 독립적인 탐색 공간이 되므로, 마지막 단계(`clf`) 자체를 다른 모델로 바꿔 끼우는 탐색도 가능합니다.

```python
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import GridSearchCV

param_grid = [
    {   # 후보 1: 로지스틱 회귀
        "preprocess__num__imputer__strategy": ["mean", "median"],
        "clf__C": [0.1, 1.0, 10.0],
    },
    {   # 후보 2: 마지막 단계를 랜덤 포레스트로 교체
        "clf": [RandomForestClassifier(random_state=0)],
        "clf__n_estimators": [100, 200],
        "clf__max_depth": [3, 6],
    },
]
search = GridSearchCV(model, param_grid, cv=5, scoring="roc_auc", n_jobs=1)
search.fit(X_train, y_train)

print("최고 교차검증 AUC:", round(search.best_score_, 4))
print("최적 조합:")
for k, v in search.best_params_.items():
    print(f"  {k}: {v}")
print("테스트 AUC:", round(search.score(X_test, y_test), 4))
```

**실행 결과**

```text
최고 교차검증 AUC: 0.8774
최적 조합:
  clf__C: 10.0
  preprocess__num__imputer__strategy: mean
테스트 AUC: 0.8312
```

탐색이 끝난 뒤 `refit=True`(기본값) 덕분에 최적 조합으로 **전체 훈련 데이터를 다시 학습한** 모델이 `search.best_estimator_`에 들어 있습니다. 이것도 `Pipeline`이므로 곧바로 저장해서 배포할 수 있습니다. 두 후보 계열의 성적도 비교해 봅시다.

```python
res = pd.DataFrame(search.cv_results_)
is_rf = res["params"].apply(lambda p: "clf" in p)      # 두 번째 후보(랜덤 포레스트) 행
print(f"로지스틱 회귀 후보 {int((~is_rf).sum())}개 중 최고 AUC:",
      round(res.loc[~is_rf, "mean_test_score"].max(), 4))
print(f"랜덤 포레스트 후보 {int(is_rf.sum())}개 중 최고 AUC:",
      round(res.loc[is_rf, "mean_test_score"].max(), 4))
```

**실행 결과**

```text
로지스틱 회귀 후보 6개 중 최고 AUC: 0.8774
랜덤 포레스트 후보 4개 중 최고 AUC: 0.8588
```

이 데이터는 로지스틱 회귀의 가정에 맞게 만든 것이라 단순한 선형 모델이 이겼습니다. 현실 데이터에서는 어느 쪽이 이길지 알 수 없으니, 이처럼 후보를 같은 전처리와 같은 교차검증 조건에서 겨루게 하는 것이 공정한 비교입니다. 이 과정에서 전처리가 매 fold의 훈련 데이터로만 `fit`되므로 17.1절의 누수 문제도 발생하지 않습니다.

> 팁: 전처리 단계가 오래 걸리는데 하이퍼파라미터 탐색은 모델 쪽만 바꾼다면 `Pipeline(..., memory="캐시폴더")` 옵션으로 변환 결과를 캐시해 시간을 아낄 수 있습니다.

## 17.4 모델 저장과 불러오기

### joblib으로 저장하기

학습이 끝난 파이프라인을 파일로 저장해 두면, 다른 프로세스(서버, 배치 작업, 다른 노트북)에서 학습 없이 바로 예측에 쓸 수 있습니다. scikit-learn 객체를 저장할 때는 보통 **`joblib`** 을 씁니다. `joblib`은 NumPy 배열을 많이 가진 객체를 Python 기본 `pickle`보다 효율적으로 저장하며, scikit-learn을 설치하면 함께 설치됩니다.

모델만 달랑 저장하기보다, 나중에 필요한 **메타데이터**(입력 열 이름, 학습에 쓴 라이브러리 버전)를 함께 딕셔너리로 묶어 저장하는 습관을 권합니다.

```python
import joblib
from pathlib import Path
import sklearn

Path("models").mkdir(exist_ok=True)
best_model = search.best_estimator_

bundle = {
    "model": best_model,
    "feature_names": numeric_features + categorical_features,
    "sklearn_version": sklearn.__version__,
}
joblib.dump(bundle, "models/churn_model.joblib")

loaded = joblib.load("models/churn_model.joblib")
same = np.array_equal(best_model.predict_proba(X_test), loaded["model"].predict_proba(X_test))
print("저장 전후 예측이 같은가:", same)
print("파일 크기(KB):", round(Path("models/churn_model.joblib").stat().st_size / 1024, 1))
print("저장된 scikit-learn 버전:", loaded["sklearn_version"])

joblib.dump(bundle, "models/churn_model_c3.joblib", compress=3)
print("압축 후 크기(KB):", round(Path("models/churn_model_c3.joblib").stat().st_size / 1024, 1))
```

**실행 결과**

```text
저장 전후 예측이 같은가: True
파일 크기(KB): 4.3
저장된 scikit-learn 버전: 1.9.0
압축 후 크기(KB): 1.9
```

`joblib.dump(객체, 경로)`로 저장하고 `joblib.load(경로)`로 불러옵니다. 저장 전후의 예측 확률이 완전히 같음을 확인했습니다. 이 모델은 로지스틱 회귀라 몇 킬로바이트에 불과하지만, 랜덤 포레스트처럼 트리를 수백 개 가진 모델은 수십~수백 메가바이트가 되기도 합니다. `compress` 인자(0~9)를 주면 파일이 작아지는 대신 저장과 로딩이 느려집니다. 확장자는 관례적으로 `.joblib`을 씁니다.

### 주의 1: 신뢰할 수 없는 파일을 불러오지 마세요

`joblib`은 내부적으로 Python의 `pickle` 형식을 사용합니다. **pickle은 데이터가 아니라 "객체를 다시 만드는 절차"를 저장하는 형식**이며, 그 절차에는 임의의 함수를 호출하는 명령도 담을 수 있습니다. 즉, 파일을 **불러오는 순간 그 안에 숨겨진 코드가 실행**될 수 있습니다. 무해한 예제로 동작 원리만 확인해 봅시다.

```python
import pickle

class Harmless:
    def __reduce__(self):
        # 불러올 때 print(...)를 호출하라는 지시. print 대신 어떤 함수든 올 수 있다.
        return (print, ("pickle.loads가 임의의 함수를 실행했습니다",))

payload = pickle.dumps(Harmless())
result = pickle.loads(payload)
print("반환값:", result)
```

**실행 결과**

```text
pickle.loads가 임의의 함수를 실행했습니다
반환값: None
```

`Harmless` 객체는 존재하지도 않았는데, `pickle.loads`가 `print`를 호출했습니다. `print` 자리에 파일을 지우거나 외부로 정보를 보내는 함수가 들어 있다면 어떻게 될지 상상해 보시기 바랍니다. 그래서 다음 규칙을 지켜야 합니다.

- **출처를 모르는 `.pkl`, `.joblib` 파일은 절대 `load`하지 않습니다.** 인터넷에서 내려받은 모델 파일, 메신저로 받은 파일도 마찬가지입니다. 내가 만들었거나 신뢰할 수 있는 저장소에서 온 파일만 불러옵니다.
- 서버에서 사용자가 업로드한 파일을 곧바로 `joblib.load`하는 기능은 만들지 않습니다.
- 모델 파일을 저장하는 위치의 쓰기 권한을 제한합니다. 파일이 바꿔치기되면 같은 결과가 됩니다.
- 신뢰 경계를 넘나드는 상황이라면 pickle에 의존하지 않는 방식을 고려합니다. 예를 들어 scikit-learn 모델을 더 안전하게 저장하려는 `skops` 라이브러리나, 프레임워크 중립적인 모델 형식인 ONNX 같은 대안이 있습니다. 이 책에서는 이름만 소개합니다.

### 주의 2: 버전 호환성

pickle은 객체의 **내부 구조**를 그대로 저장합니다. scikit-learn 버전이 올라가면서 클래스의 내부 속성이 바뀔 수 있으므로, 한 버전으로 저장한 파일을 다른 버전에서 불러오면 경고가 나오거나(`InconsistentVersionWarning`), 불러오기가 실패하거나, 심하면 **조용히 다른 결과**를 낼 수 있습니다. NumPy나 Python 버전이 다른 경우도 마찬가지입니다.

대응 방법은 다음과 같습니다.

- 학습 환경과 서비스 환경의 라이브러리 버전을 **같게 고정**합니다. 가장 흔한 방법은 `requirements.txt`에 `scikit-learn==1.9.0`처럼 정확한 버전을 적고 같은 파일로 두 환경을 구성하는 것입니다. 컨테이너를 쓰면 더 확실합니다.
- 저장할 때 버전을 메타데이터로 함께 기록하고(위 `sklearn_version` 항목), 불러올 때 현재 버전과 비교해 다르면 경고하거나 중단합니다.
- 버전을 올릴 때는 저장된 모델을 그대로 쓰지 말고, **학습 코드와 데이터로 모델을 다시 만드는 것**을 기본으로 삼습니다. 모델 파일은 "재생성 가능한 산출물"로 생각하는 편이 안전합니다.

### 주의 3: 직접 만든 코드는 불러오는 쪽에도 있어야 합니다

pickle은 클래스나 함수의 **코드를 저장하지 않고 이름(모듈 경로)만 저장**합니다. 그래서 파이프라인 안에 내가 정의한 변환 함수(`FunctionTransformer(my_func)`)나 직접 만든 클래스가 있으면, 불러오는 쪽에서도 같은 이름으로 그 코드를 가져올 수 있어야 합니다. 특히 학습 스크립트의 `__main__`에 정의한 클래스는 서버에서 불러올 때 찾지 못해 오류가 납니다. 사용자 정의 코드는 별도의 모듈 파일로 분리해 두고 학습과 서빙이 같은 모듈을 `import`하도록 하세요. 이 장의 예제처럼 scikit-learn이 제공하는 클래스만 쓰면 이 문제가 없습니다.

## 17.5 추론 API로 서빙하기

모델을 파일로 저장했으니, 이제 다른 프로그램이 HTTP로 예측을 요청할 수 있게 감싸 봅시다. 흐름은 단순합니다.

1. 서버가 **시작할 때 한 번만** 모델 파일을 불러와 메모리에 둡니다. 요청이 올 때마다 불러오면 매우 느립니다.
2. 클라이언트가 JSON으로 입력(고객 정보)을 보냅니다.
3. 서버가 입력을 검증하고, 학습 때와 같은 열 구성의 DataFrame으로 바꿔 `predict_proba`를 호출합니다.
4. 결과를 JSON으로 돌려줍니다.

웹 프레임워크로는 **FastAPI**를 사용하겠습니다. 입력 검증이 간결하고 자동으로 API 문서를 만들어 주기 때문입니다. **Flask**도 같은 목적에 널리 쓰이는데, 아래 코드의 구조(시작 시 로드, 입력 변환, 예측, JSON 응답)는 어느 프레임워크든 같습니다. 필요한 패키지는 `fastapi`와 서버 실행기인 `uvicorn`이며, 이 책에서는 설치법을 `pip install fastapi uvicorn` 정도로만 언급합니다.

### 핵심 로직은 프레임워크와 분리하기

먼저 웹 프레임워크와 무관하게 "입력 dict 목록을 받아 예측 결과를 돌려주는" 순수 함수를 만듭니다. 이렇게 분리하면 서버 없이도 함수만 따로 테스트할 수 있고, 나중에 프레임워크를 바꾸거나 배치 작업에서 재사용하기도 쉽습니다. 이 함수는 앞에서 저장한 `loaded` 번들로 실제 실행해 확인했습니다.

```python
def predict_records(bundle, records, threshold=0.5):
    """dict 목록을 받아 예측 결과 목록을 돌려준다. 웹 프레임워크와 무관한 순수 함수."""
    frame = pd.DataFrame(records, columns=bundle["feature_names"])
    for c in ["age", "income", "tenure_months"]:
        frame[c] = pd.to_numeric(frame[c])          # None -> NaN
    for c in ["plan", "region"]:
        frame[c] = frame[c].astype(object).where(frame[c].notna(), np.nan)
    proba = bundle["model"].predict_proba(frame)[:, 1]
    return [{"churn_probability": round(float(p), 4), "churn": bool(p >= threshold)}
            for p in proba]

print(predict_records(loaded, [
    {"age": 34, "income": None, "tenure_months": 5, "plan": "basic", "region": None},
    {"age": 52, "income": 420.0, "tenure_months": 60, "plan": "premium", "region": "seoul"},
]))
```

**실행 결과**

```text
[{'churn_probability': 0.7842, 'churn': True}, {'churn_probability': 0.0026, 'churn': False}]
```

JSON의 `null`은 Python에서 `None`으로 들어옵니다. 함수 안에서 `None`을 `NaN`으로 바꿔 준 이유를 실험으로 확인해 봅시다. 요청 한 건만 DataFrame으로 만들면, 값이 `None` 하나뿐인 열은 `object` 자료형이 됩니다. 이때 `SimpleImputer`는 `None`을 결측치로 인식하지 못하고 채우지 않은 채 통과시킵니다.

```python
rec = {"age": 34, "income": 300.0, "tenure_months": 5, "plan": "basic", "region": None}
naive = pd.DataFrame([rec])
print("region 자료형:", naive["region"].dtype)
print("변환 없이  :", loaded["model"].predict_proba(naive)[0, 1].round(4))
print("변환 후    :", predict_records(loaded, [rec])[0]["churn_probability"])
```

**실행 결과**

```text
region 자료형: object
변환 없이  : 0.9159
변환 후    : 0.8537
```

오류는 한 번도 나지 않았지만 같은 고객의 이탈 확률이 0.9159와 0.8537로 달라졌습니다. 변환하지 않은 쪽은 결측 지역을 "가장 흔한 지역"으로 채우는 단계를 건너뛰어, 학습 때와 다른 방식으로 처리된 것입니다. 이처럼 **서빙 환경의 지저분한 입력은 오류 없이 조용히 결과만 바꾸는** 경우가 많으므로, 서빙 코드를 짤 때는 결측치와 자료형 변환을 꼭 점검하고 테스트하세요.

### FastAPI로 `/predict` 엔드포인트 만들기

다음은 위 함수를 FastAPI로 감싼 서버 코드입니다. `serve.py`로 저장합니다.

```python
# serve.py
from contextlib import asynccontextmanager
from typing import Literal

import joblib
import numpy as np
import pandas as pd
from fastapi import FastAPI
from pydantic import BaseModel, Field

MODEL_PATH = "models/churn_model.joblib"
state = {}  # 서버가 떠 있는 동안 모델을 들고 있는 공간


def predict_records(bundle, records, threshold=0.5):
    """(위에서 만든 함수와 동일)"""
    frame = pd.DataFrame(records, columns=bundle["feature_names"])
    for c in ["age", "income", "tenure_months"]:
        frame[c] = pd.to_numeric(frame[c])
    for c in ["plan", "region"]:
        frame[c] = frame[c].astype(object).where(frame[c].notna(), np.nan)
    proba = bundle["model"].predict_proba(frame)[:, 1]
    return [{"churn_probability": round(float(p), 4), "churn": bool(p >= threshold)}
            for p in proba]


@asynccontextmanager
async def lifespan(app: FastAPI):
    state["bundle"] = joblib.load(MODEL_PATH)  # 서버 시작 시 한 번만 로드
    yield
    state.clear()


app = FastAPI(title="Churn Prediction API", lifespan=lifespan)


class Customer(BaseModel):
    age: int = Field(ge=18, le=100)
    income: float | None = None
    tenure_months: int = Field(ge=0)
    plan: Literal["basic", "standard", "premium"]
    region: str | None = None


class Prediction(BaseModel):
    churn_probability: float
    churn: bool


@app.get("/health")
def health():
    return {"status": "ok", "sklearn_version": state["bundle"]["sklearn_version"]}


@app.post("/predict", response_model=Prediction)
def predict(customer: Customer):
    return predict_records(state["bundle"], [customer.model_dump()])[0]
```

이 코드의 포인트는 다음과 같습니다.

- **`lifespan`**: 서버가 시작될 때 `yield` 앞부분이 실행되어 모델을 한 번 불러오고, 종료될 때 뒷부분이 실행됩니다.
- **`Customer`(pydantic 모델)**: 요청 본문의 JSON 구조와 타입, 허용 범위를 선언합니다. `plan`에 `"vip"` 같은 값을 보내거나 `age`를 `-3`으로 보내면 FastAPI가 모델 코드에 닿기도 전에 **422 오류로 거부**합니다. 모델은 이상한 입력에도 아무 일 없다는 듯 숫자를 내놓기 때문에, 입력 검증은 서빙 쪽의 중요한 책임입니다.
- **`/health`**: 서버가 살아 있는지 확인하는 용도입니다. 배포 환경의 상태 점검(health check)에 흔히 쓰입니다.
- **`/predict`**: POST로 고객 정보를 받아 이탈 확률을 돌려줍니다. `def`로 정의한 동기 함수는 FastAPI가 별도의 스레드에서 실행하므로, 예측이 약간 오래 걸려도 다른 요청 처리를 막지 않습니다.
- `customer.model_dump()`는 pydantic v2의 메서드로 모델 객체를 dict로 바꿉니다.

실행 방법과 호출 예시는 다음과 같습니다. 프로젝트 폴더에서 서버를 시작합니다(`--reload`는 개발 중에 코드 변경을 자동 반영하는 옵션이며 운영에서는 쓰지 않습니다).

```bash
uvicorn serve:app --reload
```

다른 터미널에서 `curl`로 호출합니다.

```bash
curl -X POST http://127.0.0.1:8000/predict \
  -H "Content-Type: application/json" \
  -d '{"age": 34, "income": null, "tenure_months": 5, "plan": "basic", "region": null}'
```

응답은 다음과 같은 JSON이 되도록 작성되었습니다. 숫자는 위 `predict_records`를 같은 입력으로 실행해서 얻은 값입니다.

```json
{"churn_probability": 0.7842, "churn": true}
```

브라우저에서 `http://127.0.0.1:8000/docs`를 열면 FastAPI가 자동으로 만든 대화형 문서 화면에서 입력을 채워 바로 호출해 볼 수도 있습니다. 윈도우 PowerShell은 따옴표 처리 방식이 달라 위 `curl` 명령이 그대로 동작하지 않을 수 있으니 이 화면을 쓰는 편이 편합니다.

서버를 띄우지 않고 코드로 테스트하려면 FastAPI가 제공하는 `TestClient`를 쓸 수 있습니다(`httpx` 패키지가 필요합니다). `with` 문으로 감싸야 `lifespan`이 실행되어 모델이 로드됩니다.

```python
from fastapi.testclient import TestClient
from serve import app

with TestClient(app) as client:
    r = client.post("/predict", json={"age": 34, "income": None, "tenure_months": 5,
                                      "plan": "basic", "region": None})
    print(r.status_code, r.json())
```

### 서빙할 때 더 생각할 것들

- **모델 버전 관리.** 응답에 모델 버전(저장 파일 이름이나 학습 일자)을 담아 두면, 나중에 "이 예측은 어느 모델이 냈는가"를 추적할 수 있습니다.
- **입력과 예측의 기록(로깅).** 요청과 응답을 저장해 두어야 17.6절의 모니터링이 가능합니다. 개인정보는 비식별화하거나 저장하지 않도록 주의합니다.
- **배치 예측.** 요청이 많으면 한 건씩 보내기보다 여러 건을 묶어 한 번에 `predict_proba`를 호출하는 편이 훨씬 빠릅니다. `predict_records`가 목록을 받도록 만든 이유입니다.
- **인증과 접근 제어.** 예제에는 인증이 없습니다. 실제 서비스에서는 인증, 요청 제한, HTTPS를 반드시 갖추세요.
- 모델을 서버에 맡기지 않고 앱 안에 내장하거나(모바일, 엣지) 클라우드의 관리형 서빙 서비스를 쓰는 방법도 있습니다. 어느 쪽이든 "저장 → 로드 → 입력 검증 → 예측 → 로깅"이라는 골격은 같습니다.

## 17.6 모델 모니터링과 재학습: 데이터 드리프트

### 배포는 끝이 아니라 시작입니다

모델은 **과거 데이터에서 배운 규칙**으로 미래를 예측합니다. 세상이 바뀌면 그 규칙이 더는 맞지 않게 됩니다. 코드는 한 번 짜면 그대로 돌아가지만, 머신러닝 모델은 코드가 한 줄도 바뀌지 않아도 **시간이 지나며 성능이 떨어질 수 있습니다.** 흔히 구분하는 두 가지 변화는 다음과 같습니다.

| 종류 | 무엇이 변하는가 | 예시 |
|---|---|---|
| **데이터 드리프트**(공변량 변화) | 입력 `X`의 분포 | 물가가 올라 고객 소득 분포가 전체적으로 올라감 |
| **개념 드리프트** | 입력과 정답의 관계 `X → y` | 경쟁사의 신규 요금제 출시로 프리미엄 고객도 대거 이탈 |

데이터 드리프트는 정답이 없어도 입력만 보고 감지할 수 있습니다. 반면 개념 드리프트는 정답(실제 이탈 여부)이 확인된 뒤에야 성능 저하로 드러나는데, 이탈 여부는 몇 주~몇 달 뒤에야 알 수 있는 경우가 많습니다. 그래서 **정답이 오기 전에는 입력 분포와 예측 분포의 변화를 먼저 감시**합니다.

### 드리프트 흉내 내기

17.2절의 `make_customers(..., drift=True)`는 두 가지 변화를 모두 일으킵니다. 소득이 30% 오르고(데이터 드리프트), 프리미엄 고객의 이탈 성향이 크게 높아집니다(개념 드리프트). 먼저 입력 분포가 달라졌는지 **콜모고로프-스미르노프(KS) 검정**으로 확인합니다. `scipy.stats.ks_2samp`는 두 표본의 분포가 얼마나 다른지를 0(똑같음)~1(완전히 다름)의 통계량으로 알려 줍니다. scikit-learn을 설치하면 SciPy도 함께 설치됩니다.

```python
from scipy.stats import ks_2samp
from sklearn.metrics import roc_auc_score

df_new, y_new = make_customers(1000, seed=2, drift=True)

print("특성별 KS 통계량 (학습 데이터 vs 운영 데이터)")
for col in numeric_features:
    a = X_train[col].dropna()
    b = df_new[col].dropna()
    stat, p = ks_2samp(a, b)
    print(f"  {col:14s} KS={stat:.3f}  p={p:.2e}")

auc_old = roc_auc_score(y_test, best_model.predict_proba(X_test)[:, 1])
auc_new = roc_auc_score(y_new, best_model.predict_proba(df_new)[:, 1])
print(f"\n학습 시점 테스트 AUC: {auc_old:.3f}")
print(f"운영 데이터 AUC     : {auc_new:.3f}")

proba_new = best_model.predict_proba(df_new)[:, 1]
print(f"\n운영 데이터 평균 예측 확률: {proba_new.mean():.3f}")
print(f"운영 데이터 실제 이탈 비율: {y_new.mean():.3f}")
```

**실행 결과**

```text
특성별 KS 통계량 (학습 데이터 vs 운영 데이터)
  age            KS=0.032  p=5.63e-01
  income         KS=0.366  p=1.37e-66
  tenure_months  KS=0.043  p=2.06e-01

학습 시점 테스트 AUC: 0.831
운영 데이터 AUC     : 0.784

운영 데이터 평균 예측 확률: 0.165
운영 데이터 실제 이탈 비율: 0.240
```

결과를 읽어 봅시다.

- 소득(`income`)의 KS 통계량만 0.366으로 크고, 나이와 가입 기간은 0.03~0.04로 사실상 변화가 없습니다. 어느 특성이 변했는지 짚어 낼 수 있습니다.
- p값은 표본이 많으면 아주 작은 차이에도 극단적으로 작아집니다. **p값보다 통계량의 크기**를 보고 "실질적으로 얼마나 변했는가"를 판단하세요.
- 성능은 AUC 0.831에서 0.784로 떨어졌습니다. 더 눈에 띄는 것은 **보정(calibration)의 붕괴**입니다. 모델은 평균 16.5%의 고객이 이탈할 것이라 예측했지만 실제로는 24.0%가 이탈했습니다. 예측 확률을 그대로 의사결정에 쓰고 있었다면 이탈 규모를 크게 과소평가하게 됩니다.

### PSI: 실무에서 자주 쓰는 지표

**PSI(Population Stability Index)** 는 학습 시점의 분포를 기준으로 구간을 나누고, 구간별 비율이 얼마나 달라졌는지를 하나의 숫자로 요약합니다. 금융권에서 오래 쓰여 왔으며, 관례적으로 0.1 미만이면 안정, 0.1~0.25면 주의, 0.25 이상이면 큰 변화로 해석합니다. 이 기준은 절대적인 규칙이 아니라 경험칙입니다. NumPy로 간단히 구현할 수 있습니다.

```python
def psi(reference, current, bins=10):
    """Population Stability Index. 0.1 미만 안정, 0.25 이상 큰 변화로 보는 것이 관례."""
    edges = np.quantile(reference, np.linspace(0, 1, bins + 1))
    edges[0], edges[-1] = -np.inf, np.inf
    ref_frac = np.histogram(reference, edges)[0] / len(reference)
    cur_frac = np.histogram(current, edges)[0] / len(current)
    ref_frac = np.clip(ref_frac, 1e-6, None)
    cur_frac = np.clip(cur_frac, 1e-6, None)
    return float(np.sum((cur_frac - ref_frac) * np.log(cur_frac / ref_frac)))

for col in numeric_features:
    value = psi(X_train[col].dropna().to_numpy(), df_new[col].dropna().to_numpy())
    print(f"{col:14s} PSI = {value:.3f}")
```

**실행 결과**

```text
age            PSI = 0.010
income         PSI = 0.805
tenure_months  PSI = 0.014
```

기준 데이터의 분위수로 구간 경계를 만들었으므로 기준 데이터는 각 구간에 10%씩 들어갑니다. 운영 데이터가 그 비율에서 벗어난 만큼 PSI가 커집니다. 소득의 PSI가 0.805로, 0.25를 훌쩍 넘는 "큰 변화"입니다.

### 무엇을 감시하고 언제 재학습할까

실제 서비스에서 감시할 항목을 정리하면 다음과 같습니다.

1. **입력 품질**: 결측치 비율, 값의 범위, 처음 보는 범주의 비율. 데이터 파이프라인 오류는 드리프트보다 훨씬 흔한 장애 원인입니다.
2. **입력 분포**: 특성별 PSI나 KS 통계량. 주기적으로(매일, 매주) 계산해 임계값을 넘으면 알림을 보냅니다.
3. **예측 분포**: 예측 확률의 평균과 분포, 양성으로 판정하는 비율. 정답이 없어도 볼 수 있는 가장 빠른 신호입니다.
4. **실제 성능**: 정답이 확인되는 대로 AUC, 정밀도, 재현율 등을 다시 계산합니다. 지연된 정답을 모아 두는 저장소가 필요합니다.

재학습 전략은 크게 두 가지입니다. **주기적 재학습**은 매주나 매월처럼 일정 주기로 최신 데이터를 포함해 다시 학습하는 방식으로 단순하고 예측하기 쉽습니다. **조건부 재학습**은 위 지표가 임계값을 넘었을 때 시작하는 방식으로 불필요한 학습을 줄입니다. 어느 쪽이든 새 모델을 곧바로 교체하지 말고, 같은 최신 테스트 데이터에서 **현재 모델과 비교**하고, 가능하면 일부 트래픽에만 적용해 보다가(카나리 배포) 문제가 있으면 이전 모델로 **되돌릴 수 있게** 해 두세요. 17.4절의 버전 메타데이터가 여기서 쓰입니다.

데이터 드리프트가 감지되었다고 해서 모델이 반드시 틀리는 것은 아닙니다. 위 실험에서 소득 분포의 이동만으로는 성능이 크게 달라지지 않았을 수도 있습니다. 드리프트 지표는 "점검하라는 신호"이지 "재학습하라는 명령"이 아닙니다. 신호가 울리면 실제 성능에 영향이 있는지 확인하고 결정하세요.

## 17.7 PyTorch 모델 저장하기

딥러닝 모델은 scikit-learn의 `joblib` 대신 PyTorch의 방식으로 저장합니다. 14장 14.7절에서 `state_dict`를 저장하고 복원하는 법을 이미 맛봤으니, 여기서는 배포 관점에서 주의할 점을 정리합니다. 핵심은 **모델 객체 전체가 아니라 `state_dict`(학습된 가중치의 사전)만 저장**하는 것입니다. `state_dict`는 층 이름을 키로, 가중치 텐서를 값으로 갖는 딕셔너리입니다.

```python
import torch
from torch import nn


class Net(nn.Module):
    def __init__(self):
        super().__init__()
        self.layers = nn.Sequential(nn.Linear(30, 16), nn.ReLU(), nn.Linear(16, 1))

    def forward(self, x):
        return self.layers(x)


model = Net()
# ... 학습 ...

# 저장: 가중치만 저장한다
torch.save(model.state_dict(), "models/net.pt")

# 불러오기: 같은 구조의 모델을 먼저 만들고, 저장한 가중치를 덮어쓴다
model = Net()
model.load_state_dict(torch.load("models/net.pt", weights_only=True))
model.eval()  # 추론 모드: Dropout, BatchNorm 같은 층이 평가용으로 동작

with torch.no_grad():  # 추론에서는 기울기 계산이 필요 없다
    logits = model(torch.randn(4, 30))
```

- 모델 **구조(클래스 정의)는 코드로 따로 관리**하고, 파일에는 가중치만 담깁니다. 이 때문에 `Net` 클래스 정의가 불러오는 쪽에도 있어야 합니다. 17.4절의 "직접 만든 코드는 불러오는 쪽에도 있어야 한다"와 같은 이야기입니다.
- `torch.save(model, 경로)`처럼 모델 객체 전체를 저장하는 방식도 가능하지만, 클래스의 코드 위치에 의존하고 pickle의 보안 문제를 그대로 안고 있어 권하지 않습니다.
- `torch.load`의 `weights_only=True`는 파일 안에서 텐서와 기본 자료형만 불러오도록 제한해, 17.4절에서 본 임의 코드 실행의 위험을 줄입니다. 최근 PyTorch 버전에서는 이 값이 기본으로 켜져 있지만, 명시해 두면 버전에 상관없이 안전합니다. 그래도 **출처를 모르는 파일은 불러오지 않는다**는 원칙은 그대로입니다.
- `model.eval()`을 잊으면 Dropout이 켜진 채로 예측해 결과가 매번 달라질 수 있습니다. 추론 코드의 단골 실수입니다.
- **전처리도 함께 보관해야 합니다.** 가중치만으로는 예측이 되지 않습니다. 학습에 쓴 스케일러(예: `StandardScaler`)를 `joblib`으로 따로 저장하고, 서빙할 때 같은 변환을 적용해야 합니다. 이 점이 17.1절의 파이프라인 사고방식과 이어집니다.
- 학습을 중간에 멈췄다가 이어서 하려면 옵티마이저 상태와 에폭 번호까지 묶어 저장합니다. 이런 파일을 **체크포인트**라고 부릅니다.

```python
checkpoint = {"model": model.state_dict(),
              "optimizer": optimizer.state_dict(),
              "epoch": epoch}
torch.save(checkpoint, "models/checkpoint.pt")
```

이외에 파이썬 없이 모델을 실행하려는 용도로는 TorchScript나 ONNX 같은 내보내기 형식이 있습니다. 필요해질 때 공식 문서에서 찾아보시기 바랍니다.

## 요약

- **데이터 누수**는 검증 데이터의 정보가 전처리나 특성 선택 과정에 새어 들어가 점수를 부풀리는 현상입니다. 순수한 난수 데이터에서도 정확도 81%가 나올 수 있습니다.
- `Pipeline`은 전처리와 모델을 하나의 추정기로 묶어 교차검증의 매 fold마다 전처리를 훈련 데이터로만 `fit`하게 하므로 누수를 막고, 학습과 예측의 전처리를 일치시켜 줍니다.
- `ColumnTransformer`는 열마다 다른 전처리를 적용합니다. 범주형에는 `handle_unknown="ignore"`를 지정해 처음 보는 범주에 대비합니다.
- 파이프라인 전체를 `GridSearchCV`에 넣을 수 있으며, 하이퍼파라미터는 `단계__파라미터` 형식으로 지정합니다. `param_grid`를 딕셔너리 리스트로 주면 마지막 모델 자체를 바꿔 가며 겨룰 수 있습니다.
- 모델은 `joblib.dump`/`joblib.load`로 저장하고 불러오며, 입력 열 이름과 라이브러리 버전 같은 메타데이터를 함께 저장하면 좋습니다.
- `joblib`과 `pickle`은 **불러오는 순간 임의의 코드가 실행될 수 있으므로 신뢰할 수 없는 파일은 절대 불러오면 안 됩니다.** 학습과 서빙의 라이브러리 버전은 맞춰야 하고, 직접 만든 클래스나 함수는 불러오는 쪽에도 있어야 합니다.
- 추론 API는 시작 시 모델을 한 번 로드하고, 입력을 검증하며, 학습 때와 같은 열 구성으로 변환한 뒤 예측합니다. 핵심 로직을 프레임워크와 분리하면 테스트와 재사용이 쉽습니다.
- 모델은 배포 후에도 열화합니다. **데이터 드리프트**(입력 분포 변화)와 **개념 드리프트**(입력-정답 관계 변화)를 구분하고, 입력 품질, 입력 분포(PSI, KS), 예측 분포, 실제 성능을 감시해 재학습 시점을 판단합니다.
- PyTorch 모델은 `state_dict`만 `torch.save`로 저장하고, 같은 구조의 모델을 만든 뒤 `load_state_dict`로 불러옵니다. `weights_only=True`, `model.eval()`, 전처리 별도 보관을 기억하세요.

## 연습문제

1. **누수의 크기 측정.** 17.1절의 실험에서 특성 수를 20, 200, 2000, 5000으로 바꿔 가며(`SelectKBest`의 `k`는 20으로 고정) "누수가 있는 평가"와 "파이프라인 평가"의 정확도를 표로 출력해 보세요. 특성 수가 늘수록 누수의 효과가 어떻게 달라지나요? 특성 수가 20일 때 두 값이 같게 나오는 이유도 설명해 보세요. 그 이유를 "우연히 정답과 상관이 높게 나오는 특성"이라는 말을 써서 설명해 보세요.

2. **전처리 단계 추가.** 17.2절의 `numeric_pipe`에서 소득(`income`)만 로그 변환(`np.log1p`)하고 싶습니다. `FunctionTransformer(np.log1p)`를 이용해 소득 열 전용 파이프라인을 만들고, `ColumnTransformer`의 숫자 열 목록을 `age`, `tenure_months`와 `income`으로 나눠 구성해 보세요. 테스트 AUC가 달라지는지 확인하세요. (이 데이터는 소득이 정규분포 모양이어서 크게 달라지지 않을 수도 있습니다. 그렇다면 그 이유도 생각해 보세요.)

3. **후보 모델 추가.** 17.3절의 `param_grid`에 세 번째 후보로 `GradientBoostingClassifier(random_state=0)`를 추가하고(`n_estimators`와 `learning_rate`를 두세 가지씩), 세 후보 계열의 최고 교차검증 AUC를 각각 출력해 보세요.

4. **안전한 로더 작성.** 17.4절의 번들을 입력으로 받아, 저장된 `sklearn_version`이 현재 설치된 `sklearn.__version__`과 다르면 `RuntimeError`를 일으키는 `load_checked(path)` 함수를 작성하세요. 버전이 같으면 번들을 돌려줍니다. 이 함수가 해결해 주지 못하는 위험(예: 악의적으로 바뀐 파일)은 무엇인지도 한 줄로 적어 보세요.

5. **드리프트 경보기.** `make_customers`의 `drift=True` 분기를 참고해 나이가 전체적으로 10살 높아진 운영 데이터를 직접 만들어 보세요(예: `df_new["age"] += 10`). 17.6절의 `psi` 함수로 숫자 특성 세 개의 PSI를 구하고, PSI가 0.25 이상인 특성의 이름을 리스트로 돌려주는 함수 `drifted_features(reference_df, current_df, columns, threshold=0.25)`를 작성하세요. 소득이 아니라 나이만 드리프트한 경우에도 이 함수가 올바른 특성을 짚어 내는지 확인해 보세요.

---

📖 [◀ 목차](00-목차.md) | [◀ 이전: 16장. 시퀀스 데이터와 RNN 입문](ch16-시퀀스데이터와RNN입문.md) | [다음: 18장. 실전 프로젝트: 데이터 수집부터 배포까지 엔드투엔드 ▶](ch18-실전프로젝트엔드투엔드.md)
