# 3장. Slint 언어(.slint) 기초 문법

📖 [◀ 목차](00-목차.md) | [◀ 이전: 2장. 개발 환경 설정과 첫 창](ch02-개발환경설정과첫창.md) | [다음: 4장. 레이아웃 시스템 ▶](ch04-레이아웃시스템.md)

---

Slint 애플리케이션은 언제나 두 개의 세계로 나뉘어 있습니다. 화면의 생김새를 기술하는 `.slint` 파일과, 그 화면을 실행하고 데이터를 흘려 넣는 Rust 코드입니다. 지금까지는 프로젝트를 준비하고 창 하나를 띄워 보는 데 집중했다면, 이번 장에서는 그 `.slint` 파일 안에 실제로 무엇을 적을 수 있는지를 문법 단위로 하나씩 파헤쳐 보겠습니다. 그리고 각 문법 요소가 `slint::include_modules!()`를 통해 Rust 쪽에서 어떤 모습의 코드로 나타나는지도 함께 확인합니다. `.slint` 문법을 익히는 것과 그것이 Rust API로 어떻게 연결되는지를 아는 것, 이 두 가지가 이 책 전체를 관통하는 축입니다.

## 컴포넌트: `component ... inherits ...`

Slint의 UI는 **컴포넌트(component)** 단위로 조립됩니다. 컴포넌트는 화면의 일부 혹은 전체를 표현하는 재사용 가능한 정의이며, 다음과 같은 형태로 선언합니다.

```slint
component Greeting inherits Rectangle {
    // 컴포넌트의 내용
}
```

`component` 키워드 다음에 컴포넌트 이름(관례적으로 파스칼 케이스를 씁니다)이 오고, `inherits` 다음에는 이 컴포넌트가 어떤 엘리먼트나 다른 컴포넌트의 성질을 물려받는지를 적습니다. `Rectangle`을 상속하면 배경색, 테두리, 크기 같은 사각형의 프로퍼티를 그대로 물려받은 채로 컴포넌트를 시작하게 됩니다.

Rust 코드에서 최상위 창으로 인스턴스화할 컴포넌트는 `Window`를 상속하고, 앞에 `export` 키워드를 붙입니다. `export`가 붙지 않은 컴포넌트는 같은 파일 안에서 부품으로만 쓰이고 Rust 쪽에는 노출되지 않습니다.

```slint
export component AppWindow inherits Window {
    width: 320px;
    height: 200px;
    title: "첫 Rust 연동 창";
}
```

이렇게 정의한 `AppWindow`는 빌드 과정을 거치면 Rust 쪽에서 `AppWindow`라는 이름의 구조체로 나타나며, `AppWindow::new()`로 인스턴스를 만들 수 있게 됩니다.

```rust
slint::include_modules!();

fn main() -> Result<(), slint::PlatformError> {
    let ui = AppWindow::new()?;
    ui.run()
}
```

`slint::include_modules!()`는 `build.rs`에서 `slint_build::compile("경로/AppWindow.slint")`로 컴파일해 둔 결과를 현재 Rust 파일에 그대로 포함시키는 매크로입니다. `.slint` 파일에 `export`로 선언한 컴포넌트마다 이렇게 대응하는 Rust 구조체가 하나씩 생겨난다고 생각하면 됩니다.

## 프로퍼티 선언: `property <타입> 이름: 값;`

컴포넌트가 가지는 데이터는 **프로퍼티(property)** 로 선언합니다. 문법은 다음과 같습니다.

```slint
property <int> counter: 0;
property <string> username: "guest";
property <bool> is-enabled: true;
property <color> accent-color: #3498db;
property <length> box-size: 48px;
property <float> ratio: 0.5;
```

콜론 뒤의 값은 **기본값**입니다. `<타입>` 자리에 들어갈 수 있는 대표적인 타입들은 다음과 같습니다.

| 타입 | 설명 | 예시 값 |
|---|---|---|
| `int` | 정수 | `42`, `-3` |
| `float` | 부동소수점 실수 | `0.5`, `3.14` |
| `bool` | 참/거짓 | `true`, `false` |
| `string` | 문자열 | `"hello"` |
| `color` | 색상 (알파 채널 포함) | `#3498db`, `#3498dbff` |
| `brush` | 색상 또는 그라디언트 | `#3498db`, `@linear-gradient(...)` |
| `length` | 논리 픽셀 단위 길이 | `48px` |
| `percent` | 백분율 값 | `50%` |

