# 11장. 커스텀 렌더링과 ImDrawList

[◀ 이전: 10장. 도킹과 멀티 뷰포트](ch10-도킹과멀티뷰포트.md) | [📖 목차](00-목차.md) | [다음: 12장. 스타일링과 테마 ▶](ch12-스타일링과테마.md)


지금까지 살펴본 위젯들은 버튼, 슬라이더, 트리, 테이블처럼 이미 모양이 정해진 부품이었다. 이 부품들을 조합하면 웬만한 도구창은 충분히 만들 수 있지만, 게이지, 미니맵, 파형 그래프, 원형 다이얼처럼 정해진 모양이 없는 시각 요소를 만들어야 하는 순간이 반드시 온다. 이럴 때 ImGui는 위젯 뒤에 숨겨져 있던 저수준 드로잉 도구를 그대로 꺼내 쓸 수 있게 해준다. 그 도구가 바로 `ImDrawList`다.

이 장에서는 `ImDrawList`로 선, 사각형, 원, 텍스트를 직접 그리는 방법을 익히고, 이를 `ImGui::InvisibleButton()`과 결합해 완전히 새로운 모양의 커스텀 위젯을 만드는 패턴을 다룬다. 3장에서 살펴본 Immediate Mode의 핵심 — 매 프레임 다시 그린다는 개념 — 이 여기서 왜 커스텀 렌더링을 이렇게 쉽게 만들어 주는지도 자연스럽게 확인하게 될 것이다.

## 11.1 ImDrawList란 무엇인가

ImGui가 화면에 그리는 모든 것 — 창의 배경, 버튼의 사각형, 텍스트 글자 하나하나 — 은 결국 정점(vertex)과 인덱스로 이루어진 명령 목록으로 변환된 뒤 렌더링 백엔드(OpenGL, DirectX 등)에 전달된다. 이 명령 목록을 담는 자료구조가 `ImDrawList`이며, 위젯 함수들은 내부적으로 이 `ImDrawList`에 그리기 명령을 쌓는 방식으로 동작한다. 즉 우리가 `ImGui::Button()`을 호출할 때 ImGui는 내부에서 `AddRectFilled()`와 `AddText()` 같은 함수를 직접 호출하고 있는 셈이다. 이 장에서 배우는 것은 그 내부 구현에서 쓰이는 것과 동일한 도구다.

현재 창에 그리고 있는 draw list는 다음 함수로 얻는다.

```cpp
ImDrawList* draw_list = ImGui::GetWindowDrawList();
```

`GetWindowDrawList()`가 반환하는 draw list는 **현재 창의 콘텐츠 영역 안**에서만 보인다. 창 밖으로 벗어나는 좌표에 그려도 자동으로 잘려나간다(클리핑). 이 draw list에 그림을 추가하는 시점도 중요하다. ImGui는 draw list에 쌓인 명령을 호출된 순서대로 그리기 때문에, 어떤 위젯을 그리기 **전에** `AddRectFilled()`를 호출하면 그림이 위젯 아래에 깔리고, **후에** 호출하면 위젯 위에 덮인다. 이는 3장에서 다룬 "매 프레임 순서대로 명령을 쌓는다"는 Immediate Mode의 동작 방식과 정확히 같은 원리다.

참고로 특정 창에 속하지 않고 전체 화면의 맨 뒤 또는 맨 앞에 그리고 싶다면 `ImGui::GetBackgroundDrawList()`나 `ImGui::GetForegroundDrawList()`를 쓸 수 있다. 예를 들어 모든 창 위에 항상 보이는 디버그 오버레이를 그릴 때 유용하다. 이 장에서는 가장 자주 쓰이는 `GetWindowDrawList()`를 중심으로 다룬다.

## 11.2 좌표계 이해하기: 화면 좌표

`ImDrawList`의 모든 함수는 **화면 좌표(screen space)** 를 인자로 받는다. 화면 좌표는 창의 로컬 좌표가 아니라, OS 윈도우(백엔드가 만든 실제 창) 좌상단을 원점 `(0, 0)`으로 하는 절대 픽셀 좌표다. 이는 위젯 배치에 쓰는 `ImGui::SetCursorPos()` 같은 함수가 사용하는 "창 로컬 좌표"와는 다르다는 점을 반드시 기억해야 한다.

