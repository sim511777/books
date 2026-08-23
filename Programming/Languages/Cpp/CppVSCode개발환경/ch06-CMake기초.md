# 6장. CMake 기초

[◀ 이전: 5장. IntelliSense 설정](ch05-IntelliSense설정.md) | [📖 목차](00-목차.md) | [다음: 7장. CMake 프로젝트 구조화 ▶](ch07-CMake프로젝트구조화.md)


2장에서는 `g++` 명령을 직접 입력해 소스 파일을 컴파일했습니다. 파일이 한두 개일 때는 이 방법으로 충분하지만, 프로젝트가 커지고 여러 소스 파일과 외부 라이브러리가 얽히기 시작하면 매번 긴 컴파일 명령을 손으로 입력하는 것은 현실적이지 않습니다. 이 장에서는 이런 문제를 해결하기 위해 널리 쓰이는 도구인 CMake의 기초를 다룹니다.

## CMake는 컴파일러가 아니다

CMake를 처음 접할 때 가장 흔한 오해는 CMake 자체가 코드를 컴파일해 준다고 생각하는 것입니다. 하지만 CMake는 컴파일러가 아닙니다. CMake는 **빌드 시스템을 생성하는 도구**입니다.

이 말의 의미를 조금 더 풀어보겠습니다. 2장에서 사용한 `g++`은 소스 코드를 직접 읽어서 기계어로 번역하는 컴파일러였습니다. 반면 CMake는 소스 코드를 직접 컴파일하지 않습니다. 대신 `CMakeLists.txt`라는 설정 파일을 읽고, 그 내용을 바탕으로 Make나 Ninja(8장에서 다룹니다) 같은 실제 빌드 도구가 사용할 빌드 스크립트(예: `Makefile`이나 `build.ninja`)를 만들어냅니다. 그리고 이 빌드 스크립트를 실행하는 단계에서 비로소 `g++`이나 `clang++` 같은 실제 컴파일러가 호출됩니다.

즉 전체 흐름을 정리하면 다음과 같은 계층 구조가 됩니다.

```
CMakeLists.txt (내가 작성하는 설정)
        ↓  CMake가 읽고 처리
Makefile 또는 build.ninja (자동 생성되는 빌드 스크립트)
        ↓  Make 또는 Ninja가 읽고 처리
g++ 호출 (실제 컴파일 수행)
        ↓
실행 파일
```

CMake는 이 흐름에서 맨 위, 즉 "어떤 빌드 도구를 쓰든 그 빌드 도구가 이해할 수 있는 스크립트를 만들어주는" 역할을 담당합니다. 이런 성격 때문에 CMake를 "메타 빌드 시스템" 또는 "빌드 시스템 생성기"라고 부릅니다.

## CMakeLists.txt의 기본 구조

CMake 프로젝트는 `CMakeLists.txt`라는 이름의 파일 하나로 시작합니다. 이 파일은 프로젝트 루트 디렉터리에 위치하며, CMake는 이 파일에 적힌 명령들을 순서대로 해석해 빌드 설정을 구성합니다.

가장 단순한 형태의 `CMakeLists.txt`는 다음과 같습니다.

```cmake
cmake_minimum_required(VERSION 3.20)

project(HelloCMake)

add_executable(hello main.cpp)
```

세 줄이 하는 일을 하나씩 살펴보겠습니다.

### cmake_minimum_required

```cmake
cmake_minimum_required(VERSION 3.20)
```

이 프로젝트를 처리하는 데 필요한 CMake의 최소 버전을 지정합니다. CMake는 버전마다 새로운 명령이나 문법이 추가되기 때문에, 이 파일을 작성할 때 사용한 기능이 어느 버전부터 지원되는지 명시해 두는 것입니다. 설치된 CMake 버전이 이보다 낮으면 CMake는 즉시 오류를 내고 실행을 중단합니다. 관례적으로 `CMakeLists.txt`의 가장 첫 줄에 위치합니다.

### project()

```cmake
project(HelloCMake)
```

프로젝트의 이름을 선언합니다. 이 명령은 단순히 이름만 정하는 것이 아니라, 내부적으로 컴파일러를 탐지하고 여러 변수(`PROJECT_NAME`, `PROJECT_SOURCE_DIR` 등)를 설정하는 역할도 합니다. 필요하다면 버전이나 사용 언어도 함께 지정할 수 있습니다.