프로퍼티 이름은 Slint 관례상 케밥 케이스(`is-enabled`, `box-size`처럼 하이픈으로 단어를 구분)로 씁니다. 이 이름이 Rust 쪽 API로 넘어갈 때는 자동으로 스네이크 케이스(`is_enabled`, `box_size`)로 바뀝니다.

### 프로퍼티의 방향: `in`, `out`, `in-out`

프로퍼티 앞에 아무 키워드도 붙이지 않으면 그 프로퍼티는 **컴포넌트 내부에서만** 읽고 쓸 수 있는 비공개 프로퍼티가 됩니다. 컴포넌트 바깥, 즉 Rust 코드나 이 컴포넌트를 사용하는 다른 컴포넌트가 값을 주고받으려면 방향을 명시해야 합니다.

```slint
export component AppWindow inherits Window {
    in property <string> title-text: "기본 제목";
    out property <bool> is-busy: false;
    in-out property <int> counter: 0;
}
```

- `in`: 바깥에서 값을 넣어줄 수 있는 프로퍼티. Rust에서 `set_...` 메서드로 값을 밀어 넣습니다.
- `out`: 컴포넌트 내부에서 계산된 값을 바깥으로 읽게 해주는 프로퍼티. Rust에서 `get_...` 메서드로 읽을 수 있지만 `set_...`는 생성되지 않습니다.
- `in-out`: 양방향으로 열려 있어 `get_...`와 `set_...` 모두 생성됩니다.

이 방향 지정은 단순한 문서화가 아니라 실제로 Rust 쪽에 어떤 메서드가 생성되는지를 결정합니다. `in-out property <int> counter: 0;`을 선언하면 Rust 쪽에는 다음과 같은 메서드가 생성됩니다.

```rust
let ui = AppWindow::new()?;
ui.set_counter(10);
let value: i32 = ui.get_counter();
```

방향 키워드가 없는 프로퍼티는 Rust 쪽에 아예 메서드가 생성되지 않는다는 점을 꼭 기억해 두세요. "Rust에서 건드리고 싶은 값에는 `in`, `out`, `in-out` 중 하나를 붙인다"가 실무에서 가장 자주 저지르는 실수를 막아 주는 규칙입니다.

## 표현식과 바인딩: 선언적 사고방식

Slint의 프로퍼티 값 자리에는 리터럴뿐 아니라 **표현식**을 쓸 수 있습니다.

```slint
property <int> base: 10;
property <int> doubled: base * 2;
property <string> label: "합계: " + (base + doubled);
```

여기서 `doubled`는 `base * 2`라는 **바인딩(binding)** 을 가집니다. 명령형 언어에 익숙하다면 이 줄을 "지금 이 순간 `base * 2`를 계산해서 `doubled`에 대입한다"는 일회성 대입문으로 읽고 싶어질 수 있습니다. 하지만 Slint에서는 전혀 다르게 동작합니다. `doubled`는 `base`가 바뀔 때마다 **다시 계산되는 살아있는 관계**로 선언된 것입니다. `base`의 값이 10에서 20으로 바뀌면, 누가 명시적으로 갱신 코드를 실행하지 않아도 `doubled`는 자동으로 40이 되고, 그 값을 참조하고 있는 `label`도 함께 갱신됩니다.

이것이 Slint가 **선언적(declarative)** 언어라고 불리는 이유입니다. 우리는 "언제 다시 계산할지"를 신경 쓰지 않고 "값들 사이의 관계가 무엇인지"만 선언합니다. Rust 코드에서 흔히 보는 다음과 같은 명령형 스타일과 비교해 보면 차이가 분명해집니다.

```rust
// 명령형 스타일: base가 바뀔 때마다 doubled를 다시 계산해서 대입해야 한다
let mut base = 10;
let mut doubled = base * 2;
base = 20;
doubled = base * 2; // 이 줄을 잊으면 doubled는 낡은 값(20)으로 남는다
```

