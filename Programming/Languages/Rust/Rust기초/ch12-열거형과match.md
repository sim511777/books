# 12장. 열거형과 match

[◀ 이전: 11장. 구조체](ch11-구조체.md) | [📖 목차](00-목차.md) | [다음: 13장. 트레이트(Trait) ▶](ch13-트레이트.md)

---

11장에서는 여러 값을 "모두" 묶어서 표현하는 구조체를 배웠습니다. 그런데 현실에는 값이 여러 경우 "중 하나"만 될 수 있는 상황도 많습니다. 신호등은 빨강, 노랑, 초록 중 하나이고, 네트워크 요청은 성공 또는 실패 중 하나이며, 어떤 값은 존재하거나 존재하지 않을 수 있습니다. 이런 "여러 가능성 중 정확히 하나"를 표현하기 위한 도구가 열거형(`enum`)입니다. 이번 장에서는 열거형을 정의하는 방법, 각 변형(variant)에 데이터를 담는 방법, 그리고 열거형의 모든 경우를 빠짐없이 처리하는 `match` 표현식을 배웁니다.

## enum으로 여러 경우 중 하나 표현하기

열거형은 `enum` 키워드로 정의하며, 가능한 값들을 변형(variant)이라는 이름으로 나열합니다.

```rust
// 신호등의 상태는 세 가지 중 하나다
enum TrafficLight {
    Red,
    Yellow,
    Green,
}

fn main() {
    let light = TrafficLight::Red;

    // 변형은 타입 이름과 :: 로 접근한다
    describe(&light);
}

fn describe(light: &TrafficLight) {
    // 지금은 이렇게만 확인하지만, 곧 match로 더 깔끔하게 처리할 것이다
    match light {
        TrafficLight::Red => println!("정지"),
        TrafficLight::Yellow => println!("주의"),
        TrafficLight::Green => println!("진행"),
    }
}
```

구조체와의 차이를 분명히 짚어 두겠습니다. `Rectangle { width, height }`는 너비"와" 높이를 동시에 가지지만, `TrafficLight` 값은 `Red`, `Yellow`, `Green` 중 정확히 하나입니다. 두 가지가 동시에 성립하는 값이 아니라, 셋 중 하나로만 존재할 수 있는 값을 표현할 때 열거형을 씁니다.

## 변형이 데이터를 가지는 경우

열거형의 강력함은 각 변형이 자신만의 데이터를 가질 수 있다는 점에서 나옵니다. 변형마다 서로 다른 타입과 개수의 데이터를 담을 수 있습니다.

```rust
// 도형의 종류에 따라 필요한 데이터가 다르다
enum Shape {
    Circle(f64),           // 반지름 하나
    Rectangle(f64, f64),   // 너비, 높이
    Triangle(f64, f64, f64), // 세 변의 길이
}

fn main() {
    let shapes = vec![
        Shape::Circle(3.0),
        Shape::Rectangle(4.0, 5.0),
        Shape::Triangle(3.0, 4.0, 5.0),
    ];

    for shape in &shapes {
        println!("면적: {:.2}", area(shape));
    }
}

fn area(shape: &Shape) -> f64 {
    match shape {
        Shape::Circle(radius) => std::f64::consts::PI * radius * radius,
        Shape::Rectangle(width, height) => width * height,
        Shape::Triangle(a, b, c) => {
            // 헤론의 공식
            let s = (a + b + c) / 2.0;
            (s * (s - a) * (s - b) * (s - c)).sqrt()
        }
    }
}
```

이 예제에서 `Circle`, `Rectangle`, `Triangle`은 모두 같은 타입인 `Shape`에 속하지만, 각자 다른 개수와 종류의 데이터를 담고 있습니다. 이렇게 하나의 타입으로 "종류가 다른 데이터 묶음들"을 표현할 수 있다는 점이 구조체 하나로는 하기 어려운 열거형만의 특징입니다.

튜플처럼 위치로 데이터를 담을 수도 있지만, 구조체처럼 이름 있는 필드를 가진 변형도 만들 수 있습니다.

```rust
enum Message {
    Quit,                       // 데이터 없음
    Move { x: i32, y: i32 },    // 이름 있는 필드
    Write(String),              // 값 하나
    ChangeColor(u8, u8, u8),    // 값 여러 개
}
```

`Message`는 한 열거형 안에 데이터가 전혀 없는 변형, 이름 있는 필드를 가진 변형, 값 하나만 담는 변형, 여러 값을 담는 변형을 모두 섞어 담고 있습니다. 이것만으로도 열거형이 얼마나 다양한 모양의 데이터를 하나의 타입으로 묶어낼 수 있는지 알 수 있습니다.

## match로 모든 경우를 빠짐없이 처리하기

5장에서 `match`를 간단히 맛보았는데, 이제 열거형과 함께 본격적으로 다뤄보겠습니다. `match`는 값을 여러 패턴과 비교해 처음으로 일치하는 갈래(arm)의 코드를 실행하는 표현식입니다.

