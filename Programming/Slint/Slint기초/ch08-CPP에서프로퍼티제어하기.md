# 8장. C++에서 프로퍼티 제어하기

[◀ 이전: 7장. C++와 Slint 연동 기초](ch07-CPP와Slint연동기초.md) | [📖 목차](00-목차.md) | [다음: 9장. 표준 위젯 라이브러리(std-widgets) ▶](ch09-표준위젯라이브러리.md)


7장에서 `.slint` 파일이 C++ 클래스로 컴파일되고, 그 클래스를 `ComponentHandle<T>`와 `create()`, `run()`으로 다루는 기본 골격을 익혔습니다. 하지만 그 예제에서는 UI와 C++ 코드 사이에 어떤 데이터도 오가지 않았습니다. 창을 띄우고, 사용자가 닫을 때까지 기다리는 것이 전부였습니다.

이번 장에서는 그 골격에 실제 데이터를 흘려보냅니다. 5장에서 배운 `.slint`의 `property`가 C++ 쪽에서는 어떤 형태로 나타나는지, 6장에서 배운 콜백을 C++에서 어떻게 연결하는지, 그리고 반대로 C++에서 UI의 콜백을 직접 호출하는 방법까지 다룹니다. 이 세 가지가 합쳐지면 비로소 "UI에서 로직을 호출하고, 로직의 결과를 다시 UI에 반영하는" 완전한 상호작용 루프가 만들어집니다.

## 8.1 프로퍼티는 get_이름() / set_이름()으로 노출된다

5장에서 `.slint` 컴포넌트 안에 다음과 같이 프로퍼티를 선언했던 것을 떠올려 보세요.

```slint
export component Counter inherits Window {
    property <int> count: 0;
}
```

이 컴포넌트가 빌드될 때, Slint 컴파일러는 `count` 프로퍼티에 접근할 수 있는 두 개의 C++ 멤버 함수를 생성된 `Counter` 클래스에 자동으로 추가합니다.

- `int get_count() const` — 현재 값을 읽습니다.
- `void set_count(int value)` — 값을 새로 설정합니다.

즉 `.slint` 쪽의 프로퍼티 이름이 `count`라면, C++ 쪽 메서드 이름은 `get_count()` / `set_count()`가 됩니다. 이 규칙은 프로퍼티 타입과 무관하게 항상 동일합니다. `.slint`의 `int`, `string`, `bool`, `float` 등은 각각 C++의 `int`, `slint::SharedString`, `bool`, `float`에 대응됩니다.

실제로 사용해 보면 다음과 같습니다.

```cpp
#include "counter.h"

int main() {
    auto ui = Counter::create();

    // 현재 값을 읽는다
    int current = ui->get_count(); // 0

    // 값을 5로 설정한다
    ui->set_count(5);

    ui->run();
    return 0;
}
```

`ui->set_count(5)`를 호출하면 곧바로 UI에 반영됩니다. `.slint` 쪽에서 `count`를 참조하는 텍스트나 바인딩이 있다면 자동으로 다시 계산되어 화면이 갱신됩니다. 5장에서 배운 바인딩의 반응성이 C++에서 값을 바꾸는 경우에도 똑같이 적용된다고 이해하면 됩니다. `run()`을 호출하기 전에 `set_count()`로 초기값을 세팅해 두면, 창이 뜨는 순간부터 그 값이 반영된 화면을 보여줄 수 있습니다.

> 프로퍼티가 컴포넌트 내부에 중첩된 다른 요소에 선언되어 있고 최상위로 노출(export)되지 않았다면 C++에서 접근할 수 없습니다. C++에서 다루고 싶은 프로퍼티는 최상위 컴포넌트에 직접 선언하거나, `in-out property` 등으로 명시적으로 노출해야 한다는 점을 기억해 두세요.

## 8.2 콜백을 C++ 쪽에서 연결하기: on_이름()

6장에서는 `.slint` 안에서 콜백을 선언하고 연결하는 방법을 배웠습니다.

```slint
export component Counter inherits Window {
    callback clicked();
}
```

