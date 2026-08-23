# 부록

[◀ 이전: 18장. 실전 프로젝트 구성과 좋은 습관](ch18-실전프로젝트구성과좋은습관.md) | [📖 목차](00-목차.md)


## 자주 발생하는 오류와 해결 가이드

C++ 개발환경은 여러 도구가 맞물려 동작하기 때문에, 문제가 생겼을 때 어느 도구가 원인인지 파악하는 것 자체가 초반의 가장 큰 장벽입니다. 여기서는 이 책을 따라오면서 특히 자주 마주치게 되는 문제 다섯 가지를 정리합니다.

### 1. "헤더를 찾을 수 없음" — IntelliSense 문제인가, 실제 빌드 문제인가

VS Code에서 `#include` 아래에 빨간 물결선이 뜨면서 "파일을 찾을 수 없습니다"라는 경고가 나타나는 경우가 있습니다. 이때 가장 먼저 확인해야 할 것은 **이 에러가 편집 중에 뜨는 IntelliSense의 경고인지, 아니면 실제로 빌드를 실행했을 때 컴파일러가 내는 에러인지**를 구분하는 일입니다. 이 둘은 원인과 해결 방법이 완전히 다릅니다.

구분하는 방법은 간단합니다. 빨간 물결선을 무시하고 실제로 `cmake --build` 또는 터미널에서 `g++` 명령을 실행해 봅니다.

- **빌드는 성공하는데 에디터에서만 빨간 줄이 뜬다면**, 이는 5장에서 다룬 IntelliSense 설정 문제입니다. `.vscode/c_cpp_properties.json`의 `includePath`가 실제 프로젝트의 include 경로와 다르거나, CMake Tools 확장을 쓰는 경우 IntelliSense 구성 공급자가 아직 최신 `compile_commands.json`을 읽어오지 못한 상태일 수 있습니다. CMake Tools를 쓰고 있다면 명령 팔레트에서 "CMake: Configure"를 다시 실행해 보고, 그래도 해결되지 않으면 VS Code 창을 재시작해 봅니다.
- **빌드 자체가 실패한다면**, 이는 진짜 컴파일 문제입니다. `CMakeLists.txt`의 `target_include_directories`나 `target_link_libraries`가 해당 헤더가 있는 위치를 실제로 포함하고 있는지 확인해야 합니다. vcpkg로 설치한 라이브러리의 헤더라면 아래 2번 항목을 참고합니다.

이 두 가지를 헷갈리면 IntelliSense 설정만 계속 고치면서 정작 빌드 자체는 계속 실패하는 상황이 반복될 수 있으므로, "지금 겪고 있는 문제가 에디터의 경고인가, 실제 빌드 결과인가"를 항상 먼저 확인하는 습관을 들이는 것이 좋습니다.

### 2. vcpkg로 설치한 라이브러리를 CMake가 찾지 못함

`find_package(...)`가 "패키지를 찾을 수 없습니다"라는 에러를 내는 경우, vcpkg로 라이브러리를 분명히 설치했는데도 CMake가 이를 인식하지 못하는 상황이라면 거의 대부분 **toolchain file이 지정되지 않았기 때문**입니다. vcpkg가 설치한 라이브러리는 시스템 표준 경로가 아니라 vcpkg 전용 디렉터리에 위치하므로, CMake에게 그 위치를 알려주는 toolchain file을 명시적으로 지정해야 합니다.

커맨드라인에서 직접 구성한다면 다음과 같이 지정합니다.

```bash
cmake --preset default -DCMAKE_TOOLCHAIN_FILE=[vcpkg 설치 경로]/scripts/buildsystems/vcpkg.cmake
```

이 책에서처럼 `CMakePresets.json`을 쓰고 있다면 매번 이 옵션을 손으로 입력하는 대신 프리셋 안에 `cacheVariables`로 등록해 두는 편이 안전합니다.

```json
{
  "cacheVariables": {
    "CMAKE_TOOLCHAIN_FILE": "$env{VCPKG_ROOT}/scripts/buildsystems/vcpkg.cmake"
  }
}
```

toolchain file을 지정했는데도 여전히 찾지 못한다면, 다음 두 가지를 추가로 점검합니다. 첫째, manifest 모드(`vcpkg.json` 사용)인 경우 라이브러리 이름이 `vcpkg.json`의 `dependencies` 목록에 실제로 들어 있는지 확인합니다. 둘째, `find_package`에 넘긴 이름이 vcpkg 패키지 이름과 다를 수 있다는 점입니다. 예를 들어 vcpkg 패키지 이름은 `catch2`이지만 CMake에서 찾을 때는 `find_package(Catch2 ...)`처럼 대소문자와 표기가 다른 경우가 흔하므로, 해당 라이브러리의 vcpkg 문서에서 정확한 `find_package` 사용 예시를 확인하는 것이 가장 확실합니다.

### 3. Windows에서 MSYS2/MinGW의 PATH 문제