```slint
// 선언적 스타일: 관계를 한 번 선언해 두면 base가 바뀔 때 doubled가 저절로 따라온다
property <int> base: 10;
property <int> doubled: base * 2;
```

바인딩은 다른 프로퍼티뿐 아니라 조건식과도 함께 쓸 수 있습니다.

```slint
property <int> score: 0;
property <color> status-color: score >= 60 ? #2ecc71 : #e74c3c;
```

`score`가 바뀌어 60을 넘나드는 순간 `status-color`는 자동으로 초록과 빨강 사이를 오갑니다. 이런 자동 재계산은 이후 5장(프로퍼티와 바인딩)에서 더 깊이 다루게 될 Slint의 핵심 메커니즘입니다. 지금은 "`:` 뒤에 오는 것은 대입이 아니라 관계의 선언"이라는 감각만 확실히 잡고 넘어가면 충분합니다.

## 콜백 선언: `callback 이름(인자) -> 반환타입;`

프로퍼티가 "상태"를 표현한다면, **콜백(callback)** 은 "특정 시점에 일어나는 사건"을 표현합니다. 버튼이 눌렸다거나, 값이 특정 조건을 넘었다거나 하는 순간에 컴포넌트 바깥으로 신호를 보내고 싶을 때 콜백을 선언합니다.

```slint
callback clicked();
callback value-changed(int);
callback compute-sum(int, int) -> int;
```

- 인자 없는 콜백: `callback clicked();`
- 인자가 있는 콜백: 타입만 나열합니다. `callback value-changed(int);`
- 반환값이 있는 콜백: `-> 타입`을 붙입니다. `callback compute-sum(int, int) -> int;`

콜백은 선언만으로는 아무 동작도 하지 않습니다. `.slint` 파일 안에서 `=>` 문법으로 실제 동작을 연결하거나, Rust 쪽에서 핸들러를 등록해야 비로소 의미가 생깁니다.

### `.slint` 안에서 콜백에 반응하기

내장 엘리먼트인 `TouchArea`는 `clicked`라는 콜백을 이미 가지고 있습니다. 여기에 `=>`로 동작을 연결할 수 있습니다.

```slint
export component ClickCounter inherits Window {
    width: 200px;
    height: 100px;

    in-out property <int> count: 0;

    Rectangle {
        background: #3498db;

        TouchArea {
            clicked => {
                count += 1;
            }
        }
    }
}
```

`clicked => { ... }`는 "`clicked` 콜백이 발생하면 중괄호 안의 문장을 실행한다"는 뜻입니다. 이렇게 `.slint` 파일 안에서 콜백을 소비할 수도 있고, 아래처럼 내가 만든 컴포넌트의 콜백을 부모(여기서는 Rust)에게 그대로 전달할 수도 있습니다.

```slint
export component ClickCounter inherits Window {
    width: 200px;
    height: 100px;

    callback increase-requested();

    Rectangle {
        background: #3498db;

        TouchArea {
            clicked => {
                root.increase-requested();
            }
        }
    }
}
```

`root`는 현재 컴포넌트 자기 자신을 가리키는 예약어입니다. `TouchArea`가 눌렸을 때 `ClickCounter` 컴포넌트 자신이 선언한 `increase-requested` 콜백을 호출해서, 이 사건을 컴포넌트 바깥, 즉 Rust 쪽으로 넘겨주는 구조입니다.

### Rust에서 콜백 다루기

Rust 쪽에서는 `on_이름` 메서드로 콜백이 발생했을 때 실행할 클로저를 등록하고, `invoke_이름` 메서드로 Rust에서 반대로 콜백을 직접 호출할 수 있습니다.

```rust
slint::include_modules!();

fn main() -> Result<(), slint::PlatformError> {
    let ui = ClickCounter::new()?;

    let ui_weak = ui.as_weak();
    ui.on_increase_requested(move || {
        let ui = ui_weak.unwrap();
        ui.set_count(ui.get_count() + 1);
    });

    ui.run()
}
```