현재 커서 위치, 즉 다음 위젯이 그려질 위치를 화면 좌표로 얻으려면 다음 함수를 쓴다.

```cpp
ImVec2 p = ImGui::GetCursorScreenPos();
```

이와 이름이 비슷한 `ImGui::GetCursorPos()`는 창 로컬 좌표를 반환하므로 `ImDrawList` 함수에 그대로 넘기면 엉뚱한 위치에 그림이 그려진다. 커스텀 렌더링 코드에서 위치 계산이 이상하게 어긋난다면 십중팔구 이 둘을 혼동한 경우이니, 항상 `GetCursorScreenPos()`를 쓰는 습관을 들이자.

간단한 예로, 현재 커서 위치에서 오른쪽으로 100픽셀, 아래로 20픽셀 떨어진 지점에 선을 하나 그어보자.

```cpp
ImVec2 p0 = ImGui::GetCursorScreenPos();
ImVec2 p1 = ImVec2(p0.x + 100.0f, p0.y + 20.0f);

ImDrawList* draw_list = ImGui::GetWindowDrawList();
draw_list->AddLine(p0, p1, IM_COL32(255, 255, 0, 255), 2.0f);
```

여기서 등장한 `IM_COL32(r, g, b, a)`는 0~255 범위의 정수 네 개를 받아 `ImU32` 색상 값으로 변환해주는 매크로다. `ImDrawList`의 색상 인자는 5장이나 12장에서 다루는 `ImVec4`(0.0~1.0 실수 범위) 대신 대부분 이 `ImU32` 형식을 쓴다. `ImVec4` 색상을 갖고 있다면 `ImGui::ColorConvertFloat4ToU32()`로 변환할 수 있다.

## 11.3 기본 드로잉 함수

`ImDrawList`는 다양한 도형을 그리는 함수를 제공한다. 여기서는 가장 자주 쓰이는 네 가지를 살펴본다.

### 선: AddLine

```cpp
void AddLine(const ImVec2& p1, const ImVec2& p2, ImU32 col, float thickness = 1.0f);
```

두 점을 잇는 직선을 그린다.

### 사각형: AddRect / AddRectFilled

```cpp
void AddRect(const ImVec2& p_min, const ImVec2& p_max, ImU32 col,
             float rounding = 0.0f, ImDrawFlags flags = 0, float thickness = 1.0f);
void AddRectFilled(const ImVec2& p_min, const ImVec2& p_max, ImU32 col,
                    float rounding = 0.0f, ImDrawFlags flags = 0);
```

`p_min`은 좌상단, `p_max`는 우하단 좌표다. `AddRect()`는 테두리만, `AddRectFilled()`는 내부를 채운 사각형을 그린다. `rounding` 값을 0보다 크게 주면 모서리가 둥글게 처리되는데, 이는 12장에서 다루는 `WindowRounding`, `FrameRounding` 같은 스타일 값과 동일한 개념이다.

```cpp
ImDrawList* draw_list = ImGui::GetWindowDrawList();
ImVec2 p0 = ImGui::GetCursorScreenPos();
ImVec2 p1 = ImVec2(p0.x + 120.0f, p0.y + 60.0f);

draw_list->AddRectFilled(p0, p1, IM_COL32(60, 60, 200, 255), 8.0f);
draw_list->AddRect(p0, p1, IM_COL32(255, 255, 255, 255), 8.0f, 0, 1.5f);
```

### 원: AddCircle / AddCircleFilled

```cpp
void AddCircle(const ImVec2& center, float radius, ImU32 col,
               int num_segments = 0, float thickness = 1.0f);
void AddCircleFilled(const ImVec2& center, float radius, ImU32 col,
                      int num_segments = 0);
```

`num_segments`는 원을 구성하는 다각형의 변 개수다. `0`을 넘기면 ImGui가 반지름에 맞춰 적절한 값을 자동으로 계산해주므로, 특별한 이유가 없다면 `0`으로 두면 된다.

### 텍스트: AddText

