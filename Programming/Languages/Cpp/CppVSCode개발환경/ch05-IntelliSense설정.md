# 5장. IntelliSense 설정

[◀ 이전: 4장. VS Code로 빌드하고 디버깅하기](ch04-VSCode로빌드하고디버깅하기.md) | [📖 목차](00-목차.md) | [다음: 6장. CMake 기초 ▶](ch06-CMake기초.md)


3장에서 VS Code에 C/C++ 확장을 설치했고, 4장에서는 그 확장을 이용해 코드를 빌드하고 디버깅하는 방법을 다뤘습니다. 그런데 코드를 작성하다 보면 분명히 정상적으로 컴파일되는 코드인데도 편집기 화면에는 빨간 밑줄이 그어지고 "헤더 파일을 찾을 수 없습니다"라는 경고가 뜨는 경우를 종종 마주치게 됩니다. 이 장에서는 그 원인이 되는 IntelliSense라는 기능과, 이를 제어하는 `c_cpp_properties.json` 파일에 대해 다룹니다.

## IntelliSense란 무엇인가

IntelliSense는 VS Code의 C/C++ 확장이 제공하는 코드 분석 엔진입니다. 컴파일러가 실제로 코드를 기계어로 번역하는 것과는 별개로, IntelliSense는 편집기 안에서 다음과 같은 기능을 실시간으로 제공합니다.

- **자동완성**: 변수나 함수의 이름을 일부만 입력해도 나머지를 제안해 줍니다. 클래스의 멤버 함수 목록을 마침표(`.`)나 화살표(`->`) 뒤에 바로 보여주는 것도 이 기능입니다.
- **정의로 이동 / 선언으로 이동**: 함수나 변수를 우클릭해서 "Go to Definition"을 선택하면, 그 심벌이 실제로 정의된 위치로 즉시 이동합니다. 여러 파일에 걸쳐 흩어진 코드를 탐색할 때 특히 유용합니다.
- **참조 찾기**: 어떤 변수나 함수가 코드 전체에서 어디에서 사용되고 있는지 찾아줍니다.
- **매개변수 힌트**: 함수를 호출할 때 괄호를 열면 그 함수가 어떤 인자를 받는지 툴팁으로 보여줍니다.
- **오류 및 경고 표시**: 문법 오류나 정의되지 않은 심벌을 편집기 안에서 빨간 밑줄이나 물결선으로 미리 알려줍니다.

이 모든 기능을 제공하려면 IntelliSense 엔진도 컴파일러와 비슷하게 헤더 파일을 파싱하고, 매크로를 해석하고, 타입 정보를 구축해야 합니다. 즉 IntelliSense는 실제 컴파일러는 아니지만, 코드를 이해하기 위해 컴파일러와 유사한 작업을 내부적으로 수행하는 별도의 프로그램입니다.

여기서 핵심은 "별도"라는 단어입니다. IntelliSense는 실제 빌드 과정과 완전히 독립적으로 동작합니다. 즉 여러분이 터미널에서 `g++`을 실행해 컴파일하는 것과, VS Code 편집기 화면에 빨간 밑줄이 뜨는지 여부는 서로 다른 두 개의 시스템이 결정합니다. 이 둘이 사용하는 설정이 어긋나면, 실제로는 컴파일이 잘 되는데도 화면에는 오류가 표시되는 상황이 벌어집니다.

## c_cpp_properties.json의 역할

IntelliSense 엔진이 코드를 올바르게 분석하려면 두 가지 정보가 반드시 필요합니다.

1. **컴파일러 경로(`compilerPath`)**: 어떤 컴파일러를 기준으로 분석할 것인지. GCC인지 Clang인지, 그리고 어떤 버전인지에 따라 지원하는 내장 매크로나 표준 라이브러리 헤더의 위치가 달라지기 때문입니다.
2. **include 경로(`includePath`)**: `#include`로 불러오는 헤더 파일들을 어느 디렉터리에서 찾을 것인지.

이 두 정보를 담는 파일이 바로 `c_cpp_properties.json`입니다. 이 파일은 프로젝트 루트의 `.vscode` 폴더 안에 위치하며, C/C++ 확장이 이 파일을 읽어 IntelliSense 엔진을 구성합니다.

4장에서 다룬 `tasks.json`(빌드 명령을 정의)이나 `launch.json`(디버거 실행 설정을 정의)과 마찬가지로, `c_cpp_properties.json`도 `.vscode` 폴더에 저장되는 프로젝트별 설정 파일입니다. 다만 역할이 완전히 다릅니다. `tasks.json`과 `launch.json`은 실제로 코드를 빌드하고 실행하는 방법을 정의하지만, `c_cpp_properties.json`은 오직 편집기 화면에서 코드를 어떻게 "이해"하고 보여줄지만 결정합니다.

## GUI로 설정하기: C/C++: Edit Configurations (UI)

`c_cpp_properties.json`을 직접 열어 편집할 수도 있지만, VS Code는 이 파일을 시각적으로 편집할 수 있는 UI를 제공합니다.

