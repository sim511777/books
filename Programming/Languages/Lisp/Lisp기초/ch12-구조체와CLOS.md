# 12장. 구조체와 CLOS 객체 시스템 기초

[◀ 이전: 11장. 매크로 기초](ch11-매크로기초.md) | [📖 목차](00-목차.md) | [다음: 13장. 조건 시스템을 이용한 에러 처리 ▶](ch13-조건시스템을이용한에러처리.md)

---

7장에서 리스트로 데이터를 표현하는 방법을 배웠습니다. 예를 들어 사람의 이름과 나이를 `("철수" 25)`처럼 리스트에 담거나, `(:name "철수" :age 25)` 같은 속성 목록(plist)으로 표현할 수도 있었습니다. 하지만 이런 방식에는 단점이 있습니다. `(nth 0 person)`이 이름인지 나이인지는 코드를 보는 사람이 순서를 외우고 있어야만 알 수 있고, 실수로 순서를 바꿔 넣어도 Lisp은 아무 것도 경고해주지 않습니다. 이번 장에서는 이런 문제를 해결하는 두 가지 도구, **구조체(struct)**와 **CLOS(Common Lisp Object System)**를 소개합니다.

## defstruct로 이름 있는 레코드 만들기

`defstruct`는 이름이 붙은 슬롯(slot)들의 묶음, 즉 레코드 타입을 정의합니다.

```lisp
(defstruct point
  x
  y)
```

이 한 줄이 실행되면 Lisp은 다음 네 가지를 자동으로 만들어줍니다.

- 생성자 함수 `make-point` (키워드 인자로 슬롯 값을 받습니다)
- 접근자 함수 `point-x`, `point-y`
- 타입 검사 함수 `point-p`
- 복사 함수 `copy-point`

실제로 사용해봅시다.

```lisp
* (defparameter *p1* (make-point :x 3 :y 4))
*P1*

* (point-x *p1*)
3

* (point-y *p1*)
4

* *p1*
#S(POINT :X 3 :Y 4)

* (point-p *p1*)
T

* (point-p "문자열")
NIL
```

`#S(POINT :X 3 :Y 4)`라는 출력 형태는 "이것은 POINT라는 구조체이고, X 슬롯은 3, Y 슬롯은 4다"라는 뜻입니다. `(nth 0 ...)`처럼 위치로 접근할 필요 없이 `point-x`, `point-y`라는 이름으로 바로 접근할 수 있으니, 코드를 읽는 사람도 각 값이 무엇을 의미하는지 즉시 알 수 있습니다.

슬롯 값은 `setf`로 바꿀 수 있습니다.

```lisp
* (setf (point-x *p1*) 10)
10

* *p1*
#S(POINT :X 10 :Y 4)
```

### 기본값 지정하기

슬롯 이름만 적으면 초기값은 `nil`입니다. `(슬롯이름 기본값)` 형태로 적으면 `make-point`를 호출할 때 값을 생략해도 그 기본값이 채워집니다.

```lisp
(defstruct point
  (x 0)
  (y 0))
```

```lisp
* (make-point)
#S(POINT :X 0 :Y 0)

* (make-point :x 5)
#S(POINT :X 5 :Y 0)
```

### 생성자와 접근자 이름 커스터마이징

기본 접두어(`point-`)가 마음에 들지 않거나, 위치 인자로 값을 넘기는 생성자를 원한다면 옵션을 줄 수 있습니다.

```lisp
(defstruct (point (:conc-name pt-)
                   (:constructor make-pt (x y)))
  x
  y)
```

```lisp
* (defparameter *p2* (make-pt 3 4))
*P2*

* (pt-x *p2*)
3

* *p2*
#S(POINT :X 3 :Y 4)
```

`:conc-name`은 접근자 이름의 접두어를 바꾸고, `:constructor`는 위치 인자를 받는 새 생성자를 정의합니다.

### :include로 슬롯 확장하기

구조체는 `:include` 옵션으로 다른 구조체의 슬롯을 그대로 물려받을 수 있습니다.

```lisp
(defstruct point
  (x 0)
  (y 0))

(defstruct (point-3d (:include point))
  (z 0))
```

```lisp
* (defparameter *p3* (make-point-3d :x 1 :y 2 :z 3))
*P3*

* (point-x *p3*)
1

* (point-3d-z *p3*)
3

* (point-p *p3*)
T
```