```cpp
void AddText(const ImVec2& pos, ImU32 col, const char* text_begin, const char* text_end = NULL);
```

지정한 화면 좌표에 텍스트를 그린다. `ImGui::Text()`와 달리 커서 위치를 자동으로 이동시키지 않으므로, 다른 도형 위에 라벨을 겹쳐 그리는 용도로 자주 쓰인다.

```cpp
draw_list->AddText(ImVec2(p0.x + 4.0f, p0.y + 4.0f),
                    IM_COL32(255, 255, 255, 255), "Hello, ImDrawList!");
```

이 네 가지 함수만으로도 상당히 많은 커스텀 그래픽을 표현할 수 있다. `ImDrawList`에는 이 밖에도 `AddTriangleFilled()`, `AddPolyline()`, `AddBezierCubic()`, `AddConvexPolyFilled()` 같은 함수도 있지만, 대부분의 실무 위젯은 앞서 소개한 네 함수의 조합만으로 충분히 만들 수 있다.

## 11.4 실전 예제: 커스텀 프로그레스 바

ImGui는 기본 위젯으로 `ImGui::ProgressBar()`를 제공하지만, 모서리를 둥글게 하거나 안에 세그먼트 구분선을 넣는 등 디자인을 바꾸고 싶다면 직접 그리는 편이 빠르다. 아래는 배경, 채워진 진행 바, 가운데 퍼센트 텍스트로 구성된 커스텀 프로그레스 바다.

```cpp
void CustomProgressBar(const char* label, float fraction, const ImVec2& size)
{
    fraction = fraction < 0.0f ? 0.0f : (fraction > 1.0f ? 1.0f : fraction);

    ImVec2 p0 = ImGui::GetCursorScreenPos();
    ImVec2 p1 = ImVec2(p0.x + size.x, p0.y + size.y);

    ImDrawList* draw_list = ImGui::GetWindowDrawList();

    // 배경
    draw_list->AddRectFilled(p0, p1, IM_COL32(50, 50, 50, 255), size.y * 0.5f);

    // 채워진 부분
    if (fraction > 0.0f)
    {
        ImVec2 fill_max = ImVec2(p0.x + size.x * fraction, p1.y);
        draw_list->AddRectFilled(p0, fill_max, IM_COL32(80, 180, 100, 255), size.y * 0.5f);
    }

    // 테두리
    draw_list->AddRect(p0, p1, IM_COL32(20, 20, 20, 255), size.y * 0.5f);

    // 퍼센트 텍스트 (가운데 정렬)
    char buf[32];
    snprintf(buf, sizeof(buf), "%.0f%%", fraction * 100.0f);
    ImVec2 text_size = ImGui::CalcTextSize(buf);
    ImVec2 text_pos = ImVec2(p0.x + (size.x - text_size.x) * 0.5f,
                              p0.y + (size.y - text_size.y) * 0.5f);
    draw_list->AddText(text_pos, IM_COL32(255, 255, 255, 255), buf);

    // 레이아웃 공간 확보: 이 영역만큼 커서를 이동시켜서
    // 다음 위젯이 그림과 겹치지 않도록 한다.
    ImGui::Dummy(size);
}
```

여기서 `ImGui::Dummy(size)` 호출이 핵심이다. `ImDrawList`에 직접 그리는 것만으로는 레이아웃 커서가 전혀 움직이지 않는다. 5장에서 배운 대로 ImGui는 위젯을 그릴 때마다 커서를 자동으로 다음 줄로 내리는데, 커스텀 그리기는 이 과정을 거치지 않으므로 `Dummy()`로 직접 커서를 이동시켜 다음에 배치할 위젯이 겹치지 않게 해줘야 한다.

사용하는 쪽 코드는 다음과 같이 간단하다.

```cpp
static float progress = 0.35f;
ImGui::SliderFloat("진행률", &progress, 0.0f, 1.0f);
CustomProgressBar("build", progress, ImVec2(240.0f, 24.0f));
```

## 11.5 실전 예제: 간단한 미니맵

다음은 여러 개의 점(적, 아이템 등)과 플레이어 위치를 사각형 영역 안에 표시하는 미니맵 예제다. 월드 좌표를 미니맵 픽셀 좌표로 변환하는 계산이 핵심이다.

