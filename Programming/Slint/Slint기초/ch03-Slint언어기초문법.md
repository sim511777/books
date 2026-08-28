# 3장. Slint 언어(.slint) 기초 문법

[◀ 이전: 2장. 개발 환경 설정과 첫 창](ch02-개발환경설정과첫창.md) | [📖 목차](00-목차.md) | [다음: 4장. 레이아웃 시스템 ▶](ch04-레이아웃시스템.md)


2장에서 우리는 개발 환경을 갖추고 화면에 첫 창을 띄워 보았습니다. 이번 장에서는 그 창을 채우고 있던 `.slint` 파일의 문법을 하나씩 뜯어보겠습니다. Slint는 UI를 기술하기 위해 만들어진 선언적(declarative) 언어입니다. "버튼을 클릭하면 무슨 일이 일어나야 하는가"를 명령문으로 나열하는 대신, "화면이 어떤 모습이어야 하는가"를 구조로 선언한다는 점이 핵심입니다. 이 관점을 이해하고 나면 Slint의 나머지 문법들은 자연스럽게 따라옵니다.

## 컴포넌트: Slint UI의 기본 단위

Slint에서 모든 UI는 **컴포넌트(component)** 라는 단위로 정의됩니다. 컴포넌트는 화면의 일부 혹은 전체를 나타내는 재사용 가능한 설계도이며, 다음과 같은 문법으로 선언합니다.

```slint
component MyComponent inherits Rectangle {
    // 이 안에 컴포넌트의 내용을 채웁니다
}
```

`component` 키워드 다음에는 컴포넌트 이름(파스칼 케이스, 즉 `MyComponent`처럼 각 단어의 첫 글자를 대문자로 쓰는 것이 관례입니다)이 오고, `inherits` 키워드 다음에는 이 컴포넌트가 어떤 기존 엘리먼트(또는 다른 컴포넌트)의 성질을 물려받는지를 씁니다. 자바나 C++의 클래스 상속과 비슷하게 생각하면 이해하기 쉽습니다. `Rectangle`을 상속하면 이 컴포넌트는 사각형이 가진 배경색, 테두리, 크기 등의 프로퍼티를 기본으로 갖게 됩니다.

애플리케이션의 최상위 컴포넌트, 즉 실제로 화면에 독립된 창(window)으로 나타날 컴포넌트는 보통 `Window`를 상속합니다.

```slint
component AppWindow inherits Window {
    width: 400px;
    height: 300px;
    title: "나의 첫 Slint 앱";
}
```

`Window`를 상속한 컴포넌트는 창의 크기(`width`, `height`), 제목(`title`) 같은 창 고유의 프로퍼티를 사용할 수 있습니다. 하나의 `.slint` 파일 안에는 여러 개의 컴포넌트를 선언할 수 있고, 그중 하나를 C++ 쪽에서 최상위 창으로 인스턴스화하게 됩니다. 이 부분은 7장에서 C++ 연동을 다룰 때 자세히 살펴보겠습니다.

지금 단계에서 기억해 둘 것은 간단합니다. **모든 Slint UI는 `component 이름 inherits 부모 { ... }` 형태로 시작하고, 중괄호 안에 그 컴포넌트를 구성하는 내용을 채워 넣는다**는 점입니다.

## 기본 엘리먼트 살펴보기

컴포넌트 안을 채우는 재료가 되는 것이 **엘리먼트(element)** 입니다. Slint는 여러 내장 엘리먼트를 제공하는데, 이번 장에서는 가장 기본이 되는 세 가지, `Rectangle`, `Text`, `Image`를 소개합니다.

### Rectangle: 사각 영역과 컨테이너

`Rectangle`은 이름 그대로 배경색과 테두리를 가진 사각형 영역을 그립니다.

```slint
component ColorBox inherits Window {
    width: 200px;
    height: 200px;

    Rectangle {
        width: 100px;
        height: 100px;
        background: #3498db;
        border-width: 2px;
        border-color: #2c3e50;
        border-radius: 8px;
    }
}
```

`background`는 배경색, `border-width`와 `border-color`는 테두리의 두께와 색, `border-radius`는 모서리를 둥글게 깎는 정도를 지정합니다. 이 예제만 보면 `Rectangle`이 단순히 색칠된 도형처럼 보이지만, 실무에서는 이보다 훨씬 자주 쓰이는 역할이 하나 더 있습니다. 바로 **다른 엘리먼트를 담는 컨테이너**로 쓰이는 것입니다.