1. `Ctrl+Shift+P`를 눌러 명령 팔레트를 엽니다.
2. `C/C++: Edit Configurations (UI)`를 입력하고 선택합니다.

이 명령을 실행하면 다음과 같은 항목들을 갖춘 설정 화면이 열립니다.

- **Configuration name**: 설정 묶음의 이름입니다. 기본값은 운영체제에 따라 `Win32`, `Linux`, `Mac`처럼 자동으로 지정됩니다. 하나의 파일 안에 여러 개의 configuration을 두고 상황에 따라 전환할 수도 있습니다.
- **Compiler path**: IntelliSense가 기준으로 삼을 컴파일러의 실행 파일 경로입니다. 2장에서 설치한 `g++`의 전체 경로(예: `C:/msys64/mingw64/bin/g++.exe`)를 지정합니다. 이 항목을 지정하면 확장이 해당 컴파일러의 기본 include 경로와 표준 버전을 자동으로 추론합니다.
- **IntelliSense mode**: 어떤 컴파일러의 동작 방식을 흉내 낼 것인지 지정합니다. GCC를 사용한다면 `gcc-x64`, MSVC라면 `msvc-x64`처럼 선택합니다.
- **Include path**: 프로젝트에서 사용하는 헤더 파일들이 위치한 디렉터리 목록입니다. 기본적으로 `${workspaceFolder}/**`가 포함되어 있어 작업 폴더 하위의 모든 디렉터리를 재귀적으로 탐색하지만, 외부 라이브러리를 쓴다면 그 라이브러리의 헤더 디렉터리를 추가로 등록해야 합니다.
- **C standard / C++ standard**: 분석 기준으로 삼을 언어 표준(예: `c++17`, `c++20`)을 지정합니다.

이 UI에서 값을 입력하면 VS Code가 내부적으로 `.vscode/c_cpp_properties.json` 파일을 생성하거나 수정합니다.

## JSON을 직접 편집하기

UI 대신 파일을 직접 열어 편집할 수도 있습니다. 명령 팔레트에서 `C/C++: Edit Configurations (JSON)`을 선택하거나, `.vscode/c_cpp_properties.json` 파일을 직접 열면 됩니다. 실제 파일의 구조는 다음과 같은 형태를 띱니다.

```json
{
    "configurations": [
        {
            "name": "Win32",
            "includePath": [
                "${workspaceFolder}/**",
                "${workspaceFolder}/include",
                "C:/libs/mylib/include"
            ],
            "defines": [
                "_DEBUG",
                "UNICODE"
            ],
            "compilerPath": "C:/msys64/mingw64/bin/g++.exe",
            "cStandard": "c17",
            "cppStandard": "c++20",
            "intelliSenseMode": "gcc-x64"
        }
    ],
    "version": 4
}
```

각 항목의 의미를 정리하면 다음과 같습니다.

- `includePath`: 헤더 검색 경로 목록. 배열의 각 항목은 디렉터리 경로이며, `**`는 그 하위 디렉터리까지 재귀적으로 포함한다는 뜻입니다.
- `defines`: 매크로 정의 목록. 컴파일 시 `-D` 옵션으로 전달하는 것과 동일한 효과를 IntelliSense 분석에 반영합니다.
- `compilerPath`: 분석 기준 컴파일러의 경로.
- `cStandard`, `cppStandard`: 각각 C와 C++ 언어 표준.
- `intelliSenseMode`: 분석 엔진이 흉내 낼 컴파일러 동작 방식.
- `version`: 이 설정 파일 스키마의 버전으로, 확장이 내부적으로 관리하는 값입니다. 직접 바꿀 필요는 없습니다.

여러 플랫폼이나 컴파일러를 오가며 작업한다면, `configurations` 배열에 항목을 여러 개 추가하고 VS Code 하단 상태 바에서 현재 사용할 configuration을 전환할 수 있습니다.

## 흔히 겪는 문제: 헤더는 못 찾는데 컴파일은 되는 경우

C++ 개발 환경을 처음 구성할 때 가장 자주 마주치는 혼란 중 하나가 바로 이것입니다. 터미널에서 `g++`으로 컴파일하면 아무 문제 없이 실행 파일이 만들어지는데, VS Code 편집기에서는 `#include` 구문 아래에 빨간 밑줄이 그어지고 "포함 파일을 열 수 없습니다"라는 경고가 표시되는 경우입니다.

이 현상이 발생하는 이유는 앞서 설명한 것처럼 **IntelliSense와 실제 빌드가 서로 다른 include 경로 정보를 사용하기 때문**입니다.

- 실제 빌드는 `g++ -I외부라이브러리/include main.cpp`처럼 커맨드라인 인자로 전달된 `-I` 옵션이나, 4장에서 다룬 `tasks.json`의 빌드 명령에 정의된 경로를 사용합니다.
- IntelliSense는 오직 `c_cpp_properties.json`의 `includePath` 항목만을 참고합니다.

