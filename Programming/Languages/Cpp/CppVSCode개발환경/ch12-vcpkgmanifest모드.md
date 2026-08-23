# 12장. vcpkg manifest 모드

[◀ 이전: 11장. vcpkg로 라이브러리 사용하기](ch11-vcpkg로라이브러리사용하기.md) | [📖 목차](00-목차.md) | [다음: 13장. 컴파일러 표준과 플래그 ▶](ch13-컴파일러표준과플래그.md)


## 11장에서 남은 문제

11장 끝에서 classic 모드의 한계를 하나 짚었습니다. `vcpkg install fmt`로 라이브러리를 설치했지만, 그 사실은 오직 여러분의 컴퓨터에 설치된 vcpkg 내부에만 기록되어 있을 뿐, `fmt-demo` 프로젝트 폴더 자체에는 아무 흔적도 남지 않았습니다. `CMakeLists.txt`에는 `find_package(fmt CONFIG REQUIRED)`라는 줄이 있지만, 이 줄만 보고는 "그래서 `fmt`를 어떻게 준비해야 하는지"까지는 알 수 없습니다.

이 프로젝트를 깃 저장소에 올려 동료에게 전달했다고 가정해 봅시다. 동료가 프로젝트를 내려받아 CMake로 구성을 시도하면, 자신의 컴퓨터에는 `fmt`가 설치되어 있지 않으니 `find_package`가 실패하고 구성 단계에서 오류가 납니다. 동료는 오류 메시지를 보고 "아, `fmt`가 없구나"라고 짐작한 뒤, README나 다른 문서를 뒤져 `vcpkg install fmt`라는 명령을 찾아 손으로 실행해야 합니다. 프로젝트가 의존하는 라이브러리가 하나가 아니라 여러 개라면, 이 수작업은 사람이 실수하기 딱 좋은 지점이 됩니다.

이번 장에서 다룰 **manifest 모드**는 이 문제를 "프로젝트가 필요로 하는 라이브러리 목록을 프로젝트 폴더 안에 파일로 선언해 두면, CMake 구성 단계에서 vcpkg가 그 목록을 읽어 알아서 설치까지 끝내 놓는다"는 방식으로 해결합니다.

## classic 모드와 manifest 모드의 차이

두 모드의 차이를 표로 정리하면 다음과 같습니다.

| 구분 | classic 모드 | manifest 모드 |
|---|---|---|
| 의존성을 선언하는 곳 | 없음 (커맨드라인 명령으로만 존재) | 프로젝트 폴더의 `vcpkg.json` |
| 라이브러리가 설치되는 위치 | vcpkg 설치 폴더 안 `installed/` (전역) | 프로젝트 폴더 안 `vcpkg_installed/` (프로젝트별) |
| 설치 시점 | 개발자가 `vcpkg install`을 손으로 실행할 때 | CMake 구성(configure) 시점에 자동으로 |
| 다른 프로젝트와 라이브러리 공유 | 같은 버전이면 공유됨 | 프로젝트마다 독립적으로 설치됨 |
| 새 개발자가 참여할 때 | README 등을 보고 수동으로 설치 명령 실행 | 저장소를 받아 CMake configure만 하면 끝 |

가장 중요한 차이는 "의존성 정보가 어디에 있는가"입니다. classic 모드에서는 이 정보가 개발자의 머릿속이나 문서에만 있지만, manifest 모드에서는 `vcpkg.json`이라는 파일 자체가 곧 의존성 목록입니다. 이는 Node.js의 `package.json`이나 파이썬의 `requirements.txt`, `pyproject.toml`이 하는 역할과 정확히 같은 개념입니다. 프로젝트 저장소 안에 "이 프로젝트를 빌드하려면 무엇이 필요한지"가 코드처럼 함께 버전 관리된다는 뜻입니다.

## vcpkg.json 작성하기

11장에서 만든 `fmt-demo` 프로젝트를 그대로 manifest 모드로 바꿔보겠습니다. 프로젝트 폴더 구조는 다음과 같이 파일 하나가 늘어납니다.