이렇게 선언된 `clicked` 콜백은, `.slint` 파일 내부뿐 아니라 **C++ 쪽에서도** 연결할 수 있습니다. 생성된 클래스에는 `on_clicked()`라는 메서드가 자동으로 추가되며, 여기에 람다(또는 함수 객체)를 전달해 콜백이 발생했을 때 실행할 C++ 코드를 등록합니다.

```cpp
auto ui = Counter::create();

ui->on_clicked([] {
    // 버튼이 클릭될 때마다 실행되는 C++ 로직
    std::cout << "clicked!" << std::endl;
});
```

`.slint` 쪽에서 버튼의 `clicked` 이벤트를 컴포넌트의 `clicked` 콜백에 연결해 두었다면(6장에서 다룬 방식), 사용자가 버튼을 누르는 순간 이 람다가 실행됩니다. 즉 UI에서 발생한 이벤트가 C++ 코드로 전달되는 통로가 바로 `on_이름()`입니다.

람다에서 `ui` 자체나 다른 외부 상태를 참조해야 하는 경우가 많으므로, 캡처는 보통 `[&]`나 필요한 변수를 명시적으로 캡처하는 형태를 씁니다. 다만 `ui`를 참조 캡처할 때는 람다의 수명이 `ui`보다 길어지지 않도록 주의해야 합니다. 일반적으로 `main()` 안에서 `ui`를 만들고 그 자리에서 콜백을 연결한 뒤 `run()`을 호출하는 구조라면 이런 문제가 생기지 않습니다.

```cpp
int main() {
    auto ui = Counter::create();

    ui->on_clicked([&] {
        int current = ui->get_count();
        ui->set_count(current + 1);
    });

    ui->run();
    return 0;
}
```

이 코드는 벌써 이번 장의 핵심 패턴을 보여줍니다. 버튼 클릭 → C++ 콜백 실행 → `get_count()`로 현재 값을 읽고 → `set_count()`로 새 값을 반영. UI 이벤트가 C++ 로직을 거쳐 다시 UI 상태로 돌아오는 왕복이 완성됩니다.

## 8.3 반대 방향: C++에서 UI 콜백을 직접 호출하기 (invoke_이름)

지금까지는 "UI에서 발생한 이벤트를 C++가 받는" 방향이었습니다. 하지만 반대 방향, 즉 **C++ 코드가 먼저 나서서 UI 쪽 콜백을 호출**해야 하는 경우도 있습니다. 예를 들어 백그라운드에서 파일 다운로드가 끝났다거나, 외부 센서 값이 갱신됐다거나 하는, UI 바깥에서 발생한 이벤트를 UI에 알려야 할 때입니다.

이럴 때는 생성된 클래스의 `invoke_이름()` 메서드를 사용합니다. `.slint`에 선언된 콜백을 C++ 쪽에서 마치 함수처럼 직접 호출하는 것입니다.

```slint
export component Counter inherits Window {
    callback notify-updated(int);
}
```

```cpp
// 외부 이벤트(예: 타이머, 백그라운드 스레드 결과 등)가 발생했을 때
ui->invoke_notify_updated(42);
```

`.slint`에서 콜백 이름에 하이픈(`-`)을 썼더라도 C++에서 생성되는 메서드 이름은 언더스코어(`_`)로 변환된다는 점에 유의하세요(`notify-updated` → `invoke_notify_updated`). 이는 콜백뿐 아니라 프로퍼티 이름에서도 동일하게 적용되는 규칙입니다.

`invoke_이름()`을 호출하면, `.slint` 쪽에서 이 콜백에 연결해 둔 핸들러(예: 어떤 애니메이션을 트리거하거나 다른 프로퍼티를 갱신하는 로직)가 실행됩니다. 즉 `on_이름()`이 "UI → C++로 향하는 콜백 연결"이라면, `invoke_이름()`은 "C++ → UI로 향하는 콜백 호출"이라고 대비해서 기억하면 됩니다.

## 8.4 실전 예제: 카운터 앱

지금까지 배운 프로퍼티 접근(`get_`/`set_`)과 콜백 연결(`on_`)을 조합해 완전한 카운터 앱을 만들어 보겠습니다. 버튼을 누르면 C++ 쪽에서 카운트를 증가시키고, 그 결과를 다시 UI 프로퍼티에 반영하는 구조입니다.

