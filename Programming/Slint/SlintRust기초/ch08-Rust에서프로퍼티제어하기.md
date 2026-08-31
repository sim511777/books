# 8장. Rust에서 프로퍼티 제어하기

📖 [◀ 목차](00-목차.md) | [◀ 이전: 7장. Rust와 Slint 연동 기초](ch07-Rust와Slint연동기초.md) | [다음: 9장. 표준 위젯 라이브러리(std-widgets) ▶](ch09-표준위젯라이브러리.md)

---

7장에서 `.slint` 파일이 Rust 코드로 컴파일되고, 그 코드를 `new()`와 `ComponentHandle`의 `run()`으로 다루는 기본 골격을 익혔습니다. 하지만 그 예제에서는 UI와 Rust 코드 사이에 어떤 데이터도 오가지 않았습니다. 창을 띄우고, 사용자가 닫을 때까지 기다리는 것이 전부였습니다.

이번 장에서는 그 골격에 실제 데이터를 흘려보냅니다. `.slint`의 `property`가 Rust 쪽에서는 어떤 형태로 나타나는지, Slint 고유 타입들이 Rust 표준 타입과 어떻게 다른지, 그리고 콜백 클로저 안에서 UI 상태와 애플리케이션 상태를 안전하게 동기화하는 방법까지 다룹니다. 이 세 가지가 합쳐지면 비로소 "UI에서 이벤트가 발생하면 Rust 로직이 실행되고, 그 결과가 다시 UI에 반영되는" 완전한 상호작용 루프가 만들어집니다.

## 8.1 프로퍼티는 get_이름() / set_이름()으로 노출된다

`.slint` 컴포넌트 안에 다음과 같이 프로퍼티를 선언했다고 해봅시다.

```slint
export component Counter inherits Window {
    in-out property <int> count: 0;
}
```

이 컴포넌트가 빌드될 때, Slint 컴파일러는 `count` 프로퍼티에 접근할 수 있는 두 개의 메서드를 생성된 `Counter` 구조체에 자동으로 추가합니다.

- `fn get_count(&self) -> i32` — 현재 값을 읽습니다.
- `fn set_count(&self, value: i32)` — 값을 새로 설정합니다.

즉 `.slint` 쪽의 프로퍼티 이름이 `count`라면, Rust 쪽 메서드 이름은 `get_count()` / `set_count()`가 됩니다. 프로퍼티 이름이 케밥-케이스(kebab-case, 하이픈으로 단어를 구분하는 표기법)로 되어 있다면, 하이픈은 언더스코어(`_`)로 바뀐 스네이크-케이스(snake_case) 이름이 됩니다.

```slint
export component Scoreboard inherits Window {
    in-out property <int> player-score: 0;
}
```

이 경우 Rust 쪽에는 `get_player_score()`와 `set_player_score(i32)`가 생성됩니다. 이 변환 규칙은 프로퍼티뿐 아니라 콜백 이름에도 동일하게 적용되며, `.slint` 언어 자체가 관례적으로 케밥-케이스를 쓰는 반면 Rust는 스네이크-케이스를 쓰기 때문에 생기는 자연스러운 변환입니다.

실제로 사용해 보면 다음과 같습니다.

```rust
slint::include_modules!();

fn main() -> Result<(), slint::PlatformError> {
    let ui = Counter::new()?;

    // 현재 값을 읽는다
    let current = ui.get_count(); // 0

    // 값을 5로 설정한다
    ui.set_count(5);

    ui.run()
}
```

`ui.set_count(5)`를 호출하면 곧바로 UI에 반영됩니다. `.slint` 쪽에서 `count`를 참조하는 텍스트나 바인딩이 있다면 자동으로 다시 계산되어 화면이 갱신됩니다. `run()`을 호출하기 전에 `set_count()`로 초기값을 세팅해 두면, 창이 뜨는 순간부터 그 값이 반영된 화면을 보여줄 수 있습니다.

