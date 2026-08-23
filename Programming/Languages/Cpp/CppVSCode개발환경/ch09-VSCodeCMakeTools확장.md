# 9장. VS Code CMake Tools 확장

[◀ 이전: 8장. Ninja 빌드 시스템](ch08-Ninja빌드시스템.md) | [📖 목차](00-목차.md) | [다음: 10장. vcpkg 소개와 설치 ▶](ch10-vcpkg소개와설치.md)


## 이 장에서 할 일

6장과 7장에서는 커맨드라인에서 `cmake -S . -B build`, `cmake --build build` 명령을 직접 입력해 프로젝트를 구성하고 빌드했습니다. 8장에서는 그 뒤에서 실제로 파일을 컴파일하는 Ninja를 살펴봤습니다. 4장에서는 `tasks.json`과 `launch.json`을 손으로 작성해서 VS Code 안에서 빌드와 디버깅을 연결했고, 5장에서는 `c_cpp_properties.json`을 편집해서 IntelliSense가 올바른 헤더 경로를 인식하도록 만들었습니다.

이 장에서 다룰 CMake Tools 확장은 지금까지 손으로 해온 이 작업들 중 상당 부분을 대신 처리해 줍니다. CMake 프로젝트라는 사실을 확장이 인식하기만 하면, 컴파일러 선택부터 구성(configure), 빌드, 디버그 실행까지 상태 바의 버튼 몇 개로 끝낼 수 있습니다. 그렇다고 4~8장에서 배운 내용이 쓸모없어지는 것은 아닙니다. CMake Tools는 결국 같은 `cmake`, `ninja` 명령을 대신 실행해 주는 도구일 뿐이므로, 그 명령들이 무엇을 하는지 이해하고 있어야 확장이 뭔가 이상하게 동작할 때 원인을 짚어낼 수 있습니다.

## CMake Tools란 무엇인가

CMake Tools는 마이크로소프트가 만들어 배포하는 VS Code 확장으로, 마켓플레이스에서 게시자 이름이 `ms-vscode.cmake-tools`입니다. 3장에서 설치한 C/C++ 확장(`ms-vscode.cpptools`)이 IntelliSense, 디버깅 어댑터 같은 언어 서비스를 제공한다면, CMake Tools는 그 위에서 CMake 프로젝트의 생명 주기, 즉 구성(configure) → 빌드(build) → 실행/디버그(launch)를 관리하는 역할을 맡습니다. 두 확장은 서로 협력하도록 설계되어 있어서, CMake Tools가 컴파일 옵션과 include 경로를 찾아내면 그 정보를 C/C++ 확장에 전달해 IntelliSense가 자동으로 맞춰지게 해줍니다.

확장 설치는 3장에서 C/C++ 확장을 설치했던 방법과 동일합니다. VS Code 왼쪽의 확장 아이콘을 클릭하거나 `Ctrl+Shift+X`를 눌러 확장 뷰를 열고 검색창에 `CMake Tools`를 입력합니다. 게시자가 Microsoft로 표시되는 확장을 찾아 설치 버튼을 누르면 됩니다. 참고로 `CMake` 확장(문법 하이라이팅과 스니펫 제공)이 의존성으로 함께 설치되는 경우가 많은데, `CMakeLists.txt` 파일을 편집할 때 문법 강조와 자동완성을 제공하는 별도 확장이니 함께 두어도 무방합니다.

## 프로젝트 열기와 자동 인식

CMake Tools를 설치한 상태에서 최상위 폴더에 `CMakeLists.txt`가 있는 프로젝트를 VS Code로 열면, 확장이 이를 자동으로 감지합니다. 처음 여는 경우 VS Code 하단에 킷을 선택하라는 알림이 뜨거나, 상태 바에 킷 선택 항목이 바로 나타납니다. 이 상태 바는 화면 맨 아래에 위치하며, 왼쪽부터 대략 다음과 같은 항목들이 순서대로 나타납니다.

- 현재 선택된 킷(컴파일러 조합) 이름
- 빌드 variant(예: Debug, Release)
- 활성 타겟(빌드할 대상)을 지정하는 항목
- 빌드 버튼(톱니바퀴 모양 또는 "Build" 텍스트)
- 디버그 버튼(벌레 모양 아이콘)
- 실행 버튼(재생 모양 아이콘)