```cpp
struct MapEntity
{
    ImVec2 world_pos; // 월드 좌표 (0~world_size 범위)
    ImU32  color;
};

void DrawMinimap(const ImVec2& size, float world_size,
                  const ImVec2& player_pos, const std::vector<MapEntity>& entities)
{
    ImVec2 p0 = ImGui::GetCursorScreenPos();
    ImVec2 p1 = ImVec2(p0.x + size.x, p0.y + size.y);

    ImDrawList* draw_list = ImGui::GetWindowDrawList();

    // 미니맵 배경과 테두리
    draw_list->AddRectFilled(p0, p1, IM_COL32(20, 30, 20, 220));
    draw_list->AddRect(p0, p1, IM_COL32(180, 180, 180, 255));

    // 월드 좌표 -> 미니맵 픽셀 좌표 변환 함수
    auto to_minimap = [&](const ImVec2& world) {
        return ImVec2(p0.x + (world.x / world_size) * size.x,
                       p0.y + (world.y / world_size) * size.y);
    };

    // 엔티티들을 작은 점으로 표시
    for (const MapEntity& e : entities)
    {
        ImVec2 pos = to_minimap(e.world_pos);
        draw_list->AddCircleFilled(pos, 3.0f, e.color);
    }

    // 플레이어는 조금 더 크게, 다른 색으로 강조
    ImVec2 player_screen = to_minimap(player_pos);
    draw_list->AddCircleFilled(player_screen, 5.0f, IM_COL32(255, 220, 60, 255));

    ImGui::Dummy(size);
}
```

`AddCircleFilled()`를 반복 호출해 엔티티 목록을 점으로 찍는 방식은 7장에서 다룬 리스트 순회 패턴과 본질적으로 같다. 다른 점은 그 결과를 텍스트 대신 좌표 변환된 도형으로 그린다는 것뿐이다.

## 11.6 InvisibleButton 패턴: 클릭 가능한 커스텀 위젯

지금까지의 예제는 순수하게 "보여주기"만 했다. 하지만 커스텀 위젯도 마우스 클릭이나 호버에 반응해야 하는 경우가 많다. 문제는 `ImDrawList`로 그린 도형 자체는 입력을 전혀 처리하지 않는다는 점이다. 사각형을 그렸다고 해서 그 위에서 클릭 이벤트가 저절로 발생하지는 않는다.

이때 쓰는 표준 패턴이 `ImGui::InvisibleButton()`이다. 이 함수는 화면에는 아무것도 그리지 않지만, 지정한 크기만큼의 영역을 만들어 그 영역에 대한 호버·클릭 상태를 추적해준다. 즉 "입력 처리만 하는 투명한 버튼"을 만든 뒤, 그 위에 원하는 모양을 `ImDrawList`로 그리는 조합이 커스텀 위젯 제작의 기본 공식이다.

```cpp
bool InvisibleButton(const char* str_id, const ImVec2& size, ImGuiButtonFlags flags = 0);
```

이 패턴을 이용해 원형 토글 스위치를 만들어보자.

```cpp
bool ToggleSwitch(const char* str_id, bool* v)
{
    ImVec2 p = ImGui::GetCursorScreenPos();
    ImDrawList* draw_list = ImGui::GetWindowDrawList();

    float height = ImGui::GetFrameHeight();
    float width = height * 1.8f;
    float radius = height * 0.5f;

    // 1. 입력 처리 전담: 클릭 가능한 투명 영역 생성
    ImGui::InvisibleButton(str_id, ImVec2(width, height));
    bool changed = false;
    if (ImGui::IsItemClicked())
    {
        *v = !*v;
        changed = true;
    }

    // 2. 상태에 따른 시각 효과: 호버 중이면 더 밝게
    float t = *v ? 1.0f : 0.0f;
    ImU32 track_col = ImGui::IsItemHovered()
        ? IM_COL32(120, 120, 120, 255)
        : IM_COL32(90, 90, 90, 255);
    ImU32 on_col = IM_COL32(80, 180, 100, 255);
    ImU32 bg_col = *v ? on_col : track_col;

    // 3. 실제 그리기: InvisibleButton이 만든 영역 위에 그린다
    draw_list->AddRectFilled(p, ImVec2(p.x + width, p.y + height), bg_col, radius);

    ImVec2 knob_center = ImVec2(p.x + radius + t * (width - height), p.y + radius);
    draw_list->AddCircleFilled(knob_center, radius - 2.0f, IM_COL32(255, 255, 255, 255));

    return changed;
}
```