> 프로퍼티가 컴포넌트 내부에 중첩된 다른 요소에 선언되어 있고 최상위로 노출(export)되지 않았다면 Rust에서 접근할 수 없습니다. Rust에서 다루고 싶은 프로퍼티는 최상위 컴포넌트에 직접 선언하거나, `in-out property`로 명시적으로 노출해야 한다는 점을 기억해 두세요. `in property`로 선언하면 Rust 쪽에는 `get_이름()`만 생성되고 `set_이름()`은 생성되지 않으며, 반대로 `out property`는 Rust에서 값을 읽는 용도로만 노출됩니다.

## 8.2 Slint 고유 타입과 Rust 표준 타입

`.slint`의 모든 프로퍼티 타입이 Rust의 기본 타입으로 그대로 대응되지는 않습니다. 정수, 실수, 불리언처럼 단순한 타입은 Rust의 기본 타입과 1:1로 대응되지만, 문자열이나 색상처럼 Slint 런타임 내부에서 여러 곳에 공유되어야 하는 타입은 Slint가 직접 정의한 전용 타입으로 노출됩니다.

| `.slint` 타입 | Rust 타입 |
|---|---|
| `int` | `i32` |
| `float` | `f32` |
| `bool` | `bool` |
| `string` | `slint::SharedString` |
| `color` | `slint::Color` |
| `brush` | `slint::Brush` |
| `length` / `physical-length` | `f32` |

### SharedString

`string` 타입 프로퍼티는 Rust의 `String`이 아니라 `slint::SharedString`으로 노출됩니다. `SharedString`은 내부적으로 참조 카운팅 방식으로 데이터를 공유하도록 설계된 문자열 타입으로, 여러 곳(UI 트리, 콜백 클로저 등)에서 같은 문자열 값을 값 복사 없이 값싸게 공유할 수 있도록 최적화되어 있습니다.

```slint
export component Greeter inherits Window {
    in-out property <string> message: "Hello";
}
```

```rust
let ui = Greeter::new()?;

// SharedString -> &str
let message: slint::SharedString = ui.get_message();
println!("message = {}", message.as_str());

// &str -> SharedString (From<&str> 구현을 이용)
ui.set_message(slint::SharedString::from("Hi there"));
ui.set_message("또는 이렇게 into()로도 변환됩니다".into());

// SharedString -> String (표준 문자열 처리가 필요할 때)
let owned: String = message.to_string();
```

`SharedString`은 `impl From<&str> for SharedString`을 제공하므로 `"문자열".into()`나 `SharedString::from("문자열")` 어느 쪽으로도 변환할 수 있고, `Display`를 구현하므로 `println!("{}", shared_string)`처럼 그대로 출력할 수도 있습니다. 반대로 표준 라이브러리 함수에 넘기기 위해 `&str`이 필요하다면 `.as_str()`을, 소유권이 있는 `String`이 필요하다면 `.to_string()`을 사용합니다.

### Color

`color` 타입 프로퍼티는 `slint::Color`로 노출됩니다.

```slint
export component Badge inherits Rectangle {
    in-out property <color> tint: #3060ff;
}
```

```rust
let ui = Badge::new()?;

let tint: slint::Color = ui.get_tint();
println!("R={} G={} B={} A={}", tint.red(), tint.green(), tint.blue(), tint.alpha());

// u8 성분 값으로부터 새 Color를 만든다
ui.set_tint(slint::Color::from_rgb_u8(0x20, 0x90, 0xff));

// 알파 채널까지 지정하고 싶다면
ui.set_tint(slint::Color::from_argb_u8(0xff, 0x20, 0x90, 0xff));
```

`Color::from_rgb_u8(r, g, b)`와 `Color::from_argb_u8(a, r, g, b)`는 각각 0~255 범위의 `u8` 성분 값으로부터 불투명한 색과 알파 채널을 가진 색을 만듭니다. 반대로 이미 만들어진 `Color`에서 각 성분 값을 읽으려면 `.red()`, `.green()`, `.blue()`, `.alpha()` 메서드를 사용합니다.

### Brush

`brush` 타입은 단색뿐 아니라 그라데이션까지 표현할 수 있는 타입으로, Rust에서는 `slint::Brush`로 노출됩니다.