`point-3d`는 `point`의 슬롯 `x`, `y`를 그대로 가지면서 `z`를 추가로 갖습니다. 흥미로운 점은 `point-3d` 값에 대해 `point-p`를 호출해도 `T`가 나온다는 것입니다. `point-3d`가 `point`를 "포함"하기 때문입니다. 다만 구조체의 `:include`는 슬롯을 물려받는 것 이상의 기능, 즉 "같은 이름의 함수가 타입에 따라 다르게 동작하는" 다형성은 제공하지 않습니다. 이 부분을 담당하는 것이 CLOS입니다.

참고로 `defstruct`는 그 자체로 매크로입니다. 11장에서 배운 것처럼 `(defstruct point x y)`라는 코드는 컴파일 시점에 `make-point`, `point-x`, `point-y` 등을 정의하는 훨씬 긴 코드로 확장됩니다. 매크로가 무엇에 쓰이는지 궁금했다면, `defstruct`가 좋은 실제 사례입니다.

## CLOS 맛보기: defclass

구조체는 데이터를 담는 상자로는 충분하지만, "타입에 따라 동작이 달라지는 함수"를 표현하기는 어렵습니다. 예를 들어 원과 사각형이 있을 때 "넓이를 구하라"는 하나의 요청을 각 도형에 맞게 다르게 처리하고 싶다면, Common Lisp의 객체 시스템인 **CLOS**를 씁니다.

CLOS에서 클래스는 `defclass`로 정의합니다.

```lisp
(defclass shape ()
  ())
```

`shape`는 슬롯도 없고 부모 클래스도 없는(두 번째 `()`가 부모 클래스 목록입니다) 가장 단순한 클래스입니다. 이제 `shape`를 부모로 하는 구체적인 도형 클래스를 만들어봅시다.

```lisp
(defclass circle (shape)
  ((radius :initarg :radius
           :accessor circle-radius)))

(defclass rectangle (shape)
  ((width :initarg :width :accessor rectangle-width)
   (height :initarg :height :accessor rectangle-height)))
```

슬롯 하나마다 `:initarg`(인스턴스를 만들 때 사용할 키워드 인자 이름)와 `:accessor`(값을 읽고 쓸 함수 이름)를 지정합니다. 인스턴스는 `make-instance`로 만듭니다.

```lisp
* (defparameter *c* (make-instance 'circle :radius 5))
*C*

* (circle-radius *c*)
5

* (defparameter *r* (make-instance 'rectangle :width 3 :height 4))
*R*

* (rectangle-width *r*)
3
```

구조체의 `make-point`, `point-x`와 하는 역할이 비슷해 보이지만, `defclass`는 여기서 한 걸음 더 나아갑니다.

## defgeneric과 defmethod로 다형성 표현하기

"넓이를 구하라"는 동작을 **제네릭 함수(generic function)**로 선언합니다. 제네릭 함수는 "이런 이름과 이런 인터페이스의 함수가 있다"는 선언일 뿐, 구체적인 구현은 담지 않습니다.

```lisp
(defgeneric area (shape))
```

실제 구현은 `defmethod`로, 클래스별로 따로 작성합니다.

```lisp
(defmethod area ((c circle))
  (* pi (expt (circle-radius c) 2)))

(defmethod area ((r rectangle))
  (* (rectangle-width r) (rectangle-height r)))
```

`(c circle)`처럼 매개변수 이름 옆에 클래스 이름을 적은 것을 **특수화(specializer)**라고 부릅니다. "이 메서드는 `c`가 `circle` 타입일 때만 호출된다"는 뜻입니다. 이제 같은 이름 `area`를 호출해도 인자의 클래스에 따라 다른 코드가 실행됩니다.

```lisp
* (area *c*)
78.53981633974483

* (area *r*)
12

* (mapcar #'area (list *c* *r*))
(78.53981633974483 12)
```

이것이 CLOS의 핵심인 **다형성(polymorphism)**입니다. 호출하는 쪽은 `area`라는 이름 하나만 알면 되고, "원이면 이렇게, 사각형이면 저렇게" 계산하는 책임은 각 클래스에 대응하는 메서드가 나누어 가집니다. 나중에 삼각형 클래스가 추가되어도, 기존 코드를 전혀 건드리지 않고 `(defmethod area ((tr triangle)) ...)`만 추가하면 됩니다. `defmethod`를 `defgeneric` 없이 바로 써도 Lisp이 알아서 제네릭 함수를 만들어주지만, 여러 메서드가 어떤 인터페이스를 공유하는지 명확히 드러내려면 `defgeneric`을 먼저 선언해두는 습관이 좋습니다.

참고로 CLOS의 제네릭 함수는 인자 하나가 아니라 **여러 인자의 조합**으로 메서드를 선택할 수도 있습니다(다중 디스패치). 이 책에서는 다루지 않지만, "두 도형이 겹치는지 검사하는 함수를 두 도형의 타입 조합별로 다르게 구현하고 싶다"는 상황을 만나면 CLOS의 이 기능을 찾아보기 바랍니다. 13장에서 다룰 조건 시스템의 조건(condition) 타입들도 사실 이 CLOS 클래스 체계 위에 만들어져 있습니다.