7장에서 다룬 대로 프로젝트 폴더 구조가 잘 잡혀 있다면, CMake Tools는 별다른 설정 없이도 이 상태 바만으로 구성, 빌드, 디버그까지 전부 처리할 수 있습니다.

## Kit 선택하기

CMake Tools에서 "킷(Kit)"이란 컴파일러, 링커, 그 밖의 도구 모음을 하나로 묶은 조합을 가리키는 용어입니다. 2장에서 설치한 GCC나 MinGW-w64 툴체인이 시스템 경로(PATH)에 잡혀 있으면, CMake Tools는 이를 자동으로 검색해 킷 목록에 올려 둡니다.

킷을 선택하는 방법은 다음과 같습니다.

1. `Ctrl+Shift+P`로 커맨드 팔레트를 엽니다.
2. `CMake: Select a Kit`을 입력하고 실행합니다.
3. 검색된 킷 목록이 나타납니다. 예를 들어 Windows에서 MinGW-w64를 설치했다면 `GCC 13.2.0 x86_64-w64-mingw32` 같은 항목이 보이고, 리눅스라면 `GCC 12.2.0 x86_64-linux-gnu` 형태로 표시됩니다.
4. 사용할 킷을 클릭하면 선택이 끝나고, 상태 바의 킷 이름이 갱신됩니다.

만약 원하는 컴파일러가 목록에 나타나지 않는다면, 해당 컴파일러의 `bin` 디렉터리가 PATH에 등록되어 있는지부터 확인합니다. 2장에서 GCC를 설치한 뒤 `g++ --version`이 터미널에서 바로 실행되었다면 PATH 문제는 없을 가능성이 높습니다. 그래도 목록에 나타나지 않으면 `CMake: Scan for Kits` 명령으로 다시 검색을 시도할 수 있습니다.

킷을 처음 선택하면 CMake Tools는 자동으로 구성(configure) 단계를 한 번 실행합니다. 이는 커맨드라인에서 `cmake -S . -B build -G Ninja`를 실행하는 것과 사실상 동일한 작업이며, 결과로 빌드 디렉터리(기본값은 프로젝트 루트의 `build` 폴더)가 생성되고 그 안에 `CMakeCache.txt`, `build.ninja` 같은 파일들이 만들어집니다.

## 제너레이터는 어떻게 정해지는가

8장에서 살펴본 것처럼 CMake는 Ninja, Makefile, Visual Studio 솔루션 등 여러 빌드 시스템(제너레이터)을 대상으로 빌드 스크립트를 생성할 수 있습니다. 커맨드라인에서는 `-G Ninja`처럼 직접 지정해야 했지만, CMake Tools는 선택한 킷과 플랫폼을 바탕으로 적절한 기본 제너레이터를 자동으로 고릅니다. Ninja가 시스템에 설치되어 있으면 대개 Ninja를 우선적으로 선택하는데, 이는 8장에서 설명했듯 Ninja가 병렬 빌드와 증분 빌드에서 Make류 제너레이터보다 빠른 경우가 많기 때문입니다.

기본 제너레이터를 바꾸고 싶다면 `.vscode/settings.json`에 다음처럼 `cmake.generator` 설정을 추가하면 됩니다.

```json
{
  "cmake.generator": "Ninja"
}
```

이 설정이 없으면 CMake Tools가 알아서 선택하므로, 특별한 이유가 없다면 굳이 지정하지 않아도 됩니다.

## 상태 바로 Configure/Build/Debug 실습하기

이제 7장에서 만든 것과 비슷한 간단한 CMake 프로젝트로 실습해 봅니다. 프로젝트 구조는 다음과 같다고 가정합니다.

```
hello-cmake-tools/
├── CMakeLists.txt
└── src/
    └── main.cpp
```

`CMakeLists.txt`는 다음과 같이 작성합니다.

```cmake
cmake_minimum_required(VERSION 3.20)
project(hello_cmake_tools LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

add_executable(hello_cmake_tools src/main.cpp)
```

`src/main.cpp`는 다음과 같이 작성합니다.

```cpp
#include <iostream>

int main() {
    std::cout << "CMake Tools에서 실행되었습니다.\n";
    return 0;
}
```

### 1단계: Configure