```slint
export component Panel inherits Rectangle {
    in-out property <brush> fill: @linear-gradient(90deg, #ff0000, #0000ff);
}
```

```rust
let ui = Panel::new()?;

let fill: slint::Brush = ui.get_fill();

// 그라데이션이더라도 대표색 하나를 얻고 싶을 때
let approx_color = fill.color();

// Color로부터 단색 Brush를 만들어 설정한다 (From<Color> 구현을 이용)
ui.set_fill(slint::Color::from_rgb_u8(255, 0, 0).into());
```

`Brush`는 `Color`로부터 `From<Color>`를 통해 변환할 수 있으므로, 단순히 단색을 설정하고 싶을 때는 `Color`를 만든 뒤 `.into()`로 `Brush`로 바꾸면 됩니다. 그라데이션처럼 더 복잡한 `Brush`를 Rust 코드에서 직접 조립하는 것도 가능하지만, 대부분의 경우 그라데이션 자체는 `.slint` 파일 안에서 `@linear-gradient(...)` 문법으로 선언하고, Rust에서는 이미 만들어진 값을 읽거나 다른 단색으로 교체하는 정도로 다루는 편이 UI 디자인과 로직의 책임을 깔끔하게 나눌 수 있습니다.

## 8.3 Rc<RefCell<T>>로 가변 상태 공유하기

콜백을 다루기 시작하면 금방 마주치는 문제가 있습니다. 버튼을 여러 개 두고, 각 버튼의 콜백에서 같은 애플리케이션 상태를 읽고 써야 하는 상황입니다. 다음과 같은 상태 구조체가 있다고 해봅시다.

```rust
#[derive(Default)]
struct AppState {
    step: i32,
}
```

이 상태를 그냥 지역 변수로 두고 여러 콜백에서 캡처하려고 하면 컴파일이 되지 않습니다.

```rust
let mut state = AppState { step: 1 };

ui.on_increment_clicked(move || {
    state.step += 1; // 여기서 state가 클로저 안으로 move된다
});

ui.on_reset_clicked(move || {
    state.step = 1; // 오류: state는 이미 위 클로저로 이동되었다
});
```

두 번째 클로저에서 `use of moved value: 'state'`라는 컴파일 오류가 발생합니다. 콜백에 등록하는 클로저는 `'static` 수명을 가져야 하므로(콜백이 언제 호출될지, 즉 클로저가 얼마나 오래 살아남아야 할지 컴파일러가 알 수 없기 때문입니다) 지역 변수를 빌려오는 것이 아니라 `move`로 소유권 자체를 가져와야 합니다. 그런데 Rust의 소유권 규칙상 값 하나는 동시에 두 곳으로 이동할 수 없습니다. `on_increment_clicked`에 등록한 클로저가 `state`를 가져가 버린 순간, `on_reset_clicked`에 등록할 클로저는 더 이상 `state`에 접근할 방법이 없습니다.

이 문제를 해결하려면 **여러 소유자가 같은 데이터를 공유**할 수 있어야 하고, 동시에 그 데이터를 **가변으로 변경**할 수 있어야 합니다. 이 두 요구를 동시에 만족시키는 표준 라이브러리 조합이 `Rc<RefCell<T>>`입니다.

- **`Rc<T>`(Reference Counted)**는 같은 데이터를 여러 곳에서 공유 소유(shared ownership)할 수 있게 해주는 스마트 포인터입니다. `.clone()`을 호출하면 데이터를 복사하는 것이 아니라 참조 카운트만 늘리고, 마지막 `Rc`가 소멸할 때 실제 데이터가 해제됩니다.
- 하지만 `Rc<T>`는 기본적으로 내부 값에 대한 **불변 참조**(`&T`)만 제공합니다. 여러 소유자가 있는 상황에서 아무나 마음대로 값을 바꿀 수 있게 하면 안전하지 않기 때문입니다.
- **`RefCell<T>`**는 컴파일 시점이 아니라 **실행 시점에** 빌림 규칙(동시에 하나의 가변 참조, 또는 여러 개의 불변 참조)을 검사하는 내부 가변성(interior mutability) 타입입니다. `.borrow_mut()`을 호출하면 `&T`만 가지고 있어도 런타임 검사를 거쳐 `&mut T`에 준하는 접근을 얻을 수 있습니다.