`ui.as_weak()`로 약한 참조를 얻어 클로저 안에서 사용하는 이유는, 클로저가 `ui`를 강하게 붙잡아버리면 `ui` 자신이 콜백 클로저를 소유하는 순환 참조가 생기기 때문입니다. 이 패턴은 Slint Rust 코드에서 매우 자주 등장하므로 지금부터 눈에 익혀 두는 것이 좋습니다.

## 기본 내장 엘리먼트

컴포넌트의 중괄호 안을 채우는 재료가 **엘리먼트(element)** 입니다. 이번 절에서는 가장 자주 쓰이는 네 가지, `Rectangle`, `Text`, `Image`, `TouchArea`를 살펴봅니다.

### Rectangle

`Rectangle`은 배경색과 테두리를 가진 사각형이면서, 동시에 다른 엘리먼트를 담는 컨테이너 역할도 합니다.

```slint
Rectangle {
    width: 120px;
    height: 80px;
    background: #ecf0f1;
    border-width: 2px;
    border-color: #bdc3c7;
    border-radius: 10px;
}
```

- `background`: 배경색(또는 그라디언트를 넣을 수 있는 `brush`)
- `border-width`, `border-color`: 테두리 두께와 색
- `border-radius`: 모서리를 둥글게 깎는 정도

### Text

`Text`는 문자열을 화면에 그립니다.

```slint
Text {
    text: "안녕하세요, Slint";
    font-size: 18px;
    color: #2c3e50;
    horizontal-alignment: center;
}
```

`text`는 문자열 프로퍼티이므로, 앞서 본 것처럼 다른 프로퍼티와 결합한 표현식을 그대로 넣을 수 있습니다. `horizontal-alignment`는 `left`, `center`, `right` 중 하나를 받아 가로 정렬을 지정합니다.

### Image

`Image`는 이미지 파일을 화면에 표시합니다. 경로는 `@image-url(...)` 문법으로 지정하며, 이 경로는 `.slint` 파일을 기준으로 한 상대 경로입니다.

```slint
Image {
    source: @image-url("./assets/logo.png");
    width: 64px;
    height: 64px;
    image-fit: contain;
}
```

`image-fit`은 원본 이미지 비율과 지정된 크기가 다를 때 어떻게 맞출지를 결정합니다(`fill`, `contain`, `cover` 등).

### TouchArea

`TouchArea`는 그 자체로는 아무것도 그리지 않는 투명한 엘리먼트로, 마우스 클릭이나 터치를 감지하는 역할만 합니다. 그래서 대개 `Rectangle` 같은 시각적 엘리먼트 안에 겹쳐서 배치합니다.

```slint
Rectangle {
    width: 100px;
    height: 40px;
    background: pressed-area.pressed ? #2980b9 : #3498db;

    pressed-area := TouchArea {
        clicked => {
            debug("버튼이 눌렸습니다");
        }
    }
}
```

`이름 := 엘리먼트타입 { ... }` 문법으로 엘리먼트에 이름을 붙이면, 같은 컴포넌트 안 다른 곳에서 `pressed-area.pressed`처럼 그 엘리먼트의 프로퍼티를 참조할 수 있습니다. 위 예제는 `TouchArea`가 기본으로 제공하는 읽기 전용 프로퍼티 `pressed`(눌려 있는 동안 `true`)를 이용해, 눌려 있는 동안 배경색이 살짝 진해지는 효과를 표현식 하나로 구현한 것입니다.

## 주석 문법

Slint는 C 계열 언어와 동일한 두 가지 주석 문법을 지원합니다.

```slint
// 한 줄 주석

/*
   여러 줄에 걸친
   블록 주석
*/

component Sample inherits Rectangle {
    // 프로퍼티 설명을 여기에 적어두면 나중에 읽기 편합니다
    property <int> value: 0; // 줄 끝 주석도 가능합니다
}
```

## 여러 컴포넌트와 `import`

### 한 파일 안에 여러 컴포넌트

하나의 `.slint` 파일 안에는 여러 컴포넌트를 나란히 선언할 수 있습니다. 화면의 부품이 커지면 이렇게 작은 컴포넌트로 쪼개어 조립하는 것이 자연스러운 접근입니다.

