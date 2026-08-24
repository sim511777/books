# 부록

[◀ 이전: 18장. 실전 프로젝트: 설정 에디터 만들기](ch18-실전프로젝트설정에디터.md) | [📖 목차](00-목차.md)


본문에서 다룬 내용 중 다시 찾아보기 좋은 정보들을 한곳에 모았습니다. 설치 방법을 간단히 재정리하고, 이 책에서 사용한 주요 함수를 장별로 표로 정리했으며, 더 공부할 때 참고할 공식 자료와 다음 학습 단계를 안내합니다.

## 설치 방법 요약

2장에서 다룬 Dear ImGui 설치 방법을 간단히 다시 정리합니다. 두 가지 방식 모두 실무에서 널리 쓰이며, 프로젝트의 성격에 따라 선택하면 됩니다.

### 방식 1: 소스 직접 포함(소스 드롭인) 방식

Dear ImGui 저장소를 내려받아 핵심 소스 파일(`imgui.cpp`, `imgui_draw.cpp`, `imgui_tables.cpp`, `imgui_widgets.cpp`, `imgui_demo.cpp`)과 헤더를 프로젝트 폴더에 직접 복사해 넣고, 사용할 렌더링 백엔드와 플랫폼 백엔드에 해당하는 파일(`backends/` 폴더의 `imgui_impl_glfw.cpp`, `imgui_impl_opengl3.cpp` 등)을 함께 포함시켜 직접 컴파일하는 방식입니다.

- 장점: 외부 패키지 관리자에 의존하지 않고, 라이브러리 버전을 프로젝트 안에서 완전히 통제할 수 있습니다. 소스 코드를 필요에 따라 직접 수정해서 쓰기도 쉽습니다.
- 단점: 새 버전으로 업그레이드할 때 파일을 수동으로 교체해야 하고, 여러 프로젝트에서 공유하기가 상대적으로 번거롭습니다.
- 이 책의 2장 예제는 이 방식을 기준으로 진행했습니다.

### 방식 2: 패키지 관리자(vcpkg) 방식

vcpkg 같은 C++ 패키지 관리자를 사용하면 다음과 같이 설치하고 바로 링크해서 쓸 수 있습니다.

```bash
vcpkg install imgui[glfw-binding,opengl3-binding]
```

- 장점: 버전 관리와 업데이트가 패키지 관리자를 통해 자동화되고, CMake 등 빌드 시스템과의 연동이 간편합니다. 여러 프로젝트에서 같은 설치본을 공유할 수도 있습니다.
- 단점: vcpkg가 제공하는 기능 조합(feature) 이름과 실제로 필요한 백엔드 조합을 미리 확인해야 하고, 소스를 직접 고쳐 쓰기에는 한 단계가 더 들어갑니다.

두 방식 모두 최종적으로 프로그램에 링크되는 코드는 동일한 Dear ImGui이므로, API 사용법이나 이 책에서 다룬 내용에는 차이가 없습니다. 처음 학습할 때는 소스를 직접 눈으로 확인할 수 있는 소스 포함 방식을, 여러 프로젝트를 오가며 안정적으로 버전을 관리하고 싶다면 vcpkg 방식을 권합니다.

## 주요 함수 요약

이 책에서 사용한 핵심 함수와 매크로를 장별로 정리했습니다. 정확한 매개변수와 사용법은 각 장의 본문과 `imgui.h`, `imgui_demo.cpp`를 함께 참고하세요.

### 2장. 개발 환경 설정과 첫 창

| 함수/매크로 | 설명 |
|---|---|
| `ImGui::CreateContext()` | ImGui 컨텍스트를 생성합니다 |
| `ImGui::DestroyContext()` | ImGui 컨텍스트를 해제합니다 |
| `ImGui_ImplGlfw_InitForOpenGL()` | GLFW 플랫폼 백엔드를 초기화합니다 |
| `ImGui_ImplOpenGL3_Init()` | OpenGL3 렌더링 백엔드를 초기화합니다 |
| `ImGui::NewFrame()` / `ImGui::Render()` | 프레임 시작과 드로우 데이터 생성 |

### 3장. Immediate Mode의 핵심 개념

| 함수/매크로 | 설명 |
|---|---|
| `ImGui::Begin()` / `ImGui::End()` | 창을 열고 닫습니다 |
| `ImGui::PushID()` / `ImGui::PopID()` | ID 스택에 식별자를 넣고 빼서 위젯 ID 충돌을 방지합니다 |