```bash
fmt-demo/
├── CMakeLists.txt
├── main.cpp
└── vcpkg.json
```

`vcpkg.json`은 다음과 같이 작성합니다.

```json
{
  "name": "fmt-demo",
  "version": "1.0.0",
  "dependencies": [
    "fmt"
  ]
}
```

각 필드의 의미는 다음과 같습니다.

- `name`: 프로젝트(매니페스트)의 이름입니다. 소문자, 숫자, 하이픈(`-`)만 사용할 수 있고 공백이나 대문자, 밑줄은 허용되지 않습니다.
- `version`: 이 프로젝트 자체의 버전입니다. `fmt`의 버전이 아니라 `fmt-demo`라는 프로젝트의 버전을 의미합니다.
- `dependencies`: 이 프로젝트가 필요로 하는 라이브러리 이름을 배열로 나열합니다. 지금은 `fmt` 하나만 있지만, 예를 들어 JSON 파싱 라이브러리와 테스트 프레임워크가 더 필요하다면 `["fmt", "nlohmann-json", "catch2"]`처럼 이어서 나열하면 됩니다.

`CMakeLists.txt`는 11장에서 작성한 내용을 그대로 사용합니다. 즉 `find_package(fmt CONFIG REQUIRED)`와 `target_link_libraries(fmt_demo PRIVATE fmt::fmt)`는 고칠 필요가 없습니다. manifest 모드로 바뀌는 것은 "라이브러리를 어떻게 준비하는가"이지, "CMake가 그 라이브러리를 어떻게 찾아 연결하는가"가 아니기 때문입니다.

```cmake
cmake_minimum_required(VERSION 3.20)
project(fmt_demo LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

find_package(fmt CONFIG REQUIRED)

add_executable(fmt_demo main.cpp)
target_link_libraries(fmt_demo PRIVATE fmt::fmt)
```

## 프로젝트 전용 설치 폴더: vcpkg_installed

`vcpkg.json`을 프로젝트 루트에 두고 11장과 동일하게 툴체인 파일을 지정해 CMake를 구성하면, vcpkg는 이번에는 명령을 손으로 치지 않아도 `vcpkg.json`의 `dependencies` 목록을 읽어서 필요한 라이브러리를 자동으로 설치합니다.

```bash
cmake -B build -DCMAKE_TOOLCHAIN_FILE=C:/vcpkg/scripts/buildsystems/vcpkg.cmake
```

이 명령을 실행하면 vcpkg 툴체인 파일이 현재 소스 디렉토리(`CMakeLists.txt`가 있는 위치)에 `vcpkg.json`이 있는지를 자동으로 확인합니다. 파일이 있으면 vcpkg는 "classic 모드가 아니라 manifest 모드로 동작하라"는 신호로 받아들이고, `dependencies`에 나열된 라이브러리를 아직 설치하지 않았다면 그 자리에서 내려받고 컴파일합니다. 이때 설치되는 위치는 11장처럼 vcpkg 설치 폴더 안이 아니라, 프로젝트 루트 아래 새로 생기는 다음 폴더입니다.

```bash
fmt-demo/
├── build/
├── CMakeLists.txt
├── main.cpp
├── vcpkg.json
└── vcpkg_installed/
    └── ...
```

`vcpkg_installed/` 폴더 안의 구조는 classic 모드에서 vcpkg 설치 폴더 아래 있던 `installed/`와 거의 동일하며, 헤더 파일과 컴파일된 라이브러리, CMake 설정 파일이 이 프로젝트만을 위해 별도로 들어 있습니다. 즉 이 프로젝트의 `fmt`와 다른 프로젝트의 `fmt`는 서로 다른 폴더에 독립적으로 존재하게 됩니다. 이는 diskspace를 조금 더 쓰는 대신, 프로젝트마다 서로 다른 버전의 라이브러리를 사용해도 충돌이 생기지 않는다는 장점으로 돌아옵니다.

