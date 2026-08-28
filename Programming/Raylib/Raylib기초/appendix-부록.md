# 부록

[◀ 이전: 18장. 실전 프로젝트: 미니 게임 만들기](ch18-실전프로젝트미니게임만들기.md) | [📖 목차](00-목차.md)


## A. raylib 설치 방법

2장에서는 Windows 환경에서 w64devkit을 이용한 가장 간단한 설치 방법을 다뤘습니다. 여기서는 주요 플랫폼별로 raylib를 설치하는 방법을 조금 더 자세히 정리합니다. 크게 "공식에서 미리 컴파일해둔 바이너리를 받는 방법"과 "패키지 관리자(vcpkg 등)를 이용하는 방법" 두 갈래로 나뉩니다.

### Windows

**방법 1: 공식 사전 빌드 바이너리**

raylib 공식 웹사이트나 GitHub 릴리스 페이지에서 `raylib-x.x.x_win64_mingw-w64.zip`처럼 컴파일러별로 미리 빌드된 압축 파일을 내려받을 수 있습니다. 2장에서 사용한 w64devkit 조합(MinGW-w64 GCC)을 그대로 쓴다면, 압축을 풀어 나온 `include`, `lib` 폴더를 컴파일러가 찾을 수 있는 경로에 두고 컴파일 시 다음처럼 연결합니다.

```bash
gcc main.c -o main.exe -I C:/raylib/include -L C:/raylib/lib -lraylib -lopengl32 -lgdi32 -lwinmm
```

**방법 2: vcpkg 사용**

마이크로소프트가 관리하는 C/C++ 패키지 관리자 vcpkg로도 raylib를 설치할 수 있습니다. Visual Studio나 CMake 기반 프로젝트를 쓴다면 이 방법이 의존성 관리가 한결 편합니다.

```bash
git clone https://github.com/microsoft/vcpkg
.\vcpkg\bootstrap-vcpkg.bat
.\vcpkg\vcpkg install raylib
```

CMake 프로젝트라면 툴체인 파일을 지정해 vcpkg가 설치한 raylib를 바로 찾게 할 수 있습니다.

```bash
cmake -B build -DCMAKE_TOOLCHAIN_FILE=[vcpkg가 설치된 경로]/scripts/buildsystems/vcpkg.cmake
```

### macOS

Homebrew를 쓰는 것이 가장 간단합니다.

```bash
brew install raylib
```

vcpkg를 macOS에서도 동일하게 사용할 수 있습니다.

```bash
git clone https://github.com/microsoft/vcpkg
./vcpkg/bootstrap-vcpkg.sh
./vcpkg/vcpkg install raylib
```

컴파일 시에는 raylib가 의존하는 macOS 프레임워크들을 함께 링크해야 합니다.

```bash
clang main.c -o main -lraylib -framework OpenGL -framework Cocoa -framework IOKit -framework CoreVideo
```

### Linux

배포판의 패키지 관리자에 raylib가 포함되어 있는 경우도 있지만(예: 일부 배포판의 `libraylib-dev`), 버전이 오래되어 있을 수 있으므로 최신 기능을 쓰고 싶다면 소스에서 직접 빌드하는 방법을 권합니다.

```bash
git clone https://github.com/raysan5/raylib.git
cd raylib/src
make PLATFORM=PLATFORM_DESKTOP
sudo make install
```

vcpkg 역시 Linux에서 macOS와 동일한 방식으로 사용할 수 있습니다.

```bash
git clone https://github.com/microsoft/vcpkg
./vcpkg/bootstrap-vcpkg.sh
./vcpkg/vcpkg install raylib
```

빌드에 필요한 개발 패키지(X11, OpenGL 관련 헤더 등)가 미리 설치되어 있어야 하므로, 빌드가 실패한다면 배포판의 문서에서 raylib 빌드 요구사항을 먼저 확인하는 것이 좋습니다.

## B. 이 책에서 사용한 주요 함수 요약

이 책 전체에서 등장한 raylib 함수 중 특히 자주 쓰인 것들을 장별로 정리했습니다. 정확한 매개변수는 각 장의 본문이나 raylib 공식 cheatsheet를 참고하세요.