```cmake
project(HelloCMake VERSION 1.0 LANGUAGES CXX)
```

`LANGUAGES CXX`는 이 프로젝트가 C++ 언어를 사용한다는 것을 명시합니다. 생략하면 CMake는 기본적으로 C와 C++을 모두 사용한다고 가정합니다.

### add_executable()

```cmake
add_executable(hello main.cpp)
```

`hello`라는 이름의 실행 파일을 만들되, 그 실행 파일은 `main.cpp`를 컴파일해서 만든다는 것을 선언합니다. 첫 번째 인자는 만들어질 타깃(target)의 이름이고, 그 뒤로 나열되는 인자들은 이 타깃을 구성하는 소스 파일 목록입니다. 소스 파일이 여러 개라면 공백으로 구분해 나열합니다.

```cmake
add_executable(hello main.cpp utils.cpp parser.cpp)
```

여기서 `hello`는 실제 실행 파일의 이름(윈도우에서는 `hello.exe`)이 됩니다. `add_executable`로 정의된 것을 CMake 용어로 "타깃"이라고 부르며, 이후 7장에서 다룰 `target_include_directories()`나 `target_link_libraries()` 같은 명령들은 모두 이 타깃 이름을 대상으로 세부 설정을 추가하는 방식으로 동작합니다.

## CMake의 3단계 흐름: Configure, Generate, Build

CMake를 사용해 프로젝트를 빌드하는 과정은 크게 세 단계로 나뉩니다.

1. **Configure(설정)**: `CMakeLists.txt`를 읽고 해석하는 단계입니다. 컴파일러를 탐지하고, 변수들을 계산하고, 프로젝트 구조를 파악합니다. 이 단계의 결과는 `CMakeCache.txt`라는 파일에 저장됩니다.
2. **Generate(생성)**: Configure 단계에서 파악한 정보를 바탕으로 실제 빌드 시스템 파일(Makefile 또는 `build.ninja` 등)을 만들어내는 단계입니다.
3. **Build(빌드)**: Generate 단계에서 만들어진 빌드 시스템 파일을 실제 빌드 도구(Make, Ninja 등)로 실행해서 컴파일러를 호출하고, 최종 실행 파일을 만드는 단계입니다.

이 세 단계 중 Configure와 Generate는 하나의 명령으로 함께 처리되고, Build는 별도의 명령으로 실행합니다.

### 실습: cmake -B build

앞서 작성한 `CMakeLists.txt`와 `main.cpp`가 같은 디렉터리에 있다고 가정하고, 다음 명령을 실행해 보겠습니다.

```bash
cmake -B build
```

`-B build`는 "build라는 이름의 디렉터리를 만들어서 그 안에 빌드 관련 파일들을 생성하라"는 의미입니다. 이 명령을 실행하면 CMake는 다음과 같은 일을 수행합니다.

- 현재 디렉터리(기본값)에서 `CMakeLists.txt`를 찾아 읽습니다.
- 시스템에 설치된 컴파일러를 탐지합니다.
- `build` 디렉터리를 생성하고, 그 안에 `CMakeCache.txt`와 `Makefile`(또는 다른 빌드 시스템 파일)을 생성합니다.

명령이 성공하면 `build` 디렉터리 안에 다음과 같은 파일들이 만들어진 것을 확인할 수 있습니다.

```bash
ls build
```

여기서 `CMakeCache.txt`, `CMakeFiles/`, 그리고 (기본 생성기가 Make 계열이라면) `Makefile`이 보일 것입니다.

이렇게 소스 코드와 별도의 디렉터리에 빌드 결과물을 만드는 방식을 "out-of-source build"라고 부릅니다. 소스 디렉터리 자체를 빌드 산출물로 어지럽히지 않고, `build` 디렉터리 하나만 삭제하면 언제든 처음 상태로 되돌릴 수 있다는 장점이 있습니다.

### 실습: cmake --build build

빌드 시스템 파일이 생성되었으니, 이제 실제로 컴파일을 수행할 차례입니다.

```bash
cmake --build build
```