즉 `Rc<RefCell<T>>`는 "여러 곳에서 공유하되(`Rc`), 필요할 때 안전하게 값을 바꾼다(`RefCell`)"는 두 가지를 합친 패턴입니다.

```rust
use std::cell::RefCell;
use std::rc::Rc;

#[derive(Default)]
struct AppState {
    step: i32,
}

let state = Rc::new(RefCell::new(AppState { step: 1 }));

let state_for_increment = state.clone();
ui.on_increment_clicked(move || {
    state_for_increment.borrow_mut().step += 1;
});

let state_for_reset = state.clone();
ui.on_reset_clicked(move || {
    state_for_reset.borrow_mut().step = 1;
});
```

이번에는 컴파일이 됩니다. `state.clone()`은 `AppState`를 복제하는 것이 아니라 `Rc`의 참조 카운트를 하나 늘린 새 핸들을 만드는 것이므로, 각 클로저는 서로 다른 `Rc` 값을 `move`로 소유하면서도 결국은 힙에 있는 같은 `RefCell<AppState>`를 가리킵니다. `.borrow_mut()`은 그 순간 다른 곳에서 동시에 빌려 쓰고 있지 않은지 런타임에 검사한 뒤 가변 접근을 내줍니다.

> `.borrow()`나 `.borrow_mut()`으로 얻은 참조는 짧게 쓰고 바로 스코프를 벗어나게 하는 것이 안전합니다. 만약 `.borrow_mut()`으로 얻은 값을 들고 있는 상태에서 같은 `RefCell`에 대해 다시 `.borrow()`나 `.borrow_mut()`을 호출하면, 컴파일 오류가 아니라 **런타임 패닉**(`already borrowed: BorrowMutError`)이 발생합니다. 콜백 클로저 하나 안에서는 보통 `.borrow_mut()`을 한 번만, 아주 짧게 쥐고 필요한 계산을 끝낸 뒤 놓아주는 방식으로 작성하면 이런 문제를 피할 수 있습니다.

## 8.4 Weak 핸들로 여러 콜백에서 UI 재사용하기

앞 절에서는 `AppState`처럼 UI와는 별개인 순수한 Rust 상태를 여러 콜백에서 공유하는 방법을 다뤘습니다. 그런데 실제로는 콜백 안에서 `AppState`뿐 아니라 `ui` 자기 자신, 즉 `get_count()` / `set_count()`를 호출하기 위한 컴포넌트 핸들에도 접근해야 하는 경우가 대부분입니다.

여기서 `ui`를 그냥 `move`로 캡처하면 어떻게 될까요?

```rust
let ui = Counter::new()?;

ui.on_increment_clicked(move || {
    ui.set_count(ui.get_count() + 1); // ui를 클로저 안으로 move
});

ui.run()?; // 오류: ui는 이미 위에서 move되었다
```

`ui`를 클로저 안으로 `move`해 버리면, 정작 `ui.run()`을 호출할 `ui` 자체가 없어져 버립니다. `Counter`를 `.clone()`(정확히는 `ComponentHandle::clone_strong()`)해서 강한 참조를 하나 더 만들어 캡처할 수도 있지만, 이 경우 콜백 클로저가 `Counter` 컴포넌트 자신에 대한 강한 참조를 들고 있고, 그 클로저는 다시 `Counter` 컴포넌트 내부에 등록되어 있으므로 **컴포넌트가 자기 자신을 참조하는 순환 구조**가 만들어집니다. 참조 카운팅 기반 메모리 관리에서 이런 순환은 참조 카운트가 결코 0에 도달하지 못하게 만들어 창을 닫아도 메모리가 해제되지 않는 문제로 이어질 수 있습니다.