| 장 | 함수 | 설명 |
|---|---|---|
| 2장 | `InitWindow(width, height, title)` | 창을 생성하고 그래픽 컨텍스트를 초기화한다 |
| 2장 | `CloseWindow()` | 창을 닫고 관련 리소스를 해제한다 |
| 2장 | `SetTargetFPS(fps)` | 게임 루프가 목표로 하는 초당 프레임 수를 설정한다 |
| 3장 | `WindowShouldClose()` | 사용자가 창을 닫으려 했는지 확인한다 |
| 3장 | `BeginDrawing()` / `EndDrawing()` | 화면 그리기 시작/종료를 표시한다 |
| 3장 | `ClearBackground(color)` | 화면을 지정한 색으로 지운다 |
| 3장 | `DrawRectangle(x, y, w, h, color)` | 사각형을 그린다 |
| 3장 | `DrawCircle(x, y, radius, color)` | 원을 그린다 |
| 4장 | `IsKeyDown(key)` / `IsKeyPressed(key)` | 키가 눌려있는지 / 이번 프레임에 처음 눌렸는지 확인한다 |
| 4장 | `GetMousePosition()` | 현재 마우스 좌표를 `Vector2`로 반환한다 |
| 4장 | `IsMouseButtonPressed(button)` | 마우스 버튼이 이번 프레임에 눌렸는지 확인한다 |
| 5장 | `Vector2Add(v1, v2)` / `Vector2Scale(v, scale)` | 벡터 덧셈 / 스칼라 곱 (raymath.h) |
| 5장 | `Vector2Normalize(v)` | 벡터를 길이 1인 방향 벡터로 정규화한다 |
| 5장 | `Vector2Distance(v1, v2)` | 두 점 사이의 거리를 계산한다 |
| 6장 | `LoadTexture(fileName)` / `UnloadTexture(texture)` | 이미지 파일을 텍스처로 불러오고 해제한다 |
| 6장 | `DrawTexture(texture, x, y, tint)` / `DrawTextureRec(texture, source, position, tint)` | 텍스처 전체 또는 일부 영역을 그린다 |
| 7장 | `GetFrameTime()` | 직전 프레임을 그리는 데 걸린 시간(초)을 반환한다 |
| 7장 | `DrawTexturePro(texture, source, dest, origin, rotation, tint)` | 회전·크기 조절·반전까지 지원하는 텍스처 그리기 |
| 8장 | `CheckCollisionRecs(rec1, rec2)` | 두 사각형이 겹치는지 확인한다 |
| 8장 | `CheckCollisionCircles(center1, r1, center2, r2)` | 두 원이 겹치는지 확인한다 |
| 8장 | `CheckCollisionCircleRec(center, radius, rec)` | 원과 사각형이 겹치는지 확인한다 |
| 9장 | `InitAudioDevice()` / `CloseAudioDevice()` | 오디오 장치를 초기화/종료한다 |
| 9장 | `LoadSound(fileName)` / `PlaySound(sound)` / `UnloadSound(sound)` | 효과음을 불러오고 재생하고 해제한다 |
| 9장 | `LoadMusicStream(fileName)` / `UpdateMusicStream(music)` | 스트리밍 음악을 불러오고 매 프레임 갱신한다 |
| 10장 | `LoadFont(fileName)` / `DrawTextEx(font, text, position, fontSize, spacing, tint)` | 사용자 폰트를 불러와 텍스트를 그린다 |
| 10장 | `MeasureTextEx(font, text, fontSize, spacing)` | 지정한 폰트로 텍스트를 그렸을 때의 크기를 계산한다 |
| 11장 | `BeginMode2D(camera)` / `EndMode2D()` | 2D 카메라 좌표계로 그리기를 시작/종료한다 |
| 12장 | `BeginMode3D(camera)` / `EndMode3D()` | 3D 카메라 좌표계로 그리기를 시작/종료한다 |
| 12장 | `DrawCube(position, w, h, l, color)` / `DrawGrid(slices, spacing)` | 3D 큐브와 기준 격자를 그린다 |
| 13장 | `LoadModel(fileName)` / `DrawModel(model, position, scale, tint)` / `UnloadModel(model)` | 3D 모델을 불러오고 그리고 해제한다 |
| 14장 | `LoadShader(vsFileName, fsFileName)` / `BeginShaderMode(shader)` / `EndShaderMode()` | 커스텀 셰이더를 불러와 적용/해제한다 |
| 16장 | `DrawCircleV(center, radius, color)` / `Fade(color, alpha)` | 파티클 등을 그릴 때 자주 쓰이는 원 그리기와 반투명 색상 처리 |
| 17장 | `LoadFileData` / `SaveFileData` / `UnloadFileData` | 바이너리 파일을 통째로 읽고 쓰고 메모리를 해제한다 |
| 17장 | `LoadFileText` / `SaveFileText` / `UnloadFileText` | 텍스트 파일을 통째로 읽고 쓰고 메모리를 해제한다 |
| 17장 | `FileExists(fileName)` | 지정한 경로에 파일이 있는지 확인한다 |

