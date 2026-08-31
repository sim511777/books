# 부록

📖 [◀ 목차](00-목차.md) | [◀ 이전: 18장. 실전 프로젝트: 할 일 관리 앱 만들기](ch18-실전프로젝트할일관리앱만들기.md)

---

## A.1 설치와 빌드 환경 재정리

2장에서 다룬 Slint 개발 환경 구성을 간단히 다시 정리합니다. 세부 설명은 2장을 참고하세요.

Rust 개발 환경 자체는 `rustup`으로 준비합니다. `rustup`은 Rust 툴체인(컴파일러 `rustc`, 패키지 관리자 겸 빌드 도구 `cargo`)을 설치하고 버전을 관리해주는 도구입니다. 설치가 끝나면 `cargo new`로 새 프로젝트를 만들고, `slint` 크레이트를 의존성에 추가하는 것으로 Slint 사용을 시작할 수 있습니다.

```bash
cargo new my_app
cd my_app
cargo add slint
```

`.slint` 파일을 Rust 코드로 컴파일하는 작업은 빌드 스크립트(`build.rs`)가 담당하므로, `slint-build` 크레이트를 빌드 의존성(`build-dependencies`)으로 함께 추가해야 합니다.

```toml
[dependencies]
slint = "1"

[build-dependencies]
slint-build = "1"
```

UI 마크업만 빠르게 미리보고 싶을 때는, Rust 프로젝트 빌드 없이 `.slint` 파일 자체를 바로 열어볼 수 있는 `slint-viewer`라는 커맨드라인 도구도 있습니다. 이 도구는 언어 바인딩과 무관하게 동작하며, `cargo install slint-viewer`로 설치할 수 있습니다.

```bash
slint-viewer ui/app-window.slint
```

## A.2 주요 타입/매크로 요약표

이 책 전체에서 등장한 핵심 문법과 Rust API를 장별로 정리합니다. 각 항목의 자세한 설명은 해당 장을 참고하세요.

| 장 | 키워드/API | 설명 |
|---|---|---|
| 1장 | `slint` 크레이트 | Slint 런타임을 Rust 프로젝트에 통합하는 공식 크레이트입니다. |
| 2장 | `slint_build::compile("경로")` | `build.rs`에서 `.slint` 파일을 컴파일해 Rust 코드를 생성합니다. |
| 2장 | `slint::include_modules!()` | `build.rs`가 생성한 Rust 코드를 현재 크레이트에 끌어옵니다. |
| 3장 | `component 이름 inherits 부모 { ... }` | 컴포넌트를 선언하고 기존 엘리먼트나 컴포넌트를 상속받습니다. |
| 3장 | `property <타입> 이름: 기본값;` | 컴포넌트에 커스텀 프로퍼티를 선언합니다. |
| 3장 | `Rectangle`, `Text`, `Image` | 배경/컨테이너, 문자열 표시, 이미지 표시를 담당하는 기본 엘리먼트입니다. |
| 4장 | `VerticalLayout`, `HorizontalLayout`, `GridLayout` | 자식 엘리먼트를 세로, 가로, 격자로 배치하는 레이아웃 컨테이너입니다. |
| 5장 | `in`, `out`, `in-out` | 프로퍼티의 값이 어느 쪽 책임인지 나타내는 방향 키워드입니다. |
| 6장 | `callback 이름(인자타입): 반환타입;` | 컴포넌트가 발생시키는 이벤트를 선언합니다. |
| 7장 | `AppWindow::new()` | 생성된 컴포넌트 구조체의 인스턴스를 만듭니다. 실패 가능성이 있어 `Result<Self, slint::PlatformError>`를 반환합니다. |
| 7장 | `ComponentHandle` 트레이트 | 생성된 컴포넌트 구조체가 구현하는 트레이트로, `run()`, `show()`, `hide()`, `as_weak()`, `global::<T>()` 등을 제공합니다. |
| 7장 | `ui.run()` | 이벤트 루프를 시작해 창을 화면에 표시합니다. |
| 8장 | `set_*`/`get_*` | `.slint`의 프로퍼티를 Rust에서 읽고 쓰는 생성된 메서드입니다(케밥 케이스는 스네이크 케이스로 변환). |
| 8장 | `on_*`/`invoke_*` | `.slint`의 콜백을 Rust에서 처리하거나 직접 호출하는 생성된 메서드입니다. |
| 8장 | `slint::SharedString` | Slint의 `string` 타입에 대응하는 Rust 문자열 타입입니다. |
| 9장 | `Button`, `CheckBox`, `LineEdit`, `ListView` 등 | `std-widgets.slint`에서 제공하는 표준 위젯들입니다. |
| 10장 | `animate 프로퍼티 { duration: ...; }` | 프로퍼티 값이 바뀔 때 그 변화를 부드럽게 보간합니다. |
| 10장 | `states [ 이름 when 조건 : { ... } ]` | 조건에 따라 여러 프로퍼티 값을 한 번에 전환하는 상태 블록입니다. |
| 11장 | `[타입]` | 배열(모델) 타입을 나타내는 프로퍼티 타입 문법입니다. |
| 11장 | `for item[i] in 모델 : ...` | 모델의 각 항목에 대해 엘리먼트를 반복 생성합니다. |
| 11장 | `slint::VecModel<T>` | `Vec` 기반으로 구현된 Rust 쪽 모델 타입으로, `Model` 트레이트를 구현합니다. |
| 11장 | `slint::ModelRc<T>` | 서로 다른 구체 모델 타입을 하나의 공통 타입으로 감싸는 타입 지워진 래퍼입니다. |
| 11장 | `push`/`remove`/`row_data`/`set_row_data` | `VecModel`과 `Model` 트레이트가 제공하는, 항목을 추가·삭제·조회·갱신하는 메서드입니다. |
| 12장 | `global 이름 { ... }` | 여러 컴포넌트가 공유하는 전역 싱글톤을 선언합니다. |
| 12장 | `ui.global::<이름>()` | Rust에서 전역 싱글톤의 프로퍼티/콜백에 접근하는 방법입니다. |
| 13장 | `slint::Weak<T>` | `ComponentHandle`의 약한 참조로, `.upgrade()`나 `.unwrap()`으로 다시 강한 참조를 얻습니다. |
| 15장 | `slint::Timer`, `TimerMode` | 일정 시간 뒤 또는 반복적으로 콜백을 실행하는 타이머입니다. |
| 15장 | `slint::invoke_from_event_loop(...)` | 다른 스레드에서 UI 이벤트 루프 스레드로 클로저 실행을 예약합니다. |
| 15장 | `Weak::upgrade_in_event_loop(...)` | 다른 스레드에서 약한 참조를 이벤트 루프 스레드로 안전하게 넘겨 클로저를 실행합니다. |
| 16장 | `slint::Image` | Slint의 `image` 타입에 대응하는 Rust 이미지 타입입니다. |
| 17장 | FemtoVG, Skia | 데스크톱에서 사용되는 GPU 가속 렌더링 백엔드의 이름입니다. |
| 17장 | 소프트웨어 렌더러(`renderer-software`) | GPU 없이 CPU로 픽셀을 직접 그리는, 임베디드 환경을 위한 렌더링 백엔드입니다. |
| 18장 | `slint::CloseRequestedResponse` | `Window::on_close_requested` 콜백이 반환하는, 창을 닫을지 계속 보여줄지를 나타내는 열거형입니다. |