이 명령은 `build` 디렉터리 안에 생성된 빌드 시스템(Makefile 등)을 실행해서, 실제로 `g++`을 호출해 컴파일과 링크를 수행합니다. 명령이 끝나면 `build` 디렉터리 안에 `hello`(윈도우에서는 `hello.exe`) 실행 파일이 만들어집니다.

```bash
./build/hello
```

주목할 점은 `cmake --build`가 내부적으로 어떤 빌드 도구를 쓰든 상관없이 동일한 명령으로 빌드를 지시할 수 있다는 것입니다. Generate 단계에서 Makefile이 만들어졌다면 내부적으로 `make`를, `build.ninja`가 만들어졌다면 내부적으로 `ninja`를 호출합니다. 사용자는 어떤 빌드 도구가 실제로 쓰이는지 몰라도 `cmake --build` 명령 하나로 일관되게 빌드를 실행할 수 있습니다.

정리하면 실습 전체는 다음 두 줄로 요약됩니다.

```bash
cmake -B build        # Configure + Generate
cmake --build build    # Build
```

`CMakeLists.txt`를 수정한 뒤에도 이 두 명령을 그대로 다시 실행하면 됩니다. CMake는 `CMakeLists.txt`가 변경되었는지 자동으로 감지해서 필요한 경우 Configure와 Generate를 다시 수행합니다.

## CMakeCache.txt란 무엇인가

Configure 단계를 거치면 `build` 디렉터리 안에 `CMakeCache.txt`라는 파일이 생성됩니다. 이 파일은 Configure 단계에서 계산되거나 탐지된 값들 — 예를 들어 어떤 컴파일러를 사용할지, 빌드 타입은 무엇인지, 각종 옵션 값은 무엇인지 — 을 키-값 쌍의 형태로 저장해 둡니다.

이 캐시가 존재하는 이유는 매번 Configure를 다시 실행할 때마다 컴파일러 탐지 같은 비교적 비용이 큰 작업을 반복하지 않기 위해서입니다. 한 번 탐지된 값은 캐시에 저장되고, 이후 Configure를 다시 실행할 때는 이 캐시를 재사용합니다.

`CMakeCache.txt`를 열어보면 다음과 같은 항목들을 확인할 수 있습니다.

```
CMAKE_CXX_COMPILER:FILEPATH=C:/msys64/mingw64/bin/g++.exe
CMAKE_BUILD_TYPE:STRING=
PROJECT_BINARY_DIR:STATIC=D:/.../build
```

빌드 설정을 완전히 새로 시작하고 싶을 때는 `build` 디렉터리 전체를 삭제하고 `cmake -B build`를 다시 실행하면 됩니다. 이 캐시 때문에 발생하는 문제(예: 컴파일러 경로를 바꿨는데 이전 설정이 남아 있는 경우)를 해결하는 가장 확실한 방법이기도 합니다. 이 파일을 직접 손으로 수정하는 일은 거의 없으며, 캐시 값을 바꾸고 싶다면 `cmake -D옵션이름=값 -B build`처럼 커맨드라인에서 값을 지정하는 것이 일반적입니다.

## g++ 직접 호출과 비교했을 때 무엇이 편해지는가

2장에서는 다음과 같은 방식으로 컴파일했습니다.

```bash
g++ -std=c++20 -Wall -o hello main.cpp utils.cpp
```

파일이 몇 개 안 될 때는 이 명령도 크게 부담스럽지 않습니다. 하지만 프로젝트가 커지면서 다음과 같은 문제들이 하나둘 드러납니다.

- 소스 파일이 수십, 수백 개로 늘어나면 매번 이 목록을 손으로 관리하기 어려워집니다.
- 팀원마다 컴파일러 경로나 운영체제가 다르면, 같은 명령이 어떤 컴퓨터에서는 동작하고 어떤 컴퓨터에서는 동작하지 않는 상황이 생깁니다.
- 리눅스에서는 `g++`을, 윈도우에서는 MSVC의 `cl.exe`를 써야 한다면 아예 서로 다른 명령을 유지해야 합니다.
- 외부 라이브러리를 추가할 때마다 `-I`(include 경로), `-L`(라이브러리 경로), `-l`(링크할 라이브러리 이름) 옵션을 일일이 늘려가야 합니다.