Windows에서 MSYS2로 GCC를 설치한 경우, 터미널을 새로 열 때마다 `g++`나 `cmake`, `ninja` 명령이 "인식할 수 없는 명령입니다"라는 에러를 내는 경우가 있습니다. 이는 대부분 MSYS2가 설치된 경로(예: `C:\msys64\ucrt64\bin`)가 시스템 PATH 환경변수에 등록되어 있지 않기 때문입니다.

특히 헷갈리는 지점은 MSYS2가 여러 개의 서로 다른 환경(UCRT64, MINGW64, CLANG64 등)을 제공한다는 점입니다. 이 중 하나의 환경에서 도구를 설치했다면, PATH에 등록해야 하는 경로도 정확히 그 환경의 `bin` 디렉터리여야 합니다. 예를 들어 UCRT64 환경에 설치했는데 PATH에는 MINGW64 경로만 등록되어 있으면, 명령을 찾지 못하거나 엉뚱한 버전의 컴파일러가 호출될 수 있습니다.

또한 VS Code의 통합 터미널은 시스템 PATH를 읽어오는 시점이 터미널을 새로 열 때이므로, PATH를 새로 등록한 뒤에는 기존에 열려 있던 VS Code 창을 완전히 재시작해야 변경 사항이 반영됩니다. 단순히 터미널 패널만 닫았다 여는 것으로는 부족한 경우가 있으니, 문제가 계속되면 VS Code 자체를 재시작해 보는 것이 좋습니다.

### 4. CMake Tools가 잘못된 컴파일러(kit)를 자동으로 선택함

한 컴퓨터에 GCC, MSVC, Clang처럼 여러 컴파일러가 함께 설치되어 있는 경우, CMake Tools 확장이 스캔 과정에서 의도하지 않은 컴파일러를 기본 kit으로 선택하는 경우가 있습니다. 이 경우 IntelliSense나 빌드 결과가 예상과 다르게 나타날 수 있습니다.

명령 팔레트에서 "CMake: Select a Kit"을 실행하면 현재 인식된 kit 목록을 확인하고 원하는 컴파일러를 직접 선택할 수 있습니다. 목록에 원하는 컴파일러가 보이지 않는다면 "CMake: Scan for Kits"를 먼저 실행해 다시 검색하게 합니다. `CMakePresets.json`을 사용하는 프로젝트라면 프리셋의 `cacheVariables`에 `CMAKE_CXX_COMPILER` 경로를 직접 명시해서 kit 자동 선택에 의존하지 않는 방법도 있으며, 팀 전체가 동일한 컴파일러를 쓰도록 강제하고 싶을 때 특히 유용합니다.

### 5. 캐시가 꼬여서 설정을 바꿔도 반영되지 않음

`CMakeLists.txt`나 `CMakePresets.json`을 수정했는데도 빌드 결과가 전혀 바뀌지 않거나, 이해할 수 없는 에러가 계속 반복되는 경우가 있습니다. 이런 증상은 대부분 CMake가 이전 구성 결과를 `CMakeCache.txt`에 캐시해 두고 있기 때문입니다. 특히 toolchain file 경로를 바꾸거나 컴파일러 자체를 교체한 경우, 기존 캐시가 새 설정과 충돌하면서 이상한 에러 메시지를 내는 경우가 흔합니다.

가장 확실한 해결 방법은 빌드 디렉터리를 통째로 삭제하고 처음부터 다시 구성하는 것입니다.

```bash
rm -rf build
cmake --preset default
```

CMake Tools 확장을 쓰고 있다면 명령 팔레트의 "CMake: Delete Cache and Reconfigure" 명령이 같은 작업을 한 번에 처리해 줍니다. 원인을 알 수 없는 CMake 에러를 만났을 때는 코드를 붙잡고 오래 씨름하기보다, 캐시를 지우고 다시 구성하는 것부터 시도해 보는 편이 시간을 절약하는 경우가 많습니다.

## 명령어와 설정 파일 요약

이 책 전체에서 사용한 주요 명령어와 설정 파일의 역할을 표로 정리합니다.

### 명령어

| 명령어 | 역할 |
|---|---|
| `g++ 파일.cpp -o 실행파일` | GCC의 C++ 프론트엔드로 소스 파일을 직접 컴파일하고 링크해서 실행 파일을 만듭니다. |
| `g++ -std=c++17 -Wall -Wextra` | 언어 표준(13장)과 경고 수준을 지정해서 컴파일합니다. |
| `cmake -S . -B build` | 소스 디렉터리(`-S`)와 빌드 디렉터리(`-B`)를 지정해서 빌드 규칙을 구성합니다. |
| `cmake --preset [이름]` | `CMakePresets.json`에 정의된 프리셋 이름으로 구성 단계를 실행합니다(16장). |
| `cmake --build --preset [이름]` | 프리셋에 대응하는 빌드 디렉터리에서 실제 빌드를 수행합니다. |
| `ninja` | `build.ninja` 파일을 읽어 변경된 부분만 다시 컴파일합니다(8장). |
| `vcpkg install [패키지명]` | classic 모드에서 라이브러리를 직접 설치합니다(10장). |
| `vcpkg new --application` | 현재 디렉터리에 manifest 모드용 `vcpkg.json` 뼈대를 생성합니다(12장). |
| `ctest --test-dir build` | 빌드 디렉터리에 등록된 테스트를 실행합니다(17장). |
| `ctest --output-on-failure` | 실패한 테스트가 있을 때만 상세 출력을 보여주며 실행합니다. |
| `clang-format -i 파일.cpp` | `.clang-format` 설정에 따라 파일을 그 자리에서 정리합니다(15장). |
| `clang-tidy 파일.cpp` | `.clang-tidy` 설정에 따라 잠재적 버그와 스타일 문제를 검사합니다(15장). |
| `gdb ./실행파일` | GDB 디버거로 실행 파일을 직접 실행하고 중단점을 걸어 조사합니다(14장). |