프로젝트 구조는 다음과 같습니다.

```
counter_app/
├── CMakeLists.txt
├── main.cpp
└── counter.slint
```

`counter.slint`는 현재 값을 표시하는 텍스트와 버튼 하나로 이루어진 단순한 UI입니다. `count` 프로퍼티와 `clicked` 콜백을 선언해 C++ 쪽과 주고받을 통로를 만듭니다.

```slint
import { Button } from "std-widgets.slint";

export component Counter inherits Window {
    width: 260px;
    height: 160px;
    title: "Counter";

    // C++에서 get_count() / set_count()로 접근할 프로퍼티
    property <int> count: 0;

    // 버튼이 클릭되면 발생시킬 콜백. C++에서 on_clicked()로 연결한다.
    callback clicked();

    VerticalLayout {
        alignment: center;
        spacing: 16px;

        Text {
            text: "count: " + count;
            font-size: 24px;
            horizontal-alignment: center;
        }

        Button {
            text: "증가";
            clicked => { root.clicked(); }
        }
    }
}
```

`CMakeLists.txt`는 7장과 동일한 패턴을 따릅니다.

```cmake
cmake_minimum_required(VERSION 3.21)
project(counter_app LANGUAGES CXX)

find_package(Slint REQUIRED)

add_executable(counter_app main.cpp)
slint_target_sources(counter_app counter.slint)
target_link_libraries(counter_app PRIVATE Slint::Slint)
```

`main.cpp`에서는 카운트 값을 C++ 쪽 변수로도 관리하면서, 버튼 클릭 콜백이 발생할 때마다 이 값을 증가시키고 `set_count()`로 UI에 반영합니다.

```cpp
// main.cpp
#include "counter.h"

int main() {
    auto ui = Counter::create();

    // C++ 쪽에서 별도로 카운트를 관리한다고 가정합니다.
    // (여기서는 단순히 UI 프로퍼티를 그대로 카운터로 쓰지만,
    //  실제로는 파일에 값을 저장하거나 다른 시스템과 동기화하는 등
    //  더 복잡한 로직이 들어갈 수 있는 자리입니다.)
    int counter = 0;

    ui->on_clicked([&] {
        counter += 1;
        ui->set_count(counter);
    });

    ui->run();
    return 0;
}
```

빌드하고 실행하면 "증가" 버튼을 누를 때마다 화면의 숫자가 1씩 늘어납니다. 이 예제의 데이터 흐름을 정리하면 다음과 같습니다.

1. 사용자가 "증가" 버튼을 클릭한다.
2. `.slint` 쪽 `Button`의 `clicked` 핸들러가 `root.clicked()`를 호출해 컴포넌트의 `clicked` 콜백을 발생시킨다. (6장에서 다룬 콜백 발생 패턴)
3. C++에서 `ui->on_clicked(...)`로 등록해 둔 람다가 실행된다.
4. 람다 안에서 C++ 변수 `counter`를 증가시키고, `ui->set_count(counter)`로 그 값을 UI 프로퍼티에 반영한다.
5. `.slint` 쪽의 `Text { text: "count: " + count; }` 바인딩이 `count` 프로퍼티 변경을 감지해 자동으로 다시 그려진다.

이 순환 구조 — **UI 이벤트 → C++ 콜백 → C++ 로직 실행 → 프로퍼티 갱신 → UI 자동 반영** — 는 Slint C++ 애플리케이션에서 가장 기본이 되는 상호작용 패턴입니다. 이후 9장에서 다룰 표준 위젯이나 11장의 리스트 모델도 결국 이 패턴의 확장이라고 볼 수 있습니다.

## 8.5 문자열 다루기: SharedString과 std::string

`.slint`의 `string` 타입 프로퍼티는 C++에서 `int`나 `bool`처럼 표준 타입으로 바로 대응되지 않고, `slint::SharedString`이라는 Slint 전용 문자열 타입으로 노출됩니다. `SharedString`은 여러 곳에서 값을 공유해도 안전하도록 설계된 문자열 타입으로, C++ 표준 라이브러리의 `std::string`과는 별개의 타입입니다.