이 문제를 피하기 위해 7.3절에서 소개한 `ComponentHandle::as_weak()`를 사용합니다. `as_weak()`는 참조 카운트에 영향을 주지 않는 약한 참조 `slint::Weak<Counter>`를 반환합니다.

```rust
let ui = Counter::new()?;

let ui_weak = ui.as_weak();
ui.on_increment_clicked(move || {
    // Weak를 업그레이드해서 실제로 사용 가능한 강한 핸들을 얻는다.
    // 컴포넌트가 이미 소멸된 뒤라면 None이 반환되므로,
    // unwrap()은 "아직 살아있을 것"이라고 가정할 수 있는 상황에서 사용한다.
    let ui = ui_weak.unwrap();
    ui.set_count(ui.get_count() + 1);
});

ui.run()
```

`ui_weak`는 `move` 클로저 안으로 들어가도 참조 카운트를 늘리지 않으므로 순환이 생기지 않습니다. 콜백이 실제로 호출되는 시점에 `.unwrap()`(또는 더 방어적으로 값을 다루고 싶다면 `.upgrade()`가 반환하는 `Option<Counter>`를 직접 매치)을 호출해 그 순간에만 잠깐 강한 핸들을 빌려 쓰고, 클로저 실행이 끝나면 그 강한 핸들도 함께 사라집니다. 창이 열려 있는 동안 콜백이 호출된다면 컴포넌트는 항상 살아있으므로 `.unwrap()`이 패닉을 일으킬 일은 없습니다.

같은 `Weak` 값을 `.clone()`해서 여러 콜백에 나눠 캡처하는 것도 자연스럽습니다.

```rust
let ui_weak = ui.as_weak();

let weak_for_increment = ui_weak.clone();
ui.on_increment_clicked(move || {
    let ui = weak_for_increment.unwrap();
    ui.set_count(ui.get_count() + 1);
});

let weak_for_decrement = ui_weak.clone();
ui.on_decrement_clicked(move || {
    let ui = weak_for_decrement.unwrap();
    ui.set_count(ui.get_count() - 1);
});
```

정리하면, 콜백 클로저 안에서 컴포넌트 자기 자신에 접근해야 할 때는 `ui.as_weak()`로 얻은 `Weak` 핸들을 캡처하고, 콜백이 호출되는 그 순간에만 `.unwrap()`으로 강한 핸들을 잠깐 빌리는 것이 Slint의 표준적인 관용구입니다.

## 8.5 실전 예제: 증가/감소/리셋 카운터 앱

지금까지 배운 프로퍼티 접근(`get_`/`set_`), `Rc<RefCell<T>>`를 통한 상태 공유, `Weak` 핸들 재사용을 모두 조합해, 증가·감소·리셋 버튼을 갖춘 카운터 앱을 만들어 보겠습니다.

프로젝트 구조는 다음과 같습니다.

```
counter_app/
├── Cargo.toml
├── build.rs
├── src/
│   └── main.rs
└── ui/
    └── app-window.slint
```

`ui/app-window.slint`는 현재 값을 표시하는 텍스트와 버튼 세 개로 이루어진 UI입니다. `count` 프로퍼티와 세 개의 콜백을 선언해 Rust 쪽과 주고받을 통로를 만듭니다.

```slint
// ui/app-window.slint
import { Button } from "std-widgets.slint";

export component AppWindow inherits Window {
    width: 300px;
    height: 200px;
    title: "Counter";

    // Rust에서 get_count() / set_count()로 접근할 프로퍼티
    in-out property <int> count: 0;

    // 각 버튼이 클릭되면 발생시킬 콜백. Rust에서 on_이름()으로 연결한다.
    callback increment-clicked();
    callback decrement-clicked();
    callback reset-clicked();

    VerticalLayout {
        alignment: center;
        spacing: 12px;

        Text {
            text: "count: " + count;
            font-size: 24px;
            horizontal-alignment: center;
        }

        HorizontalLayout {
            alignment: center;
            spacing: 8px;

            Button {
                text: "-";
                clicked => { root.decrement-clicked(); }
            }

            Button {
                text: "reset";
                clicked => { root.reset-clicked(); }
            }

            Button {
                text: "+";
                clicked => { root.increment-clicked(); }
            }
        }
    }
}
```