```rust
enum Coin {
    Penny,
    Nickel,
    Dime,
    Quarter,
}

fn value_in_cents(coin: &Coin) -> u32 {
    match coin {
        Coin::Penny => 1,
        Coin::Nickel => 5,
        Coin::Dime => 10,
        Coin::Quarter => 25,
    }
}

fn main() {
    let coin = Coin::Dime;
    println!("가치: {}센트", value_in_cents(&coin));
}
```

`match`의 가장 중요한 특징은 컴파일러가 모든 경우가 처리되었는지 검사한다는 점입니다. 만약 `Coin::Quarter` 갈래를 빠뜨리면 컴파일 오류가 발생합니다. 이를 "빠짐없음(exhaustiveness)" 검사라고 부르는데, `if`/`else` 연쇄와 비교했을 때 `match`가 갖는 큰 장점입니다. 새로운 변형을 열거형에 추가했는데 처리를 잊은 `match`가 있다면, 실행 전에 컴파일 단계에서 바로 알려줍니다.

데이터를 담은 변형은 `match`의 각 갈래에서 그 데이터를 이름에 바인딩해서 꺼내 쓸 수 있습니다.

```rust
enum Message {
    Quit,
    Move { x: i32, y: i32 },
    Write(String),
    ChangeColor(u8, u8, u8),
}

fn process(msg: &Message) {
    match msg {
        Message::Quit => println!("종료 요청"),
        // 이름 있는 필드는 그대로 이름으로 바인딩한다
        Message::Move { x, y } => println!("이동: ({}, {})", x, y),
        Message::Write(text) => println!("쓰기: {}", text),
        Message::ChangeColor(r, g, b) => println!("색상 변경: ({}, {}, {})", r, g, b),
    }
}

fn main() {
    let messages = vec![
        Message::Move { x: 10, y: 20 },
        Message::Write(String::from("안녕하세요")),
        Message::ChangeColor(255, 0, 0),
        Message::Quit,
    ];

    for msg in &messages {
        process(msg);
    }
}
```

`match`는 표현식이므로 값을 반환할 수도 있습니다. 각 갈래가 같은 타입의 값을 만들어내면, 그 값을 변수에 바로 대입할 수 있습니다.

```rust
fn value_in_cents(coin: &Coin) -> u32 {
    // 각 갈래의 결과가 match 표현식 전체의 값이 된다
    let cents = match coin {
        Coin::Penny => 1,
        Coin::Nickel => 5,
        Coin::Dime => 10,
        Coin::Quarter => 25,
    };
    cents
}
```

## 와일드카드 패턴: _

변형이 많은 열거형에서 몇 가지 경우만 특별히 처리하고 나머지는 한꺼번에 처리하고 싶을 때가 있습니다. 이때 와일드카드 패턴 `_`를 마지막 갈래로 사용합니다.

```rust
enum Direction {
    North,
    South,
    East,
    West,
}

fn describe(dir: &Direction) -> &str {
    match dir {
        Direction::North => "북쪽으로 이동",
        // 나머지 모든 경우를 한 번에 처리
        _ => "다른 방향으로 이동",
    }
}
```

`_`는 값을 바인딩하지 않고 그냥 버린다는 뜻입니다. 값을 바인딩해서 쓰고 싶다면 `_` 대신 다른 변수 이름(예: `other`)을 사용해도 되지만, 그 값을 실제로 쓰지 않는다면 컴파일러가 "사용하지 않는 변수" 경고를 낼 수 있으므로 이런 경우엔 `_`가 관용적입니다.

주의할 점은 `_`를 넣으면 빠짐없음 검사가 사실상 무력화된다는 것입니다. 나중에 `Direction`에 새로운 변형을 추가해도 `_` 갈래가 조용히 그 경우를 흡수해버리므로, 컴파일러가 "새 변형을 처리하지 않았다"고 알려주지 않습니다. 그래서 모든 변형을 의도적으로 각각 처리해야 하는 코드라면 `_`를 쓰지 않고 각 변형을 명시적으로 나열하는 편이 더 안전합니다.

숫자나 문자 같은 값에도 `match`를 쓸 수 있는데, 이때도 `_`는 "그 외 모든 값"을 뜻합니다.

```rust
fn describe_number(n: i32) -> &'static str {
    match n {
        0 => "영",
        1..=9 => "한 자리 양수", // 범위 패턴
        _ => "그 외의 값",
    }
}
```

## Option<T>: 표준 라이브러리의 열거형

지금까지 다룬 열거형은 모두 우리가 직접 만든 것이었지만, Rust 표준 라이브러리도 열거형을 적극적으로 활용합니다. 그중 가장 자주 마주치는 것이 `Option<T>`입니다. `Option<T>`는 대략 다음과 같이 정의되어 있습니다.

```rust
enum Option<T> {
    Some(T), // 값이 있는 경우, T 타입의 값을 담는다
    None,    // 값이 없는 경우
}
```