### 4장. 기본 위젯

| 함수/매크로 | 설명 |
|---|---|
| `ImGui::Text()` | 서식 있는 텍스트를 표시합니다 |
| `ImGui::Button()` | 클릭 가능한 버튼을 그립니다 |
| `ImGui::Checkbox()` | 불리언 값을 토글하는 체크박스 |
| `ImGui::SliderFloat()` / `ImGui::SliderInt()` | 슬라이더로 값을 조절합니다 |
| `ImGui::InputText()` | 문자열을 입력받습니다 |

### 5장. 창과 레이아웃

| 함수/매크로 | 설명 |
|---|---|
| `ImGui::SameLine()` | 다음 위젯을 같은 줄에 배치합니다 |
| `ImGui::Separator()` | 구분선을 그립니다 |
| `ImGui::Indent()` / `ImGui::Unindent()` | 들여쓰기를 조절합니다 |
| `ImGui::BeginChild()` / `ImGui::EndChild()` | 스크롤 가능한 자식 영역을 만듭니다 |

### 6장. 메뉴와 툴바

| 함수/매크로 | 설명 |
|---|---|
| `ImGui::BeginMainMenuBar()` / `EndMainMenuBar()` | 애플리케이션 최상단 메뉴 바 |
| `ImGui::BeginMenu()` / `EndMenu()` | 하위 메뉴 |
| `ImGui::MenuItem()` | 메뉴 항목(단축키, 체크 상태 지원) |

### 7장. 트리와 리스트

| 함수/매크로 | 설명 |
|---|---|
| `ImGui::TreeNode()` / `ImGui::TreePop()` | 펼침/접힘 가능한 트리 노드 |
| `ImGui::TreeNodeEx()` | 플래그를 지정할 수 있는 트리 노드 |
| `ImGui::CollapsingHeader()` | 접이식 섹션 헤더 |
| `ImGui::BeginListBox()` / `EndListBox()` | 스크롤 가능한 리스트 박스 |
| `ImGui::Selectable()` | 선택 가능한 항목 |

### 8장. 테이블

| 함수/매크로 | 설명 |
|---|---|
| `ImGui::BeginTable()` / `ImGui::EndTable()` | 테이블을 열고 닫습니다 |
| `ImGui::TableSetupColumn()` | 열의 이름과 폭 정책을 지정합니다 |
| `ImGui::TableHeadersRow()` | 헤더 행을 그립니다 |
| `ImGui::TableNextRow()` / `ImGui::TableNextColumn()` | 다음 행/열로 이동합니다 |
| `ImGuiListClipper` | 대량의 행을 화면에 보이는 만큼만 그리는 클리퍼 |

### 9장. 다이얼로그와 팝업

| 함수/매크로 | 설명 |
|---|---|
| `ImGui::OpenPopup()` | 팝업을 여는 요청을 등록합니다 |
| `ImGui::BeginPopup()` / `EndPopup()` | 일반 팝업 |
| `ImGui::BeginPopupModal()` | 모달 팝업(배경 입력 차단) |
| `ImGui::CloseCurrentPopup()` | 현재 팝업을 닫습니다 |

### 10장. 도킹과 멀티 뷰포트

| 함수/매크로 | 설명 |
|---|---|
| `ImGui::DockSpaceOverViewport()` | 뷰포트 전체를 덮는 도킹 스페이스를 만듭니다 |
| `ImGuiDockNodeFlags_PassthruCentralNode` | 중앙 빈 공간의 입력을 배경으로 통과시킵니다 |

### 11장. 커스텀 렌더링과 ImDrawList

| 함수/매크로 | 설명 |
|---|---|
| `ImGui::GetWindowDrawList()` | 현재 창의 드로우 리스트를 가져옵니다 |
| `ImDrawList::AddLine()` / `AddRect()` / `AddCircle()` | 기본 도형을 그립니다 |
| `ImDrawList::AddText()` | 임의 위치에 텍스트를 그립니다 |

### 12장. 스타일링과 테마

| 함수/매크로 | 설명 |
|---|---|
| `ImGui::StyleColorsDark()` / `StyleColorsLight()` | 기본 제공 테마를 적용합니다 |
| `ImGui::PushStyleColor()` / `PopStyleColor()` | 색상을 임시로 덮어씁니다 |
| `ImGui::PushStyleVar()` / `PopStyleVar()` | 여백, 둥글기 등 스타일 변수를 임시로 덮어씁니다 |