`build.rs`와 `Cargo.toml`은 7장과 동일한 패턴을 따릅니다.

```rust
// build.rs
fn main() {
    slint_build::compile("ui/app-window.slint").unwrap();
}
```

```toml
# Cargo.toml
[package]
name = "counter_app"
version = "0.1.0"
edition = "2021"

[dependencies]
slint = "1.8"

[build-dependencies]
slint-build = "1.8"
```

`src/main.rs`에서는 증감 폭(`step`)을 Rust 쪽 상태로 따로 관리한다고 가정하고, `Rc<RefCell<AppState>>`로 공유하면서 세 개의 콜백 각각에서 `Weak` 핸들로 `ui`에 접근합니다.

```rust
// src/main.rs
use std::cell::RefCell;
use std::rc::Rc;

slint::include_modules!();

struct AppState {
    step: i32,
}

fn main() -> Result<(), slint::PlatformError> {
    let ui = AppWindow::new()?;

    // 여러 콜백에서 공유할 애플리케이션 상태.
    // Rc로 공유 소유권을, RefCell로 내부 가변성을 얻는다.
    let state = Rc::new(RefCell::new(AppState { step: 1 }));

    {
        let ui_weak = ui.as_weak();
        let state = state.clone();
        ui.on_increment_clicked(move || {
            let ui = ui_weak.unwrap();
            let step = state.borrow().step;
            ui.set_count(ui.get_count() + step);
        });
    }

    {
        let ui_weak = ui.as_weak();
        let state = state.clone();
        ui.on_decrement_clicked(move || {
            let ui = ui_weak.unwrap();
            let step = state.borrow().step;
            ui.set_count(ui.get_count() - step);
        });
    }

    {
        let ui_weak = ui.as_weak();
        ui.on_reset_clicked(move || {
            let ui = ui_weak.unwrap();
            ui.set_count(0);
        });
    }

    ui.run()
}
```

각 블록을 중괄호로 감싼 것은 문법적으로 꼭 필요하지는 않지만, "이 `ui_weak`와 `state` 클론은 이 콜백 하나만을 위해 만든 것"이라는 의도를 코드 구조로 드러내기 위한 관례적인 표현입니다. 실행하면 "+"를 누를 때마다 숫자가 `step`만큼 증가하고, "-"를 누르면 감소하며, "reset"을 누르면 0으로 돌아갑니다.

이 예제의 데이터 흐름을 정리하면 다음과 같습니다.

1. 사용자가 "+" 버튼을 클릭한다.
2. `.slint` 쪽 `Button`의 `clicked` 핸들러가 `root.increment-clicked()`를 호출해 컴포넌트의 `increment-clicked` 콜백을 발생시킨다.
3. Rust에서 `ui.on_increment_clicked(...)`로 등록해 둔 클로저가 실행된다.
4. 클로저 안에서 `ui_weak.unwrap()`으로 `ui`에 대한 강한 핸들을 얻고, `state.borrow()`로 `step` 값을 읽는다.
5. `ui.set_count(ui.get_count() + step)`으로 새 값을 프로퍼티에 반영한다.
6. `.slint` 쪽의 `Text { text: "count: " + count; }` 바인딩이 `count` 프로퍼티 변경을 감지해 자동으로 다시 그려진다.

이 순환 구조 — **UI 이벤트 → Rust 콜백 → `Weak` 업그레이드 → 상태 읽기/쓰기 → 프로퍼티 갱신 → UI 자동 반영** — 은 Slint Rust 애플리케이션에서 가장 기본이 되는 상호작용 패턴입니다. 이후의 장에서 다룰 더 복잡한 위젯이나 리스트 모델도 결국 이 패턴의 확장이라고 볼 수 있습니다.

## 요약