```slint
component RoundButton inherits Rectangle {
    in property <string> label: "버튼";
    callback clicked();

    height: 40px;
    border-radius: 20px;
    background: #3498db;

    Text {
        text: label;
        color: white;
        horizontal-alignment: center;
        vertical-alignment: center;
    }

    TouchArea {
        clicked => { root.clicked(); }
    }
}

export component AppWindow inherits Window {
    width: 240px;
    height: 120px;

    RoundButton {
        x: 20px;
        y: 20px;
        width: 200px;
        label: "확인";
        clicked => { debug("확인 버튼 클릭"); }
    }
}
```

`RoundButton`은 `export`가 붙지 않았으므로 이 파일 안에서만 부품으로 쓰이고, Rust 쪽에는 노출되지 않습니다. `AppWindow`만 `export`되어 있으므로 Rust 쪽에서 인스턴스화할 수 있는 것은 `AppWindow` 하나뿐입니다.

### 여러 파일로 나누기: `import`

프로젝트가 커지면 컴포넌트를 별도의 파일로 분리하고 `import`로 가져와 쓰는 편이 유지보수에 유리합니다. 예를 들어 위의 `RoundButton`을 `round_button.slint`라는 별도 파일로 옮긴다고 해봅시다.

```slint
// round_button.slint
export component RoundButton inherits Rectangle {
    in property <string> label: "버튼";
    callback clicked();

    height: 40px;
    border-radius: 20px;
    background: #3498db;

    Text {
        text: label;
        color: white;
        horizontal-alignment: center;
        vertical-alignment: center;
    }

    TouchArea {
        clicked => { root.clicked(); }
    }
}
```

다른 파일로 분리해서 쓰려는 컴포넌트는 반드시 `export`를 붙여야 합니다. 이제 메인 파일에서는 `import` 문으로 가져와 그대로 사용합니다.

```slint
// app.slint
import { RoundButton } from "round_button.slint";

export component AppWindow inherits Window {
    width: 240px;
    height: 120px;

    RoundButton {
        x: 20px;
        y: 20px;
        width: 200px;
        label: "확인";
        clicked => { debug("확인 버튼 클릭"); }
    }
}
```

`import { 이름1, 이름2 } from "파일경로";` 형태로 필요한 컴포넌트를 콤마로 나열해서 가져올 수 있습니다. 경로는 `import`가 적힌 파일을 기준으로 한 상대 경로입니다. 이렇게 파일을 나누어 두면, `build.rs`에서는 여전히 최상위 파일(`app.slint`) 하나만 `slint_build::compile(...)`에 넘기면 되고, 그 안에서 참조하는 다른 `.slint` 파일들은 컴파일러가 알아서 따라가며 함께 컴파일합니다.

## 이번 장 예제 종합

지금까지 배운 컴포넌트, 프로퍼티, 콜백, 기본 엘리먼트를 하나로 모아 Rust와 연동되는 작은 카운터 앱을 완성해 보겠습니다.

```slint
// counter.slint
export component AppWindow inherits Window {
    width: 280px;
    height: 160px;
    title: "카운터";

    in-out property <int> counter: 0;
    callback increase-requested();
    callback reset-requested();

    Text {
        x: 24px;
        y: 20px;
        text: "현재 값: " + counter;
        font-size: 22px;
        color: #2c3e50;
    }

    Rectangle {
        x: 24px;
        y: 70px;
        width: 90px;
        height: 40px;
        background: inc-area.pressed ? #2980b9 : #3498db;
        border-radius: 6px;

        Text {
            text: "증가";
            color: white;
            horizontal-alignment: center;
            vertical-alignment: center;
        }

        inc-area := TouchArea {
            clicked => { root.increase-requested(); }
        }
    }

    Rectangle {
        x: 130px;
        y: 70px;
        width: 90px;
        height: 40px;
        background: reset-area.pressed ? #c0392b : #e74c3c;
        border-radius: 6px;

        Text {
            text: "초기화";
            color: white;
            horizontal-alignment: center;
            vertical-alignment: center;
        }

        reset-area := TouchArea {
            clicked => { root.reset-requested(); }
        }
    }
}
```