이 함수의 구조를 세 단계로 요약할 수 있다.

1. **입력 처리**: `InvisibleButton()`으로 클릭 가능한 사각 영역을 예약하고, `IsItemClicked()` / `IsItemHovered()` / `IsItemActive()`로 그 영역의 상태를 조회한다.
2. **상태 결정**: 클릭 여부나 호버 여부에 따라 색상, 위치 같은 시각적 파라미터를 계산한다.
3. **그리기**: `GetCursorScreenPos()`로 얻어둔 좌표를 기준으로 `ImDrawList` 함수를 호출해 원하는 모양을 그린다.

이 세 단계 패턴은 슬라이더, 노브, 컬러 스와치 등 어떤 형태의 커스텀 위젯을 만들든 거의 그대로 재사용된다. `InvisibleButton()`이 이미 커서 이동까지 처리해주므로, 이 패턴에서는 앞선 예제들과 달리 별도로 `Dummy()`를 호출할 필요가 없다는 점도 기억해두자.

## 요약

- `ImGui::GetWindowDrawList()`로 현재 창의 draw list를 얻어 저수준 도형을 직접 그릴 수 있다. 호출 순서가 곧 그려지는 순서(위아래 관계)를 결정한다.
- `ImDrawList` 함수는 모두 **화면 좌표**를 사용한다. 다음 위젯이 놓일 화면 좌표는 `ImGui::GetCursorScreenPos()`로 얻으며, 창 로컬 좌표를 반환하는 `GetCursorPos()`와 혼동하지 않도록 주의한다.
- `AddLine()`, `AddRect()`/`AddRectFilled()`, `AddCircle()`/`AddCircleFilled()`, `AddText()`가 가장 기본적이고 자주 쓰이는 드로잉 함수다. 색상은 보통 `IM_COL32()` 매크로로 만든 `ImU32` 값을 사용한다.
- 순수하게 그림만 그리면 레이아웃 커서가 움직이지 않으므로, `ImGui::Dummy(size)`로 직접 공간을 확보해야 다음 위젯과 겹치지 않는다.
- `ImGui::InvisibleButton()`으로 입력을 처리할 영역을 만들고 그 위에 `ImDrawList`로 그림을 그리는 조합은 커스텀 위젯을 만드는 표준 패턴이다.

## 연습문제

1. `AddRectFilled()`와 `AddLine()`을 사용해 일정 간격의 격자무늬(그리드)를 그리는 함수 `DrawGrid(ImVec2 size, float cell_size)`를 작성해보자.
2. 11.4절의 `CustomProgressBar()`를 수정하여, 진행률이 낮으면 빨간색, 중간이면 노란색, 높으면 초록색으로 채움 색이 바뀌도록 만들어보자.
3. `GetCursorScreenPos()` 대신 실수로 `GetCursorPos()`를 사용하면 어떤 문제가 발생하는지 직접 코드를 작성해 확인하고, 두 함수의 차이를 설명해보자.
4. 11.6절의 `ToggleSwitch()`를 참고하여, 클릭할 때마다 다음 색상으로 순환하는 원형 "색상 선택 버튼"을 `InvisibleButton()`과 `AddCircleFilled()`로 만들어보자.
5. 11.5절의 미니맵 예제에 `IsItemHovered()`와 `GetMousePos()`를 조합하여, 마우스가 미니맵 위에 있을 때 커서 아래에 해당하는 월드 좌표를 툴팁으로 표시하는 기능을 추가해보자.

---

[◀ 이전: 10장. 도킹과 멀티 뷰포트](ch10-도킹과멀티뷰포트.md) | [📖 목차](00-목차.md) | [다음: 12장. 스타일링과 테마 ▶](ch12-스타일링과테마.md)