## A.3 이 저장소의 C++판 Slint 책과의 차이

이 저장소에는 같은 Slint를 C++로 다루는 "Slint 기초" 책이 함께 있습니다. 두 책은 같은 `.slint` 언어와 같은 렌더링 엔진을 다루지만, 애플리케이션 코드를 작성하는 언어가 다른 만큼 다음과 같은 차이가 있습니다.

| 구분 | C++판 | Rust판(이 책) |
|---|---|---|
| 빌드 시스템 | CMake (`find_package(Slint REQUIRED)`, `slint_target_sources(...)`) | Cargo (`build.rs` + `slint-build` 크레이트) |
| `.slint` → 코드 변환 | CMake 빌드 단계에서 헤더(`.h`) 생성 | `build.rs`에서 Rust 소스 생성 후 `slint::include_modules!()`로 포함 |
| 컴포넌트 인스턴스 생성 | `AppWindow::create()` (정적 팩토리 메서드) | `AppWindow::new() -> Result<Self, PlatformError>` |
| 컴포넌트 핸들 | `slint::ComponentHandle<T>` (내부적으로 `shared_ptr` 기반) | `T`가 직접 `ComponentHandle` 트레이트를 구현 |
| 참조 카운팅 | `std::shared_ptr`/`std::weak_ptr`를 명시적으로 다룸 | `Rc`/`slint::Weak<T>`를 다루며, `Weak::unwrap()`/`upgrade()`로 접근 |
| 모델 | `slint::VectorModel<T>` (`std::vector` 기반) | `slint::VecModel<T>` (`Vec` 기반) + `ModelRc<T>` |
| 콜백 등록 | `on_이름(람다)` | `on_이름(클로저)` — 문법은 유사하지만 클로저의 캡처 방식(값/참조/`move`)이 러스트의 소유권 규칙을 따릅니다 |
| 구조체 대응 | `.slint`의 `struct`가 같은 이름의 C++ `struct`로 생성 | `.slint`의 `struct`가 같은 이름의 Rust `struct`로 생성 |
| 메모리 관리 철학 | 프로그래머가 `shared_ptr`/`weak_ptr` 사용 여부와 캡처 방식을 직접 판단 | 컴파일러의 소유권·차용 검사기가 잘못된 캡처(예: 데이터 경합)를 상당 부분 컴파일 시점에 걸러줌 |
| 문자열 타입 | `slint::SharedString` | `slint::SharedString` (동일한 이름, 동일한 역할) |
| `.slint` 언어 자체 | 동일 | 동일 |