### 설정 파일

| 파일 | 역할 |
|---|---|
| `CMakeLists.txt` | 프로젝트의 소스 구성, 타깃, 의존성, 컴파일 옵션을 선언하는 CMake 설계도입니다(6, 7장). |
| `CMakePresets.json` | 컴파일러, 빌드 디렉터리, 생성기(Ninja 등), 캐시 변수 조합을 이름 붙여 저장해 여러 환경에서 재현 가능하게 합니다(16장). |
| `vcpkg.json` | 프로젝트가 의존하는 외부 라이브러리 목록을 manifest 모드로 선언합니다(12장). |
| `.clang-format` | 들여쓰기, 줄바꿈, 중괄호 위치 등 코드 스타일 규칙을 정의합니다(15장). |
| `.clang-tidy` | 활성화할 정적 분석 검사 항목과 예외 규칙을 정의합니다(15장). |
| `c_cpp_properties.json` | VS Code C/C++ 확장의 IntelliSense가 사용할 include 경로와 컴파일러 정보를 정의합니다(5장). |
| `launch.json` | VS Code에서 디버거를 어떤 실행 파일과 인자로 실행할지 정의합니다(4장). |
| `tasks.json` | VS Code에서 빌드 등 반복 작업을 실행하는 태스크를 정의합니다(4장). |
| `.gitignore` | `build/`, `vcpkg_installed/`처럼 버전 관리에서 제외할 파일과 디렉터리를 지정합니다(18장). |

## 더 참고할 공식 문서

이 책에서 다룬 각 도구는 모두 활발히 유지되는 공식 문서를 갖추고 있습니다. 버전이 올라가면서 세부 옵션이나 문법이 조금씩 바뀔 수 있으므로, 이 책에서 다루지 않은 세부 옵션이 궁금하거나 최신 변경 사항을 확인하고 싶을 때는 아래 공식 문서를 직접 찾아보는 것을 권합니다.

- CMake 공식 문서: cmake.org
- vcpkg 공식 문서: vcpkg.io (또는 vcpkg 공식 저장소인 GitHub의 microsoft/vcpkg)
- VS Code의 C++ 지원 공식 문서: code.visualstudio.com/docs/cpp
- GCC 공식 문서: gcc.gnu.org
- Ninja 공식 문서: ninja-build.org
- Catch2 공식 문서: GitHub의 catchorg/Catch2 저장소 안의 문서 디렉터리
- GoogleTest 공식 문서: google.github.io/googletest
- LLVM/Clang 도구(clang-format, clang-tidy) 공식 문서: clang.llvm.org

## 다음 학습 단계

이 책을 마쳤다면, 다음 단계로는 실제로 여러 개의 외부 라이브러리(JSON 파싱, 로깅, 네트워크 등)를 조합한 조금 더 규모 있는 개인 프로젝트를 직접 처음부터 끝까지 구성해 보는 것을 권합니다. 이 책의 각 장에서 다룬 개념을 하나씩 실제로 적용해 보면서, 어느 부분이 아직 손에 익지 않았는지 스스로 확인할 수 있습니다.

그다음으로는 18장에서 짧게 언급한 GitHub Actions를 통한 빌드·테스트 자동화를 시도해 보는 것을 추천합니다. 로컬에서 `cmake --preset`, `cmake --build --preset`, `ctest`로 해왔던 과정을 CI 환경에서 그대로 재현해 보면, 이 책에서 CMake Presets를 강조했던 이유(환경에 상관없이 동일한 빌드를 재현하는 것)를 실감 있게 확인할 수 있습니다.

마지막으로, 이미 존재하는 규모 있는 오픈소스 C++ 프로젝트의 저장소를 클론해서 그 프로젝트의 `CMakeLists.txt`와 `CMakePresets.json`, `vcpkg.json` 또는 `conanfile.txt`를 직접 읽어보는 것도 좋은 학습 방법입니다. 이 책에서 배운 개념들이 실제 프로젝트에서 어떤 방식으로 조합되어 쓰이는지 살펴보면, 각 도구의 문법을 넘어 "실전에서 통용되는 관례"까지 함께 익힐 수 있습니다.

---

[◀ 이전: 18장. 실전 프로젝트 구성과 좋은 습관](ch18-실전프로젝트구성과좋은습관.md) | [📖 목차](00-목차.md)