CMake를 사용하면 이런 문제들이 상당 부분 해소됩니다. `CMakeLists.txt` 하나만 작성해 두면, 그 파일은 특정 컴파일러나 특정 운영체제에 종속되지 않습니다. CMake가 Configure 단계에서 현재 시스템에 설치된 컴파일러를 자동으로 탐지하고, 그 컴파일러에 맞는 빌드 스크립트를 만들어주기 때문입니다.

예를 들어 같은 `CMakeLists.txt`를 두고,

- 리눅스에서는 `cmake -B build`를 실행하면 GCC를 탐지해 Makefile을 만들고,
- 윈도우에서는 같은 명령이 MSVC나 MinGW의 `g++`을 탐지해 그에 맞는 빌드 파일을 만듭니다.

즉 "소스 파일 목록과 빌드 방법을 어떻게 정의할 것인가"라는 문제와 "실제로 어떤 컴파일러로, 어떤 운영체제에서 그것을 실행할 것인가"라는 문제가 CMake를 통해 분리됩니다. 이 분리는 프로젝트 규모가 커질수록, 그리고 여러 사람이 서로 다른 환경에서 같은 프로젝트를 빌드해야 하는 상황일수록 그 가치가 커집니다.

물론 지금까지 다룬 것은 아주 단순한 프로젝트에 불과합니다. 소스 파일이 여러 디렉터리에 나뉘어 있거나, 라이브러리와 실행 파일을 분리해서 관리해야 하는 좀 더 현실적인 프로젝트 구조는 7장에서 이어서 다룹니다. 또한 지금은 Make 계열의 기본 생성기를 사용했지만, 8장에서는 더 빠른 빌드 속도를 제공하는 Ninja를 생성기로 사용하는 방법을 살펴봅니다.

## 요약

- CMake는 컴파일러가 아니라, `CMakeLists.txt`를 읽어 Make나 Ninja 같은 실제 빌드 도구가 사용할 빌드 스크립트를 만들어주는 "빌드 시스템 생성기"입니다.
- `CMakeLists.txt`의 기본 구조는 `cmake_minimum_required()`로 CMake 최소 버전을 지정하고, `project()`로 프로젝트 이름과 언어를 선언하고, `add_executable()`로 소스 파일들을 묶어 실행 파일 타깃을 정의하는 세 요소로 이루어집니다.
- CMake의 작업은 Configure(설정 정보 계산), Generate(빌드 스크립트 생성), Build(실제 컴파일 수행)의 3단계로 나뉘며, `cmake -B build`가 앞의 두 단계를, `cmake --build build`가 마지막 단계를 수행합니다.
- Configure 단계의 결과는 `build` 디렉터리 안의 `CMakeCache.txt`에 저장되어, 반복적인 컴파일러 탐지 작업을 줄여줍니다.
- `g++`을 직접 호출하는 방식과 달리, CMake를 사용하면 컴파일러나 운영체제에 종속되지 않는 하나의 `CMakeLists.txt`로 여러 환경에서 동일하게 빌드할 수 있습니다.

## 연습문제

1. CMake가 "컴파일러가 아니라 빌드 시스템 생성기"라는 말의 의미를 자신의 언어로 설명해 보세요.
2. `cmake_minimum_required()`, `project()`, `add_executable()`이 각각 어떤 역할을 하는지 설명하세요.
3. Configure, Generate, Build 세 단계가 각각 무엇을 하는 단계인지 설명하고, 이 중 `cmake -B build` 명령이 담당하는 단계와 `cmake --build build` 명령이 담당하는 단계를 구분하세요.
4. `CMakeCache.txt` 파일이 존재하는 이유는 무엇이며, 빌드 설정을 완전히 초기화하고 싶을 때는 어떻게 해야 하나요?
5. 소스 파일 두 개(`main.cpp`, `math_utils.cpp`)로 구성된 프로젝트를 위한 `CMakeLists.txt`를 작성하고, `cmake -B build`와 `cmake --build build` 명령으로 직접 빌드해 실행 파일을 만들어 보세요.

---

[◀ 이전: 5장. IntelliSense 설정](ch05-IntelliSense설정.md) | [📖 목차](00-목차.md) | [다음: 7장. CMake 프로젝트 구조화 ▶](ch07-CMake프로젝트구조화.md)