```slint
component Card inherits Window {
    width: 300px;
    height: 150px;

    Rectangle {
        background: #ffffff;
        border-width: 1px;
        border-color: #dddddd;

        Text {
            text: "카드 안의 텍스트";
            color: #333333;
        }
    }
}
```

이렇게 `Rectangle` 중괄호 안에 `Text`를 넣으면, `Text`는 `Rectangle`이 그려 주는 배경과 테두리 위에 자식으로서 놓이게 됩니다. HTML의 `<div>`가 다른 태그를 감싸는 컨테이너로 흔히 쓰이는 것과 비슷한 감각입니다.

### Text: 문자열 표시

`Text`는 화면에 글자를 표시하는 엘리먼트입니다.

```slint
Text {
    text: "안녕하세요, Slint!";
    font-size: 24px;
    color: Colors.darkslategray;
}
```

`text` 프로퍼티에는 표시할 문자열을, `font-size`에는 글자 크기를, `color`에는 글자 색을 지정합니다. 세 프로퍼티 모두 이후 장들에서 계속 마주치게 될 만큼 자주 쓰입니다. 특히 `text` 프로퍼티는 5장에서 살펴볼 바인딩과 결합하면 프로그램 실행 중에 동적으로 바뀌는 값을 표시하는 데 쓰이게 됩니다.

### Image: 이미지 표시

`Image`는 이미지 파일을 화면에 표시합니다.

```slint
Image {
    source: @image-url("./images/logo.png");
    width: 64px;
    height: 64px;
}
```

`source` 프로퍼티에 이미지 파일의 경로를 지정하는데, `@image-url("경로")` 형태의 특수한 구문을 사용합니다. 이 구문은 지정한 경로의 이미지 파일을 컴파일 시점에 리소스로 포함시키라는 의미입니다. 이미지를 다루는 더 자세한 내용, 이를테면 회전이나 크롭 같은 처리는 16장에서 다시 다룹니다. 지금은 "이미지를 넣으려면 `Image` 엘리먼트에 `@image-url`로 경로를 지정한다"는 패턴만 기억해 두면 충분합니다.

## 엘리먼트를 중첩해서 트리 구조 만들기

지금까지 본 예제들에서 이미 눈치챘겠지만, Slint UI는 엘리먼트를 중괄호 안에 중괄호로 계속 중첩시켜 **트리 구조**를 만드는 방식으로 구성됩니다. 바깥쪽 엘리먼트가 부모가 되고, 그 안에 선언된 엘리먼트들이 자식이 됩니다.

```slint
component Profile inherits Window {
    width: 320px;
    height: 200px;

    Rectangle {
        background: #f5f5f5;

        Rectangle {
            width: 80px;
            height: 80px;
            background: #cccccc;
            border-radius: 40px;

            Image {
                source: @image-url("./images/avatar.png");
                width: 80px;
                height: 80px;
            }
        }

        Text {
            text: "김철수";
            font-size: 18px;
        }

        Text {
            text: "소프트웨어 엔지니어";
            font-size: 12px;
            color: #888888;
        }
    }
}
```

이 예제에서 `Profile` 창 안에 배경용 `Rectangle`이 하나 있고, 그 `Rectangle` 안에 다시 원형 아바타를 만드는 또 다른 `Rectangle`(그리고 그 안의 `Image`)과 이름·직함을 표시하는 두 개의 `Text`가 자식으로 들어 있습니다.

여기서 중요한 점은, **이 부모-자식 관계가 단순히 코드의 들여쓰기 구조에 그치지 않고 실제 화면의 시각적 배치에도 영향을 준다**는 것입니다. 자식 엘리먼트는 기본적으로 부모 엘리먼트의 영역 안에 배치되며, 부모가 이동하거나 크기가 바뀌면 자식도 함께 영향을 받습니다. 다만 이 예제처럼 자식들을 한 좌표에 그냥 나열만 하면 서로 겹쳐서 그려집니다. 여러 자식을 세로나 가로로 줄 세우거나 격자로 배치하려면 레이아웃 컨테이너가 필요한데, 이는 4장에서 본격적으로 다룰 주제입니다. 지금은 "엘리먼트 안에 엘리먼트를 넣으면 부모-자식 트리가 되고, 이것이 UI 구조의 뼈대가 된다"는 감각을 잡는 것으로 충분합니다.