`vcpkg_installed/`는 소스 코드가 아니라 빌드 산출물에 해당하므로, `build/` 디렉토리와 마찬가지로 `.gitignore`에 등록해서 버전 관리 대상에서 제외하는 것이 일반적입니다.

```bash
# .gitignore
build/
vcpkg_installed/
```

빌드 자체는 11장과 동일합니다.

```bash
cmake --build build
./build/fmt_demo
```

## 새 개발자가 프로젝트에 합류하는 상황 비교

manifest 모드의 진짜 가치는 "다른 사람이 이 프로젝트를 처음 받았을 때" 드러납니다.

classic 모드였다면 동료는 저장소를 내려받은 뒤 다음과 같은 과정을 거쳐야 합니다.

1. 저장소를 클론한다.
2. `CMakeLists.txt`를 읽고 `find_package(fmt ...)`가 있는 것을 발견한다.
3. `fmt`가 무엇인지, 어떻게 설치하는지 README나 팀 위키를 찾아본다.
4. `vcpkg install fmt`를 손으로 실행한다.
5. 그제서야 `cmake -B build ...`가 성공한다.

manifest 모드라면 이 과정이 다음으로 줄어듭니다.

1. 저장소를 클론한다. (`vcpkg.json`이 함께 따라옵니다.)
2. `cmake -B build -DCMAKE_TOOLCHAIN_FILE=...`를 실행한다.
3. vcpkg가 `vcpkg.json`을 읽어 `fmt`를 자동으로 설치하고, 구성이 성공한다.

라이브러리를 설치하라고 안내하는 문서도, 그 문서를 읽고 명령어를 따로 기억해서 실행하는 사람도 필요 없어집니다. 프로젝트를 구성하는 절차 자체가 "저장소를 받고 CMake configure 한 번 돌리기"로 단순해지는 것이며, 이는 실무에서 여러 명이 함께 작업하는 C++ 프로젝트에 vcpkg를 도입할 때 manifest 모드를 기본으로 선택하는 가장 큰 이유입니다.

## 버전을 고정하고 싶다면: baseline과 overrides

`vcpkg.json`에 `"fmt"`라고만 적으면, 정확히 어떤 버전의 `fmt`가 설치될지는 여러분이 사용하는 vcpkg 자체의 버전(정확히는 vcpkg가 참조하는 라이브러리 카탈로그의 시점)에 따라 달라집니다. 오늘 설치한 결과와 6개월 뒤 새로 vcpkg를 내려받은 동료가 설치한 결과가 서로 다른 버전의 `fmt`가 될 수 있다는 뜻입니다. 대부분의 경우 이는 큰 문제가 되지 않지만, 팀 전체가 정확히 같은 버전의 라이브러리로 빌드되기를 보장하고 싶은 상황도 있습니다.

이런 상황을 위해 vcpkg는 `builtin-baseline`이라는 필드로 "라이브러리 카탈로그를 특정 시점에 고정"하는 기능과, `overrides`라는 필드로 "특정 라이브러리만 명시적으로 특정 버전을 쓰도록 강제"하는 기능을 제공합니다. 이 두 기능은 팀 규모가 커지고 재현 가능한 빌드가 중요해질수록 유용해지지만, 실제 문법과 값을 채우는 방법(커밋 해시를 어떻게 구하는지, 버전 문자열을 어떤 형식으로 적는지 등)은 이 책의 범위를 벗어나는 세부 사항이므로 여기서는 이런 개념이 존재한다는 정도만 짚고 넘어가겠습니다. 실제 실무 프로젝트에서 버전 고정이 필요해지면, vcpkg 공식 문서에서 `builtin-baseline`과 `overrides` 항목을 검색해 문법을 확인하면 됩니다.

## VS Code CMake Tools와 manifest 모드