`<T>`는 14장에서 다룰 제네릭 문법으로, "어떤 타입이든 될 수 있다"는 뜻입니다. 지금은 `Option<i32>`가 "`i32` 값이 있을 수도 있고 없을 수도 있다"는 뜻이라는 정도로만 이해하면 충분합니다.

```rust
fn find_first_even(numbers: &[i32]) -> Option<i32> {
    for &n in numbers {
        if n % 2 == 0 {
            return Some(n);
        }
    }
    None
}

fn main() {
    let numbers = vec![1, 3, 5, 8, 9];

    match find_first_even(&numbers) {
        Some(n) => println!("첫 짝수: {}", n),
        None => println!("짝수를 찾지 못했습니다"),
    }
}
```

`Option<T>`가 중요한 이유는, 값이 "없을 수도 있다"는 사실을 타입 시스템에 드러낸다는 점입니다. 다른 많은 언어에서는 값이 없다는 것을 `null`이나 `nil`로 표현하는데, 이 값을 확인하지 않고 그냥 사용하려다 실행 중 오류가 나는 경우가 흔합니다. Rust에서는 `Option<i32>`와 `i32`가 서로 다른 타입이기 때문에, `Option<i32>` 안의 값을 실제로 쓰려면 `match`나 그 밖의 방법으로 반드시 `Some`과 `None`을 구분해서 처리해야 합니다. 즉 "값이 없을 가능성"을 깜빡 잊고 넘어가는 실수를 컴파일 단계에서 막아주는 것입니다.

`Option<T>`는 이후 15장에서 다룰 에러 처리와도 밀접하게 연결됩니다. 실패할 수 있는 연산의 결과를 표현하는 `Result<T, E>` 역시 `Option<T>`과 마찬가지로 여러 변형을 가진 열거형이며, `match`로 각 경우를 처리하는 방식도 동일합니다. 지금 익힌 `enum`과 `match`에 대한 이해는 그때 그대로 다시 쓰이게 됩니다.

## 요약

이번 장에서는 여러 경우 중 정확히 하나를 표현하는 `enum`을 배웠습니다. 구조체가 여러 값을 "모두" 묶는 것과 달리, 열거형은 여러 가능성 "중 하나"를 표현하며, 각 변형은 튜플처럼 값 여러 개를 담거나, 구조체처럼 이름 있는 필드를 가지거나, 아예 데이터가 없을 수도 있었습니다. `match` 표현식으로는 열거형의 모든 변형을 하나씩 처리했고, 컴파일러가 빠짐없음을 검사해준다는 것이 `if`/`else`보다 안전한 이유였습니다. 와일드카드 패턴 `_`는 나머지 경우를 한 번에 처리하되, 빠짐없음 검사를 약화시킨다는 점도 함께 짚었습니다. 마지막으로 표준 라이브러리의 `Option<T>`가 `Some(T)`와 `None`이라는 두 변형을 가진 평범한 열거형이라는 것을 확인했고, 이는 15장의 에러 처리로 이어지는 중요한 발판입니다. 다음 장에서는 서로 다른 타입들이 공통된 동작을 공유하도록 만들어주는 트레이트를 배웁니다.

## 연습문제

1. 카드 게임의 무늬를 나타내는 `Suit` 열거형(`Heart`, `Diamond`, `Club`, `Spade`)을 정의하고, `match`를 이용해 각 무늬의 이름을 한글 문자열로 반환하는 함수를 작성하세요.
2. 웹 요청의 결과를 나타내는 `Response` 열거형을 정의하세요. `Success(String)`은 응답 본문을, `Error(u32, String)`은 상태 코드와 오류 메시지를, `Timeout`은 아무 데이터도 갖지 않도록 만든 뒤, `match`로 각 경우를 다르게 출력하는 함수를 작성하세요.
3. `Option<i32>`를 반환하는 함수 `divide(a: i32, b: i32) -> Option<i32>`를 작성하세요. `b`가 0이면 `None`을, 아니면 `Some(a / b)`를 반환해야 합니다. `main`에서 `match`로 결과를 처리해 보세요.
4. 요일을 나타내는 열거형 `Weekday`(월요일부터 일요일까지 7개 변형)를 정의하고, `match`와 와일드카드 `_`를 이용해 "월요일"이면 특별한 메시지를, 그 외의 요일은 공통된 메시지를 출력하는 함수를 작성하세요. 이후 왜 이 코드에서는 `_`를 쓰지 않고 7개 변형을 모두 나열하는 것이 더 안전할 수 있는지 생각해 보세요.
5. `Shape` 열거형(`Circle(f64)`, `Rectangle(f64, f64)`)에 `impl` 블록을 추가하고, `area(&self) -> f64` 메서드 안에서 `match self { ... }`를 이용해 면적을 계산하도록 11장에서 배운 메서드 문법과 결합해 보세요.

---

[◀ 이전: 11장. 구조체](ch11-구조체.md) | [📖 목차](00-목차.md) | [다음: 13장. 트레이트(Trait) ▶](ch13-트레이트.md)