## 프로퍼티 선언 기초

지금까지 예제에서 `width`, `background`, `text`처럼 엘리먼트가 이미 갖고 있는 프로퍼티에 값을 대입하는 것을 봤습니다. Slint는 여기서 한 걸음 더 나아가, 컴포넌트에 **나만의 프로퍼티**를 선언할 수 있게 해 줍니다.

```slint
component Greeting inherits Window {
    width: 300px;
    height: 100px;

    property <string> user-name: "손님";

    Text {
        text: "환영합니다, " + user-name;
        font-size: 20px;
    }
}
```

`property <타입> 이름: 기본값;` 형태로 프로퍼티를 선언합니다. 위 예제에서는 `string` 타입의 `user-name`이라는 프로퍼티를 선언하고 기본값으로 `"손님"`을 지정했습니다. 이렇게 선언한 프로퍼티는 같은 컴포넌트 안 어디서든 이름으로 참조할 수 있으며, 위처럼 문자열 결합(`+`)에도 사용할 수 있습니다.

Slint가 지원하는 기본 타입에는 `int`(정수), `float`(실수), `string`(문자열), `bool`(참/거짓), `color`(색상), `length`(길이, 예: `px` 단위) 등이 있습니다. 몇 가지를 함께 선언해 보면 다음과 같습니다.

```slint
component Counter inherits Window {
    width: 200px;
    height: 100px;

    property <int> count: 0;
    property <bool> is-active: true;
    property <color> highlight: #e67e22;

    Text {
        text: "카운트: " + count;
        color: is-active ? highlight : #999999;
    }
}
```

프로퍼티 이름에는 단어 사이를 하이픈(`-`)으로 구분하는 케밥 케이스(kebab-case)를 쓰는 것이 Slint의 관례입니다(`user-name`, `is-active`처럼). C++ 쪽 코드와 연동할 때는 이 하이픈이 자동으로 밑줄(`_`)로 변환된다는 점만 미리 알아 두면 됩니다.

지금 소개한 것은 프로퍼티 문법의 입구일 뿐입니다. 프로퍼티에 값을 계산식으로 바인딩하거나, 다른 프로퍼티가 바뀔 때 자동으로 재계산되게 하는 등의 본격적인 활용은 5장 "프로퍼티와 바인딩"에서 자세히 다룹니다. 지금은 `property <타입> 이름: 기본값;`이라는 선언 문법 자체와, 선언한 프로퍼티를 같은 컴포넌트 안에서 참조할 수 있다는 사실만 기억해 두세요.

## 색상과 길이 표기법

앞의 예제들에서 이미 여러 차례 등장했지만, Slint에서 색상과 길이를 표기하는 방법을 한 번 정리하고 넘어가겠습니다.

### 색상

Slint에서 색상은 두 가지 방식으로 표기할 수 있습니다. 첫째는 웹에서 흔히 쓰는 16진수 표기법입니다.

```slint
Rectangle {
    background: #3498db;   // RGB
    border-color: #2c3e50ff; // RGBA(투명도 포함)
}
```

`#RRGGBB` 형태로 빨강, 초록, 파랑 각 채널을 16진수 두 자리씩 표기하며, 뒤에 두 자리를 더 붙여 `#RRGGBBAA`처럼 투명도(alpha)까지 지정할 수도 있습니다.

둘째는 `Colors` 네임스페이스를 통해 이름이 있는 색상을 사용하는 방식입니다.

```slint
Text {
    color: Colors.red;
}

Rectangle {
    background: Colors.lightblue;
}
```

`Colors.red`, `Colors.blue`, `Colors.lightblue`처럼 널리 알려진 색상 이름을 그대로 쓸 수 있어, 정확한 16진수 값이 중요하지 않은 프로토타이핑 단계에서 특히 편리합니다.

### 길이 단위

너비, 높이, 글자 크기처럼 화면상의 길이를 나타낼 때는 `px` 단위를 사용합니다.

```slint
Rectangle {
    width: 200px;
    height: 100px;
}

Text {
    font-size: 16px;
}
```