## C. 참고할 만한 공식 자료

더 깊이 공부하고 싶다면 다음 공식 자료들을 활용하는 것을 권합니다.

- **raylib 공식 웹사이트** (`https://www.raylib.com`): 다운로드 링크, 함수 전체를 한 페이지에 정리한 cheatsheet, 뉴스 등을 제공합니다.
- **raylib GitHub 저장소의 examples 폴더** (`https://github.com/raysan5/raylib`, 저장소 안의 `examples` 디렉터리): 카테고리(core, shapes, textures, audio, models, shaders 등)별로 정리된 공식 예제 코드 수백 개가 들어있습니다. 이 책에서 다룬 개념을 실제로 어떻게 응용하는지 확인하기에 가장 좋은 자료입니다.
- **raylib 커뮤니티(Discord)**: 공식 웹사이트에 안내된 커뮤니티 링크를 통해 raylib 사용자들과 질문을 주고받거나, 다른 사람들이 만든 프로젝트 쇼케이스를 구경할 수 있습니다. 초대 링크는 시점에 따라 바뀔 수 있으므로 공식 웹사이트에서 최신 링크를 확인하는 것을 권합니다.

## D. 다음 학습 단계

이 책을 마쳤다면, 다음과 같은 방향으로 학습을 이어가 볼 수 있습니다.

- **더 큰 게임 프로젝트에 도전하기**: 18장에서 만든 벽돌깨기를 확장하거나, 플랫포머·탑다운 슈팅·퍼즐 게임처럼 새로운 장르를 처음부터 직접 설계해 보세요. 규모가 커질수록 지금까지 배운 상태 관리, 파일 입출력, 충돌 감지가 왜 필요한지 더 몸으로 느끼게 됩니다.
- **ECS(Entity Component System) 아키텍처 학습하기**: 이 책의 예제들은 구조체와 배열, 상태 열거형을 직접 다루는 비교적 단순한 구조를 사용했습니다. 게임의 개체 종류와 상호작용이 크게 늘어나면 ECS라는 설계 방식이 코드를 훨씬 유연하게 만들어줍니다. `entt`(C++), `flecs`(C/C++) 같은 라이브러리를 raylib와 함께 사용해보거나, 간단한 ECS를 직접 구현해보는 것도 좋은 학습이 됩니다.
- **raylib의 다른 언어 바인딩 탐색하기**: raylib의 핵심은 C로 작성되어 있지만, 커뮤니티가 관리하는 다양한 언어 바인딩(Python, Go, Rust, C#, Odin 등)이 존재합니다. C에서 익힌 개념과 API 구조는 바인딩이 바뀌어도 거의 그대로 적용되므로, 평소 즐겨 쓰던 언어가 있다면 그 언어의 raylib 바인딩으로 넘어가 보는 것도 흥미로운 다음 단계입니다.
- **더 낮은 수준으로 내려가보기**: raylib 내부는 `rlgl`이라는 얇은 OpenGL 추상화 레이어 위에 만들어져 있습니다. raylib가 제공하는 고수준 함수 뒤에서 실제로 어떤 그래픽스 API 호출이 일어나는지 궁금해졌다면, `rlgl.h`를 들여다보거나 OpenGL을 직접 공부해보는 것도 좋은 다음 걸음이 됩니다.

---

[◀ 이전: 18장. 실전 프로젝트: 미니 게임 만들기](ch18-실전프로젝트미니게임만들기.md) | [📖 목차](00-목차.md)