```rust
// main.rs
slint::include_modules!();

fn main() -> Result<(), slint::PlatformError> {
    let ui = AppWindow::new()?;

    let ui_weak = ui.as_weak();
    ui.on_increase_requested(move || {
        let ui = ui_weak.unwrap();
        ui.set_counter(ui.get_counter() + 1);
    });

    let ui_weak = ui.as_weak();
    ui.on_reset_requested(move || {
        let ui = ui_weak.unwrap();
        ui.set_counter(0);
    });

    ui.run()
}
```

```rust
// build.rs
fn main() {
    slint_build::compile("counter.slint").unwrap();
}
```

`in-out property counter`가 Rust 쪽 `get_counter`/`set_counter`로, `callback increase-requested()`와 `callback reset-requested()`가 각각 `on_increase_requested`/`on_reset_requested`로 대응된 것을 확인할 수 있습니다. 화면 쪽 시각 효과(눌렸을 때 색이 진해지는 것)는 전부 `.slint` 파일 안의 선언적 바인딩만으로 처리되고, "값이 몇 번 증가했는가" 같은 실제 로직만 Rust 쪽 콜백 핸들러로 넘어와 있다는 점도 눈여겨볼 만합니다. 이 역할 분담이 Slint와 Rust를 함께 쓸 때의 기본적인 설계 감각입니다.

## 요약

- 모든 Slint UI는 `component 이름 inherits 부모 { ... }`로 정의되며, Rust에서 인스턴스화할 컴포넌트는 `export`를 붙이고 보통 `Window`를 상속합니다.
- 프로퍼티는 `property <타입> 이름: 값;`으로 선언합니다. 방향 키워드 `in`/`out`/`in-out`을 붙여야 Rust 쪽에 `get_...`/`set_...` 메서드가 생성됩니다.
- `:` 뒤의 값은 일회성 대입이 아니라 **선언적 바인딩**입니다. 참조하는 다른 프로퍼티가 바뀌면 자동으로 다시 계산됩니다.
- 콜백은 `callback 이름(인자타입, ...) -> 반환타입;`으로 선언하고, `.slint` 안에서는 `이름 => { ... }`로, Rust에서는 `on_이름(클로저)`로 동작을 연결합니다.
- `Rectangle`, `Text`, `Image`, `TouchArea`는 각각 사각 영역, 문자열, 이미지, 입력 감지를 담당하는 기본 내장 엘리먼트입니다. `TouchArea`는 시각 요소가 없으므로 대개 `Rectangle` 등과 겹쳐서 사용합니다.
- 주석은 `//`와 `/* */`를 사용하며, 컴포넌트는 한 파일에 여러 개 선언하거나 `export`와 `import`를 통해 여러 파일로 나눌 수 있습니다.

## 연습문제

1. `in-out property <bool> is-on: false;`를 가진 `AppWindow` 컴포넌트를 만들고, `TouchArea`를 눌렀을 때 `is-on` 값이 반전되도록 `.slint` 파일만으로 구현해 보세요.
2. 위 문제의 `is-on` 값에 따라 `Rectangle`의 `background`가 초록색(`#2ecc71`)과 회색(`#95a5a6`) 사이를 오가도록 삼항 조건식 바인딩을 작성해 보세요.
3. `callback greet(string) -> string;`처럼 문자열을 받아 문자열을 반환하는 콜백을 선언하고, 이 콜백을 Rust에서 `on_greet`으로 등록해서 받은 이름 앞에 `"안녕하세요, "`를 붙여 돌려주도록 구현해 보세요.
4. 프로퍼티 방향 키워드를 아무것도 붙이지 않은 `property <int> secret: 42;`를 `AppWindow`에 선언한 뒤, Rust 쪽에서 `ui.get_secret()`을 호출하면 어떤 일이 벌어지는지 실제로 컴파일해서 확인해 보세요.
5. `RoundButton`처럼 `label` 프로퍼티와 `clicked` 콜백을 가진 재사용 가능한 버튼 컴포넌트를 별도의 `.slint` 파일로 분리하고, 메인 파일에서 `import`로 가져와 두 개 이상 배치해 보세요.


---

📖 [◀ 목차](00-목차.md) | [◀ 이전: 2장. 개발 환경 설정과 첫 창](ch02-개발환경설정과첫창.md) | [다음: 4장. 레이아웃 시스템 ▶](ch04-레이아웃시스템.md)