### 13장. 입력 처리

| 함수/매크로 | 설명 |
|---|---|
| `ImGui::IsItemHovered()` / `IsItemActive()` | 직전 위젯의 호버/활성 상태를 확인합니다 |
| `ImGui::IsKeyPressed()` | 키보드 입력을 확인합니다 |
| `ImGui::IsMouseClicked()` / `IsMouseDragging()` | 마우스 입력을 확인합니다 |

### 15장. 플롯과 데이터 시각화

| 함수/매크로 | 설명 |
|---|---|
| `ImPlot::CreateContext()` | ImPlot 컨텍스트를 생성합니다 |
| `ImPlot::BeginPlot()` / `EndPlot()` | 플롯 영역을 열고 닫습니다 |
| `ImPlot::PlotLine()` / `PlotBars()` | 선 그래프/막대 그래프를 그립니다 |

### 16장. 다양한 렌더링 백엔드

| 함수/매크로 | 설명 |
|---|---|
| `ImGui_ImplDX11_Init()` | DirectX 11 백엔드를 초기화합니다 |
| `ImGui_ImplVulkan_Init()` | Vulkan 백엔드를 초기화합니다 |
| `ImGui_ImplSDL2_InitForOpenGL()` | SDL2 플랫폼 백엔드를 초기화합니다 |

### 17장. 성능 최적화와 디버깅

| 함수/매크로 | 설명 |
|---|---|
| `ImGui::ShowDemoWindow()` | 내장 데모 창을 표시합니다 |
| `ImGui::ShowMetricsWindow()` | 렌더링/위젯 상태를 보여주는 메트릭 창을 표시합니다 |
| `ImGui::GetIO().Framerate` | 평균 프레임레이트를 확인합니다 |

## 참고할 만한 공식 자료

이 책에서 다루지 못한 세부 사항이나 최신 변경 사항은 다음 공식 자료에서 확인하는 것이 가장 정확합니다.

- **Dear ImGui GitHub 저장소** — ocornut/imgui. 라이브러리 소스 코드, 이슈 트래커, 릴리스 노트, 그리고 각 렌더링/플랫폼 백엔드 소스가 들어있는 `backends/` 폴더를 확인할 수 있습니다. 저장소 주소는 github.com/ocornut/imgui 입니다.
- **imgui_demo.cpp** — 17장에서 소개한 내장 데모 창의 소스 파일로, 저장소를 내려받으면 함께 들어있습니다. 새로운 위젯이나 플래그의 실제 사용 예시를 확인할 때 가장 먼저 열어봐야 할 파일입니다.
- **ImPlot GitHub 저장소** — epezent/implot. 15장에서 다룬 플로팅 라이브러리의 소스와 예제 데모(`implot_demo.cpp`)를 확인할 수 있습니다. 저장소 주소는 github.com/epezent/implot 입니다.
- **Dear ImGui Wiki** — 위 저장소의 GitHub Wiki 페이지에는 백엔드별 설치 가이드, 폰트 로딩 방법, 자주 묻는 질문(FAQ) 등이 정리되어 있습니다.

## 다음 학습 단계

이 책을 마친 뒤에는 다음과 같은 방향으로 학습을 이어가는 것을 권합니다.

- 18장에서 만든 설정 에디터를 자신의 실제 프로젝트(게임, 툴, 시뮬레이터 등)의 디버그 UI로 확장해 직접 통합해보세요.
- `imgui_demo.cpp`와 `imgui_internal.h`를 순서대로 읽으며 ID 스택, 드로우 리스트, 레이아웃 시스템의 내부 구현을 살펴보세요.
- ImPlot 외에도 노드 에디터, 파일 다이얼로그 등 커뮤니티에서 널리 쓰이는 ImGui 확장 라이브러리들을 검색해보고, 자신의 프로젝트에 필요한 것이 있는지 살펴보세요.
- Dear ImGui GitHub 저장소의 이슈와 토론(Discussions)을 살펴보면 실무자들이 실제로 마주치는 문제와 그 해결 방식을 폭넓게 접할 수 있습니다.

---

[◀ 이전: 18장. 실전 프로젝트: 설정 에디터 만들기](ch18-실전프로젝트설정에디터.md) | [📖 목차](00-목차.md)