```slint
export component Greeter inherits Window {
    property <string> message: "Hello";
}
```

```cpp
auto ui = Greeter::create();

slint::SharedString msg = ui->get_message();
```

`std::string`이나 `std::string_view`로 다루던 문자열을 `SharedString`에 넘기거나, 반대로 `SharedString`을 `std::string`으로 바꿔야 하는 경우가 자주 생깁니다. 예를 들어 표준 라이브러리 함수에 값을 넘기려면 `std::string`으로 변환해야 하고, 반대로 계산한 `std::string` 결과를 `set_이름()`에 넘기려면 `SharedString`으로 변환해야 합니다.

```cpp
std::string native = std::string(ui->get_message());   // SharedString → std::string
ui->set_message(slint::SharedString(native + "!"));    // std::string → SharedString
```

두 타입 사이의 변환이 필요하다는 사실만 기억해 두면, 구체적인 변환 방법은 실제로 문자열을 다루는 코드를 작성할 때 그때그때 문서를 확인해도 충분합니다. 지금 단계에서는 "정수/실수/불리언 프로퍼티는 대응하는 C++ 기본 타입으로, 문자열 프로퍼티는 `SharedString`이라는 별도 타입으로 노출된다"는 구분만 정확히 짚고 넘어가면 됩니다.

## 요약

- `.slint`에서 `property <타입> 이름: 초기값;`으로 선언한 프로퍼티는 생성된 C++ 클래스에서 `get_이름()` / `set_이름(값)` 메서드로 노출됩니다. (예: `property <int> count` → `get_count()` / `set_count(int)`)
- `.slint`의 `callback 이름();`은 C++에서 `on_이름([](...){ ... })`으로 연결합니다. UI에서 콜백이 발생하면 등록해 둔 람다가 실행되어 UI 이벤트가 C++ 로직으로 전달됩니다.
- 반대 방향으로, C++ 코드가 UI 쪽 콜백을 직접 트리거하고 싶을 때는 `invoke_이름(인자)`를 호출합니다. 외부에서 발생한 이벤트(타이머, 백그라운드 작업 결과 등)를 UI에 알릴 때 사용합니다.
- 프로퍼티 setter와 콜백 연결을 조합하면 "UI 이벤트 → C++ 콜백 → 로직 실행 → 프로퍼티 갱신 → UI 자동 반영"이라는 기본 상호작용 루프를 구성할 수 있습니다.
- `.slint`의 `string` 프로퍼티는 `slint::SharedString`으로 노출되며, `std::string`/`std::string_view`와는 명시적인 변환이 필요합니다.

## 연습문제

1. `.slint`에 `property <bool> is-enabled: true;`가 선언되어 있을 때, C++에서 이 값을 읽고 `false`로 바꾸는 코드를 작성해 보세요. 프로퍼티 이름의 하이픈이 C++ 메서드 이름에서 어떻게 바뀌는지도 확인해 보세요.
2. `on_이름()`과 `invoke_이름()`의 방향성 차이를 한 문장씩으로 설명해 보세요. 각각 어떤 상황에 적합한지 예를 하나씩 들어 보세요.
3. 8.4절의 카운터 예제에서, "증가" 버튼 옆에 "초기화" 버튼을 추가해 클릭 시 `count`를 0으로 되돌리려면 `.slint`와 `main.cpp`를 각각 어떻게 수정해야 할지 설계해 보세요.
4. `ui->on_clicked([&] { ... })`에서 `ui`를 참조로 캡처하는 람다가 `ui`보다 더 오래 살아남는 상황이 발생하면 어떤 문제가 생길 수 있을지 생각해 보세요.
5. `slint::SharedString`과 `std::string`이 서로 다른 타입으로 존재하는 이유를, Slint 런타임이 UI 트리 전반에서 값을 공유하고 참조 카운팅으로 관리한다는 7장의 내용과 연결지어 추론해 보세요.

---

[◀ 이전: 7장. C++와 Slint 연동 기초](ch07-CPP와Slint연동기초.md) | [📖 목차](00-목차.md) | [다음: 9장. 표준 위젯 라이브러리(std-widgets) ▶](ch09-표준위젯라이브러리.md)