`px`는 픽셀을 의미하며, Slint에서 가장 흔히 쓰이는 길이 단위입니다. 지금 단계에서는 "길이가 필요한 곳에는 숫자 뒤에 `px`를 붙인다"는 정도만 기억해 두면 됩니다. 창 크기에 따라 자동으로 늘어나거나 줄어드는 유연한 크기 조정은 4장의 레이아웃 시스템에서 `horizontal-stretch`, `vertical-stretch` 같은 프로퍼티와 함께 다룹니다.

## 정리하며

이번 장에서 배운 내용을 하나의 예제로 종합해 보겠습니다.

```slint
component InfoPanel inherits Window {
    width: 320px;
    height: 160px;
    title: "정보 패널";

    property <string> message: "Slint를 배우는 중입니다";
    property <int> progress: 42;

    Rectangle {
        background: #ecf0f1;
        border-width: 1px;
        border-color: #bdc3c7;
        border-radius: 6px;

        Text {
            text: message;
            font-size: 18px;
            color: Colors.darkslategray;
        }

        Text {
            text: "진행률: " + progress + "%";
            font-size: 14px;
            color: #7f8c8d;
        }

        Image {
            source: @image-url("./images/icon.png");
            width: 32px;
            height: 32px;
        }
    }
}
```

`Window`를 상속하는 최상위 컴포넌트, 배경 역할과 컨테이너 역할을 겸하는 `Rectangle`, 문자열을 보여 주는 `Text` 두 개, 아이콘을 보여 주는 `Image`, 그리고 컴포넌트 고유의 프로퍼티 `message`와 `progress`까지 이번 장에서 다룬 요소들이 모두 들어 있습니다.

## 요약

- Slint UI는 `component 이름 inherits 부모타입 { ... }` 문법으로 정의되는 컴포넌트를 기본 단위로 삼습니다. 최상위 컴포넌트는 보통 `Window`를 상속합니다.
- `Rectangle`은 배경과 테두리를 가진 사각 영역을 그리며, 다른 엘리먼트를 담는 컨테이너로도 흔히 사용됩니다.
- `Text`는 `text`, `font-size`, `color` 프로퍼티로 문자열을 표시하고, `Image`는 `source` 프로퍼티(`@image-url("경로")`)로 이미지 파일을 표시합니다.
- 엘리먼트를 중괄호 안에 중첩시키면 부모-자식 트리 구조가 만들어지며, 이 관계는 화면의 시각적 배치에도 직접 영향을 줍니다.
- `property <타입> 이름: 기본값;` 문법으로 컴포넌트에 커스텀 프로퍼티를 선언할 수 있습니다. 바인딩을 통한 본격적인 활용은 5장에서 다룹니다.
- 색상은 `#RRGGBB`(또는 `#RRGGBBAA`) 형태나 `Colors.red`처럼 이름 있는 색상으로 표기하고, 길이는 `px` 단위를 사용합니다.

## 연습문제

1. `Window`를 상속하는 `MyApp`이라는 이름의 컴포넌트를 선언하고, 너비 400px, 높이 250px, 제목을 "연습문제"로 지정해 보세요.
2. 위에서 만든 `MyApp` 안에 배경색이 `#2c3e50`인 `Rectangle`을 하나 넣고, 그 안에 `"Hello, Slint!"`라는 텍스트를 글자 크기 20px, 색상 `Colors.white`로 표시하는 `Text`를 넣어 보세요.
3. `string` 타입의 프로퍼티 `app-title`을 선언하고 기본값으로 원하는 문자열을 지정한 뒤, `Text`의 `text` 프로퍼티에서 이 값을 참조하도록 바꿔 보세요.
4. `Rectangle` 안에 `Rectangle`을 중첩시키고, 다시 그 안에 `Text`를 넣어 3단 깊이의 엘리먼트 트리를 만들어 보세요. 부모 `Rectangle`의 배경색을 바꾸면 화면에서 어떤 부분이 영향을 받는지 관찰해 보세요.
5. `int` 타입의 프로퍼티와 `bool` 타입의 프로퍼티를 각각 하나씩 선언하고, `Text`의 `text` 프로퍼티에서 정수 프로퍼티를 문자열과 결합해 표시해 보세요.

---

[◀ 이전: 2장. 개발 환경 설정과 첫 창](ch02-개발환경설정과첫창.md) | [📖 목차](00-목차.md) | [다음: 4장. 레이아웃 시스템 ▶](ch04-레이아웃시스템.md)