킷을 선택하면 구성이 자동으로 한 번 일어나지만, `CMakeLists.txt`를 수정한 뒤에는 다시 구성을 실행해야 할 때가 있습니다. 커맨드 팔레트에서 `CMake: Configure`를 실행하거나, 상태 바의 프로젝트 이름 옆 아이콘을 눌러 수동으로 구성을 다시 실행할 수 있습니다. 구성이 진행되는 동안 하단의 출력 패널(Output 탭에서 "CMake/Build" 채널 선택)에 `cmake` 명령이 실제로 어떤 옵션으로 실행되었는지 로그가 출력되므로, 문제가 생기면 이 로그를 먼저 확인하는 것이 좋습니다.

### 2단계: Build

상태 바의 "Build" 버튼(톱니바퀴 아이콘, 텍스트로는 현재 활성 타겟 이름과 함께 표시됨)을 클릭하면 빌드가 시작됩니다. 이는 커맨드라인의 `cmake --build build`와 동일한 작업이며, 실제로 Ninja가 선택되어 있다면 내부적으로 `ninja` 명령이 실행됩니다. 여러 타겟이 있는 프로젝트라면 상태 바에서 빌드할 타겟을 먼저 선택할 수 있고, 기본값은 `all`(또는 이에 대응하는 전체 빌드) 타겟입니다.

빌드 결과는 출력 패널에 그대로 표시되며, 컴파일 오류가 발생하면 4장에서 익힌 것과 동일하게 VS Code가 오류가 발생한 줄로 이동할 수 있는 링크를 제공합니다.

### 3단계: Run/Debug

상태 바의 재생 아이콘(디버깅 없이 실행)이나 벌레 아이콘(디버거를 붙여서 실행)을 클릭하면 CMake Tools가 자동으로 실행 파일을 찾아 실행합니다. 디버그로 실행하면 4장에서 `launch.json`을 직접 작성해서 연결했던 GDB(또는 lldb) 디버거가 자동으로 붙어서, 브레이크포인트를 걸어둔 위치에서 멈추고 변수 값을 확인하는 등 동일한 디버깅 경험을 그대로 사용할 수 있습니다.

여기서 중요한 점은, CMake Tools가 실행 파일의 정확한 경로(예: `build/hello_cmake_tools` 또는 `build/hello_cmake_tools.exe`)를 자동으로 추적해서 디버거에 전달해 준다는 것입니다. 4장에서는 이 경로를 `launch.json`의 `program` 필드에 직접 적어 넣어야 했지만, CMake Tools를 쓰면 타겟이 바뀌거나 빌드 디렉터리 구조가 바뀌어도 신경 쓸 필요가 없습니다.

## 자동화되는 것과 여전히 알아야 하는 것

정리하면 CMake Tools 확장은 다음 파일들을 사람이 직접 만들거나 갱신하지 않아도 되게 해줍니다.

| 이전 방식 (4~8장) | CMake Tools 사용 시 |
|---|---|
| `tasks.json`에 빌드 태스크 정의 | 상태 바 "Build" 버튼 클릭 |
| `launch.json`에 실행 파일 경로 지정 | 상태 바 디버그 버튼이 경로 자동 추적 |
| `c_cpp_properties.json`에 include 경로 수동 등록 | CMake 캐시에서 컴파일 옵션을 읽어 IntelliSense에 자동 전달 |
| `cmake -G Ninja` 제너레이터 직접 지정 | 킷과 환경에 맞춰 제너레이터 자동 선택 |

다만 확장이 이 파일들을 "대신 만들어 준다"기보다는, 별도 설정 파일 없이도 같은 결과를 내도록 CMake 캐시와 컴파일 데이터베이스(`compile_commands.json`)를 직접 활용한다고 보는 편이 정확합니다. 그렇기 때문에 프로젝트를 다른 컴퓨터로 옮기거나 CI 환경에서 빌드할 때는 결국 6~8장에서 배운 순수 CMake 커맨드라인 지식이 필요합니다. CMake Tools는 VS Code 안에서의 반복 작업을 줄여 주는 도구이지, CMake 자체를 대체하는 도구가 아니라는 점을 기억해 두면 좋습니다.

## CMakePresets.json 살짝 맛보기