## 구조체와 클래스, 언제 무엇을 쓸까

두 도구는 겹치는 부분도 있지만 성격이 다릅니다.

| | 구조체 (`defstruct`) | 클래스 (`defclass` + CLOS) |
|---|---|---|
| 목적 | 고정된 모양의 데이터 레코드 | 동작(메서드)이 타입마다 달라지는 객체 |
| 다형성 | 기본적으로 없음 | `defmethod`로 자연스럽게 지원 |
| 상속 | `:include`로 단일 상속만 | 다중 상속 가능 |
| 실행 속도 | CLOS보다 가볍고 빠른 경우가 많음 | 약간의 오버헤드가 있음 |
| 적합한 상황 | 좌표, 설정값, 파싱 결과처럼 필드만 있으면 충분한 데이터 | "이 요청을 타입에 따라 다르게 처리하라" 같은 로직이 필요한 경우 |

경험적인 기준은 이렇습니다. 그냥 값 몇 개를 이름 붙여서 묶어두고 싶을 뿐이라면 `defstruct`로 충분하고 더 간단합니다. 반면 "같은 인터페이스를 여러 타입이 각자 다르게 구현해야 한다"거나, 나중에 새로운 타입이 계속 추가될 것으로 예상된다면 처음부터 CLOS로 설계하는 편이 유지보수에 유리합니다. 확신이 서지 않을 때는 구조체로 시작했다가, 다형성이 필요해지는 시점에 클래스로 옮겨가도 늦지 않습니다. 18장에서 이야기할 "좋은 Lisp 코드 작성 습관"에서도 이 원칙, 즉 필요한 만큼만 복잡한 도구를 쓰라는 원칙을 다시 강조할 것입니다.

## 요약

- `defstruct`는 이름 있는 슬롯을 가진 레코드 타입을 정의하며, 생성자·접근자·타입 검사 함수·복사 함수를 자동으로 만들어줍니다.
- 슬롯에는 기본값을 줄 수 있고, `:conc-name`·`:constructor` 같은 옵션으로 이름을 커스터마이징할 수 있으며, `:include`로 다른 구조체를 확장할 수 있습니다.
- CLOS는 `defclass`로 클래스와 슬롯(`:initarg`, `:accessor`)을 정의하고, `make-instance`로 인스턴스를 만듭니다.
- `defgeneric`으로 제네릭 함수를 선언하고 `defmethod`로 클래스별 구현을 등록하면, 같은 이름의 호출이 인자의 타입에 따라 다르게 동작하는 다형성을 얻습니다.
- 단순한 데이터 레코드에는 구조체가, 타입마다 동작이 달라져야 하는 경우에는 CLOS가 더 적합한 선택입니다.

## 연습문제

1. 이름(`name`)과 나이(`age`) 슬롯을 가진 `person` 구조체를 `defstruct`로 정의하고, `age` 슬롯의 기본값을 0으로 지정하세요. `make-person`으로 인스턴스 두 개를 만들고 `person-age`로 값을 확인해보세요.
2. `person` 구조체를 `:include`로 확장한 `student` 구조체를 만들어보세요. `student`는 `school`이라는 슬롯을 추가로 가져야 합니다. `student-p`와 `person-p`를 각각 호출해서 결과를 비교해보세요.
3. `animal`이라는 부모 클래스를 만들고, 이를 상속하는 `dog`와 `cat` 클래스를 정의하세요. `defgeneric speak (animal)`을 선언한 뒤 `dog`는 `"멍멍!"`을, `cat`은 `"야옹!"`을 출력하도록 `defmethod`를 각각 작성하세요.
4. 문제 3에서 만든 `dog`와 `cat` 인스턴스를 리스트에 담고 `mapcar`와 `#'speak`를 이용해 한 번에 모든 동물이 소리를 내도록 해보세요.
5. `area` 제네릭 함수 예제에 `triangle` 클래스(밑변 `base`, 높이 `height` 슬롯)를 추가하고, 넓이를 계산하는 `defmethod`를 작성해서 기존 `circle`, `rectangle`과 함께 `(mapcar #'area (list ...))`로 한꺼번에 계산해보세요.

---

[◀ 이전: 11장. 매크로 기초](ch11-매크로기초.md) | [📖 목차](00-목차.md) | [다음: 13장. 조건 시스템을 이용한 에러 처리 ▶](ch13-조건시스템을이용한에러처리.md)