- `.slint`에서 `property <타입> 이름: 초기값;`으로 선언한 프로퍼티는 생성된 Rust 구조체에서 `get_이름()` / `set_이름(값)` 메서드로 노출됩니다. 케밥-케이스 프로퍼티 이름(`player-score`)은 스네이크-케이스 메서드 이름(`get_player_score()`)으로 변환됩니다.
- `.slint`의 `string`/`color`/`brush` 프로퍼티는 각각 `slint::SharedString`, `slint::Color`, `slint::Brush`라는 Slint 전용 타입으로 노출됩니다. `SharedString`은 `From<&str>`과 `.as_str()`/`.to_string()`으로, `Color`는 `from_rgb_u8()`/`from_argb_u8()`과 `.red()`/`.green()`/`.blue()`/`.alpha()`로, `Brush`는 `From<Color>`와 `.color()`로 표준 타입과 오간다.
- 여러 콜백 클로저가 같은 가변 상태를 공유해야 할 때는 단순한 `struct` 변수를 `move`로 캡처할 수 없습니다(값은 한 곳으로만 이동할 수 있기 때문입니다). `Rc<T>`로 공유 소유권을, `RefCell<T>`로 런타임 검사 기반의 내부 가변성을 얻는 `Rc<RefCell<T>>` 패턴이 표준적인 해법입니다.
- 콜백 클로저 안에서 컴포넌트 자기 자신(`ui`)에 접근해야 할 때는 `ui.as_weak()`로 얻은 `Weak` 핸들을 캡처합니다. 강한 참조를 그대로 캡처하면 컴포넌트가 자기 자신을 참조하는 순환이 생길 수 있으므로, 콜백이 호출되는 순간에만 `.unwrap()`(또는 `.upgrade()`)으로 강한 핸들을 잠깐 빌려 씁니다.
- `Rc<RefCell<T>>`로 공유한 애플리케이션 상태와 `as_weak()`로 재사용하는 UI 핸들을 조합하면, "UI 이벤트 → Rust 콜백 → 상태 갱신 → 프로퍼티 반영 → UI 자동 갱신"이라는 기본 상호작용 루프를 여러 콜백에 걸쳐 안전하게 구성할 수 있습니다.

## 연습문제

1. `.slint`에 `in-out property <bool> is-enabled: true;`가 선언되어 있을 때, Rust에서 이 값을 읽고 `false`로 바꾸는 코드를 작성해 보세요. 프로퍼티 이름에 하이픈이 없는 경우와 있는 경우(`is-enabled`) 각각 Rust 메서드 이름이 어떻게 되는지도 확인해 보세요.
2. `slint::SharedString`이 Rust 표준 라이브러리의 `String`과 별도의 타입으로 존재하는 이유를, Slint 런타임이 UI 트리 전반에서 값을 참조 카운팅으로 공유한다는 점과 연결지어 설명해 보세요.
3. 8.3절에서 `Rc<RefCell<T>>` 대신 `Rc<T>`만 사용해서(즉 `RefCell` 없이) 여러 콜백이 상태를 공유하려고 하면 어떤 문제가 발생하는지, 그리고 `RefCell`이 그 문제를 어떻게 해결하는지 설명해 보세요.
4. 8.4절에서 `ui.as_weak()` 대신 `ui.clone_strong()`으로 얻은 강한 핸들을 콜백에 `move`로 캡처했다면 어떤 문제가 생길 수 있는지 설명해 보세요.
5. 8.5절의 카운터 예제에서, `step` 값을 1이 아니라 사용자가 입력한 값으로 바꿀 수 있도록 `.slint`에 `in-out property <int> step: 1;`을 추가하고, `main.rs`에서 이 프로퍼티를 읽어 `AppState.step` 대신 사용하도록 코드를 어떻게 수정해야 할지 설계해 보세요. 이 경우에도 여전히 `Rc<RefCell<AppState>>`가 필요할까요?


---

📖 [◀ 목차](00-목차.md) | [◀ 이전: 7장. Rust와 Slint 연동 기초](ch07-Rust와Slint연동기초.md) | [다음: 9장. 표준 위젯 라이브러리(std-widgets) ▶](ch09-표준위젯라이브러리.md)