프로젝트마다 구성 옵션(빌드 디렉터리 위치, 컴파일러, 캐시 변수 등)을 매번 상태 바에서 다시 고르는 대신, 이런 조합을 이름 붙여 파일로 저장해 두고 재사용할 수 있습니다. 이를 위한 CMake 표준 기능이 `CMakePresets.json`이며, CMake Tools는 이 파일이 프로젝트 루트에 있으면 자동으로 인식해서 킷 대신 프리셋을 선택하는 UI로 전환합니다.

간단한 예시는 다음과 같습니다.

```json
{
  "version": 6,
  "configurePresets": [
    {
      "name": "default",
      "generator": "Ninja",
      "binaryDir": "${sourceDir}/build",
      "cacheVariables": {
        "CMAKE_BUILD_TYPE": "Debug"
      }
    }
  ]
}
```

이 파일을 프로젝트 루트에 두면 커맨드 팔레트에서 `CMake: Select Configure Preset`으로 `default` 프리셋을 선택할 수 있고, 이후 구성 단계는 이 파일에 적힌 옵션을 그대로 사용합니다. 프리셋은 팀원들과 빌드 설정을 정확히 동일하게 공유하고 싶을 때, 또는 운영체제나 컴파일러별로 여러 구성을 나란히 관리하고 싶을 때 특히 유용합니다. 프리셋을 이용한 멀티플랫폼 빌드 구성, 여러 프리셋의 상속 관계, 빌드/테스트 프리셋까지 포함한 전체 구조는 16장 "멀티플랫폼 빌드와 CMake Presets"에서 자세히 다룹니다. 지금은 이런 파일이 존재하고, CMake Tools가 이를 인식해서 킷 선택 대신 프리셋 선택으로 작업 흐름을 바꿔 준다는 정도만 기억해 두면 충분합니다.

## 요약

- CMake Tools는 마이크로소프트가 배포하는 VS Code 확장으로, CMake 프로젝트의 구성·빌드·디버그 실행을 상태 바 버튼과 커맨드 팔레트로 처리할 수 있게 해줍니다.
- `Ctrl+Shift+P` → `CMake: Select a Kit`으로 사용할 컴파일러 조합(킷)을 선택하며, 시스템 PATH에 등록된 GCC/MinGW 등을 자동으로 검색합니다.
- 킷을 선택하면 자동으로 구성이 실행되고, 이후 상태 바의 Build/Debug/Run 버튼만으로 4~8장에서 손으로 다루던 `tasks.json`, `launch.json`, `c_cpp_properties.json`, 제너레이터 지정 작업 상당 부분을 대신할 수 있습니다.
- CMake Tools는 CMake 자체를 대체하지 않으며, 내부적으로 동일한 `cmake`/`ninja` 명령을 실행하는 것이므로 커맨드라인 지식은 여전히 유용합니다.
- `CMakePresets.json`을 두면 구성 옵션을 이름 붙여 저장하고 재사용할 수 있으며, 자세한 내용은 16장에서 다룹니다.

## 연습문제

1. CMake Tools 확장을 설치한 뒤, `CMake: Select a Kit` 명령을 실행해서 시스템에 어떤 킷들이 검색되는지 확인해 보세요. 원하는 컴파일러가 목록에 없다면 그 이유를 추정해 보세요.
2. 본문의 `hello_cmake_tools` 예제를 만들어 상태 바의 Build 버튼과 커맨드라인의 `cmake --build build` 명령이 각각 어떤 로그를 출력하는지 비교해 보세요.
3. 상태 바에서 빌드 variant를 Debug에서 Release로 바꾼 뒤 다시 빌드하면 `CMakeCache.txt`의 `CMAKE_BUILD_TYPE` 값이 어떻게 바뀌는지 확인해 보세요.
4. 프로젝트에 두 번째 실행 파일 타겟을 추가하고, 상태 바에서 활성 타겟을 전환해 가며 각각 빌드·디버그 실행해 보세요.
5. 본문의 `CMakePresets.json` 예제를 프로젝트에 추가한 뒤 `CMake: Select Configure Preset` 명령으로 선택해 보고, 킷을 직접 선택했을 때와 상태 바 표시가 어떻게 달라지는지 관찰해 보세요.

---

[◀ 이전: 8장. Ninja 빌드 시스템](ch08-Ninja빌드시스템.md) | [📖 목차](00-목차.md) | [다음: 10장. vcpkg 소개와 설치 ▶](ch10-vcpkg소개와설치.md)