가장 근본적인 차이는 "같은 개념을 표현하는 언어 관용구가 다르다"는 데 있습니다. C++판에서는 `shared_ptr`로 감싼 모델을 람다가 캡처하면서 수명을 프로그래머가 스스로 추적해야 했다면, Rust판에서는 `Rc`와 `Weak`, 그리고 컴파일러의 소유권 검사가 그 추적의 상당 부분을 대신해줍니다. 반면 `.slint` 파일 자체의 문법(`component`, `property`, `callback`, `for`, `animate` 등)은 두 책 사이에 차이가 없습니다. 즉 이 언어를 한 번 익히면, 그 지식은 어떤 언어 바인딩을 쓰든 그대로 재사용됩니다.

## A.4 참고할 만한 공식 자료

더 깊이 있는 내용이나 이 책에서 다루지 않은 최신 기능을 확인하고 싶다면 다음 공식 자료를 참고하시기 바랍니다. 실제 주소는 검색 엔진에서 이름 그대로 검색하면 쉽게 찾을 수 있습니다.

- **Slint 공식 웹사이트**: 언어 문법, Rust/C++/JavaScript/Python 각 바인딩의 API 문서, 튜토리얼을 제공하는 공식 채널입니다.
- **docs.rs의 `slint` 크레이트 문서**: `slint`, `slint-build` 등 Rust 크레이트들의 타입과 함수 시그니처를 상세히 확인할 수 있는 레퍼런스입니다.
- **Slint GitHub 저장소**: 소스 코드, 이슈 트래커, 다양한 예제 프로젝트(Rust 예제와 임베디드 보드 예제 포함)를 확인할 수 있습니다.
- **slint-viewer**: A.1에서 소개한, `.slint` 파일을 Rust 빌드 없이 바로 열어볼 수 있는 커맨드라인 도구입니다.
- **Slint 릴리스 노트**: Slint는 활발히 개발되고 있는 프로젝트이므로, 새 버전이 나올 때마다 추가되거나 바뀌는 API를 확인하는 습관을 들이면 좋습니다.

## A.5 다음 학습 단계 제안

이 책 한 권으로 Slint와 Rust를 이용한 GUI 프로그래밍의 기초를 모두 다졌습니다. 이후에는 다음과 같은 방향으로 학습을 이어가 볼 것을 제안합니다.

- **더 큰 실전 앱 만들기**: 18장의 Todo 앱보다 화면 수가 많고, 여러 창과 전역 상태(12장)가 복잡하게 얽히는 애플리케이션을 직접 설계하고 만들어 보세요. 규모가 커질수록 프로퍼티 방향(5장)과 전역 싱글톤 설계, 그리고 `Rc`/`Weak`를 어디까지 사용할지에 대한 판단이 중요해집니다.
- **다른 언어 바인딩과 비교해보기**: Slint는 Rust 외에도 C++, JavaScript(Node.js), Python 바인딩을 공식 지원합니다. 이 저장소의 C++판 책과 비교해가며 같은 `.slint` UI가 서로 다른 언어의 애플리케이션 로직과 어떻게 연결되는지 살펴보면, "UI 정의와 애플리케이션 로직이 분리되어 있다"는 Slint의 설계가 언어 경계를 넘어서도 얼마나 일관되는지 직접 체감할 수 있습니다.
- **임베디드 타겟 실습**: 17장에서 개념만 소개한 임베디드/MCU 타겟팅을 실제 보드로 직접 실습해보세요. 크로스 컴파일 환경 구성부터 시작해야 하므로 별도의 학습 곡선이 있지만, 데스크톱에서 검증한 `.slint` UI가 실제 하드웨어의 작은 화면에 그대로 뜨는 경험은 이 책만으로는 얻기 어려운 감각을 줍니다.
- **비동기 생태계와의 통합 심화**: 15장에서 다룬 타이머와 스레드 연동을 더 깊이 파고들어, `tokio` 같은 비동기 런타임에서 얻은 결과를 `invoke_from_event_loop`나 `Weak::upgrade_in_event_loop`로 UI에 안전하게 반영하는 패턴을 직접 설계해보세요. 네트워크 요청이나 파일 입출력이 많은 실전 애플리케이션에서는 이 패턴이 특히 중요해집니다.

Slint는 여전히 활발히 발전하고 있는 툴킷입니다. 이 책에서 다진 기초 문법과 Rust 연동 패턴을 발판 삼아, 공식 문서와 예제 저장소를 직접 탐험하면서 여러분만의 애플리케이션을 만들어 나가시기 바랍니다.


---

📖 [◀ 목차](00-목차.md) | [◀ 이전: 18장. 실전 프로젝트: 할 일 관리 앱 만들기](ch18-실전프로젝트할일관리앱만들기.md)