따라서 빌드 스크립트에는 특정 라이브러리의 경로를 추가했지만 `c_cpp_properties.json`은 갱신하지 않았다면, 컴파일은 정상적으로 되면서도 편집기에는 오류가 표시되는 불일치가 발생합니다. 반대의 경우도 성립합니다. `c_cpp_properties.json`에는 어떤 경로를 추가해 편집기상의 오류는 사라졌지만, 실제 빌드 명령에는 그 경로가 빠져 있어서 컴파일 자체가 실패하는 경우도 흔히 있습니다.

이 문제를 마주쳤을 때 점검해야 할 순서는 다음과 같습니다.

1. 정말로 컴파일이 되는지 터미널에서 다시 한번 확인합니다.
2. 컴파일에 사용된 `-I` 옵션 목록을 확인합니다.
3. 그 경로들이 `c_cpp_properties.json`의 `includePath`에도 동일하게 등록되어 있는지 확인합니다.
4. 누락되어 있다면 추가하고, C/C++ 확장이 파일 변경을 감지해 IntelliSense를 다시 계산하도록 잠시 기다립니다. 즉시 반영되지 않으면 명령 팔레트에서 `C/C++: Reset IntelliSense Database`를 실행해 강제로 다시 계산하게 할 수도 있습니다.

이렇게 두 설정을 손으로 맞춰 나가는 과정은 프로젝트가 커질수록 번거로워집니다. 라이브러리를 하나 추가할 때마다 빌드 스크립트와 `c_cpp_properties.json`을 매번 동시에 수정해야 하기 때문입니다.

## 앞으로의 방향: CMake Tools 확장과의 통합

이 번거로움을 근본적으로 해결하는 방법은 6장부터 다룰 CMake를 도입하고, 9장에서 소개할 VS Code CMake Tools 확장을 함께 사용하는 것입니다. CMake Tools 확장을 사용하면 프로젝트의 빌드 설정(`CMakeLists.txt`)에 정의된 include 경로와 컴파일 옵션을 확장이 자동으로 읽어서 IntelliSense 설정에 반영해 줍니다. 즉 `c_cpp_properties.json`을 손으로 관리할 필요 없이, 빌드 설정 하나만 유지하면 IntelliSense도 항상 그와 일치된 상태를 유지하게 됩니다.

지금 이 장에서 `c_cpp_properties.json`의 구조와 동작 원리를 이해해 두면, 9장에서 CMake Tools가 이 파일을 자동으로 생성하고 갱신하는 것을 볼 때 그 이면에서 무슨 일이 벌어지는지 정확히 파악할 수 있을 것입니다.

## 요약

- IntelliSense는 VS Code C/C++ 확장이 제공하는 코드 분석 엔진으로, 자동완성, 정의로 이동, 오류 표시 등의 기능을 담당합니다.
- IntelliSense는 실제 컴파일러가 아니며, 실제 빌드 과정과는 완전히 독립적으로 동작합니다.
- `c_cpp_properties.json`은 IntelliSense가 코드를 분석할 때 사용할 컴파일러 경로(`compilerPath`)와 헤더 검색 경로(`includePath`)를 정의하는 설정 파일이며, `.vscode` 폴더 안에 위치합니다.
- 명령 팔레트의 `C/C++: Edit Configurations (UI)`로 GUI를 통해 설정할 수도 있고, `C/C++: Edit Configurations (JSON)`으로 파일을 직접 편집할 수도 있습니다.
- 실제 빌드에 사용하는 include 경로와 `c_cpp_properties.json`의 `includePath`가 서로 어긋나면, 컴파일은 정상적으로 되는데도 편집기에는 오류가 표시되는(혹은 그 반대의) 불일치가 발생합니다.
- 9장에서 다룰 CMake Tools 확장을 사용하면 이 설정을 손으로 관리할 필요 없이 빌드 설정과 자동으로 동기화됩니다.

## 연습문제

1. IntelliSense가 제공하는 기능을 세 가지 이상 나열하고, 각각이 어떤 상황에서 유용한지 설명하세요.
2. `c_cpp_properties.json`의 `includePath` 항목과 `compilerPath` 항목이 각각 어떤 역할을 하는지 설명하세요.
3. 명령 팔레트에서 `c_cpp_properties.json`을 GUI로 편집하는 명령의 이름은 무엇인가요?
4. 터미널에서는 컴파일이 성공하는데 VS Code 편집기에는 헤더를 찾을 수 없다는 오류가 표시되는 이유를 설명하고, 이를 해결하기 위해 점검해야 할 순서를 서술하세요.
5. 자신의 실습 환경에서 외부 라이브러리 하나를 가정하고, 그 라이브러리의 include 경로를 `c_cpp_properties.json`에 직접 추가하는 JSON 조각을 작성해 보세요.

---

[◀ 이전: 4장. VS Code로 빌드하고 디버깅하기](ch04-VSCode로빌드하고디버깅하기.md) | [📖 목차](00-목차.md) | [다음: 6장. CMake 기초 ▶](ch06-CMake기초.md)