9장에서 설정한 CMake Tools 확장은 manifest 모드와도 자연스럽게 맞물립니다. 11장에서와 마찬가지로 `.vscode/settings.json`에 툴체인 파일 경로만 지정해 두면, 프로젝트 폴더에 `vcpkg.json`이 있는지 여부는 vcpkg 툴체인 파일이 알아서 판단하므로 별도 설정이 더 필요하지 않습니다.

```json
{
  "cmake.configureSettings": {
    "CMAKE_TOOLCHAIN_FILE": "C:/vcpkg/scripts/buildsystems/vcpkg.cmake"
  }
}
```

VS Code에서 CMake Tools의 구성(Configure) 명령을 실행하면, 상태 표시줄이나 출력 패널에서 vcpkg가 `vcpkg.json`을 읽어 라이브러리를 설치하는 로그를 그대로 확인할 수 있습니다. 처음 구성할 때는 라이브러리를 새로 컴파일하느라 시간이 다소 걸리지만, 한 번 설치된 뒤에는 `vcpkg_installed/`에 캐시된 결과를 그대로 재사용하므로 이후 구성은 빠르게 끝납니다.

## 요약

- classic 모드는 라이브러리를 vcpkg 설치 폴더에 전역으로 설치하고, manifest 모드는 프로젝트 폴더의 `vcpkg.json`에 의존성을 선언해 프로젝트별로 설치합니다.
- `vcpkg.json`은 `name`, `version`, `dependencies` 필드로 구성되며, `dependencies` 배열에 필요한 라이브러리 이름을 나열합니다.
- `vcpkg.json`이 있는 프로젝트를 CMake로 구성하면, vcpkg 툴체인 파일이 이를 자동으로 인식해 필요한 라이브러리를 프로젝트 폴더 안 `vcpkg_installed/`에 설치합니다.
- manifest 모드를 사용하면 다른 개발자는 저장소를 받아 CMake configure만 실행해도 필요한 라이브러리가 자동으로 준비되므로, 별도의 설치 안내 문서나 수작업이 필요 없어집니다.
- `find_package`와 `target_link_libraries`를 사용하는 `CMakeLists.txt`의 내용은 classic 모드와 manifest 모드에서 동일합니다.
- 라이브러리 버전을 팀 전체에서 고정하고 싶을 때는 `builtin-baseline`과 `overrides` 필드를 사용할 수 있으며, 자세한 문법은 필요할 때 vcpkg 공식 문서를 참고하면 됩니다.

## 연습문제

1. classic 모드와 manifest 모드에서 라이브러리가 설치되는 위치가 각각 어디인지, 그리고 이 차이가 실무에서 왜 중요한지 설명해 보세요.
2. 11장에서 만든 `fmt-demo` 프로젝트를 이번 장의 방법대로 `vcpkg.json`을 추가해 manifest 모드로 바꾸고, `build/`와 `vcpkg_installed/`를 모두 지운 뒤 처음부터 다시 구성해서 vcpkg가 자동으로 `fmt`를 설치하는 로그를 직접 확인해 보세요.
3. `vcpkg.json`의 `dependencies` 배열에 `nlohmann-json`을 추가하고, `CMakeLists.txt`에 이를 찾아 연결하는 줄을 추가해서 두 개의 라이브러리를 함께 사용하는 프로젝트로 확장해 보세요.
4. `vcpkg_installed/`를 `.gitignore`에 등록해야 하는 이유를, `build/` 디렉토리를 제외하는 이유와 비교해서 설명해 보세요.
5. `builtin-baseline`이나 `overrides` 없이 `vcpkg.json`만 사용할 때, 시간이 지나면서 팀원마다 서로 다른 버전의 라이브러리가 설치될 수 있는 상황을 구체적인 예를 들어 설명해 보세요.

---

[◀ 이전: 11장. vcpkg로 라이브러리 사용하기](ch11-vcpkg로라이브러리사용하기.md) | [📖 목차](00-목차.md) | [다음: 13장. 컴파일러 표준과 플래그 ▶](ch13-컴파일러표준과플래그.md)
