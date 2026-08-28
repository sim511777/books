# 프로그래밍 학습 자료실

이 사이트는 프로그래밍 언어 15종, 디자인 패턴, 자료구조와 알고리즘, UI 프레임워크, 개발 도구와
환경, 동시성과 병렬 프로그래밍, 컴퓨터 비전, 산업 자동화를 처음부터 새로 집필한 입문서/전문서
모음입니다. 각 책은 9~25개 장 + 부록으로 구성되어 있으며, 장마다 개념 설명·예제 코드·요약·
연습문제를 담고 있습니다.

## 프로그래밍 언어

| 언어 | 특징 | 바로가기 |
|---|---|---|
| C | 저수준 시스템 프로그래밍의 기본 | [C 기초부터 응용까지](Languages/C/C기초/00-목차.md) |
| C++ | C에 객체지향·템플릿·STL을 더한 언어 | [C++ 기초부터 응용까지](Languages/Cpp/Cpp기초/00-목차.md) |
| C# | .NET 기반 정적 타입 객체지향 언어 | [C# 기초부터 응용까지](Languages/CSharp/CSharp기초/00-목차.md) |
| Java | JVM 기반의 대표적인 객체지향 언어 | [Java 기초부터 응용까지](Languages/Java/Java기초/00-목차.md) |
| Python | 간결한 문법의 범용 인터프리터 언어 | [Python 기초부터 응용까지](Languages/Python/Python기초/00-목차.md) |
| Go | 단순함과 동시성을 강조하는 컴파일 언어 | [Go 기초부터 응용까지](Languages/Go/Go기초/00-목차.md) |
| Rust | 소유권 시스템으로 메모리 안전성을 보장하는 언어 | [Rust 기초부터 응용까지](Languages/Rust/Rust기초/00-목차.md) |
| Kotlin | JVM 기반의 null 안전 현대적 언어 | [Kotlin 기초부터 응용까지](Languages/Kotlin/Kotlin기초/00-목차.md) |
| Dart | sound null safety를 갖춘 클라이언트 최적화 언어 | [Dart 기초부터 응용까지](Languages/Dart/Dart기초/00-목차.md) |
| Clojure | JVM 기반의 Lisp 계열 함수형 언어 | [Clojure 기초부터 응용까지](Languages/Clojure/Clojure기초/00-목차.md) |
| Haskell | 순수 함수형·지연 평가 언어 | [Haskell 기초부터 응용까지](Languages/Haskell/Haskell기초/00-목차.md) |
| Lisp (Common Lisp) | 코드가 곧 데이터인 Lisp 계열의 원조 | [Lisp 기초부터 응용까지](Languages/Lisp/Lisp기초/00-목차.md) |
| Ruby | 블록·믹스인·메타프로그래밍이 자연스러운 동적 객체지향 언어 | [Ruby 기초부터 응용까지](Languages/Ruby/Ruby기초/00-목차.md) |
| Julia | 다중 디스패치와 JIT 컴파일로 과학계산에 최적화된 언어 | [Julia 기초부터 응용까지](Languages/Julia/Julia기초/00-목차.md) |
| Zig | comptime과 명시적 메모리 관리를 특징으로 하는 C 대체 시스템 언어 | [Zig 기초부터 응용까지](Languages/Zig/Zig기초/00-목차.md) |

## 소프트웨어 설계

- **[C#으로 배우는 디자인 패턴](DesignPatterns/디자인패턴/00-목차.md)** — GoF 23가지 디자인 패턴을 생성·구조·행동으로 나누어 다루며, 각 패턴이 .NET BCL 어디에 실제로 쓰이는지도 함께 소개합니다.

## 자료구조와 알고리즘

- **[자료구조와 알고리즘](DataStructuresAndAlgorithms/자료구조와알고리즘/00-목차.md)** — Big-O부터 배열, 연결 리스트, 트리, 힙, 해시 테이블, 그래프, 정렬·탐색, 재귀와 동적 프로그래밍까지. 모든 자료구조를 C#으로 직접 구현하고, 대응하는 .NET BCL 클래스도 함께 소개합니다.

## UI 프레임워크

- **[Avalonia로 배우는 크로스플랫폼 UI 개발](Avalonia/Avalonia기초/00-목차.md)** — C#/.NET 기반 크로스플랫폼 UI 프레임워크 Avalonia를 XAML 기초부터 레이아웃, 데이터 바인딩, MVVM 패턴, 스타일·템플릿, Windows/macOS/Linux 배포까지 다룹니다.
- **[Dear ImGui로 배우는 C++ GUI 프로그래밍](ImGui/ImGui기초/00-목차.md)** — C++ 즉시 모드(immediate mode) GUI 라이브러리 Dear ImGui를 핵심 개념부터 기본 위젯, 레이아웃, 테이블, 도킹, 커스텀 렌더링, 상태 관리 패턴, 실전 프로젝트까지 다룹니다.
- **[Slint로 배우는 C++ GUI 프로그래밍](Slint/Slint기초/00-목차.md)** — Rust 코어에 C++ 바인딩을 제공하는 선언적(declarative) GUI 툴킷 Slint를 `.slint` 언어 기초부터 프로퍼티 바인딩, C++ 연동, 표준 위젯, 상태/애니메이션, 리스트/모델, 임베디드 타겟팅, 실전 프로젝트까지 다룹니다.

## 개발 도구와 환경

- **[VS Code로 구축하는 C++ 개발환경](Languages/Cpp/CppVSCode개발환경/00-목차.md)** — GCC, VS Code, CMake, Ninja, vcpkg를 하나씩 설치하고 서로 연결하는 실전 강좌입니다. 빌드·디버깅·IntelliSense 설정부터 패키지 관리, 정적 분석, 테스트, 크로스플랫폼 배포까지 다룹니다.

## 동시성과 병렬 프로그래밍

- **[C#으로 배우는 동시성과 병렬 프로그래밍](Concurrency/동시성과병렬프로그래밍/00-목차.md)** — 스레드와 동기화 프리미티브 같은 저수준 기초부터 Task/async-await, Parallel/PLINQ, 동시성 컬렉션, 비동기 스트림과 Channel까지, C#의 멀티스레딩·비동기·병렬 프로그래밍을 처음부터 끝까지 다룹니다.

## 컴퓨터 비전

- **[Python으로 배우는 OpenCV 컴퓨터 비전](OpenCV/OpenCV로배우는컴퓨터비전/00-목차.md)** — NumPy 기반 이미지 표현부터 필터링, 엣지 검출, 컨투어 분석, 특징점 매칭, 얼굴/객체 검출, 실전 프로젝트와 성능 최적화까지 실무 수준으로 다룹니다.

## 산업 자동화

- **[C#/.NET 장비 제어 소프트웨어 개발 전문서](EquipmentControl/장비제어소프트웨어개발/00-목차.md)** — 반도체·디스플레이 등 산업 현장의 PC 기반 장비 제어 소프트웨어를 C#/.NET으로 개발하는 실무 엔지니어를 위한 전문서입니다. 레이어드 아키텍처, 비동기/멀티스레딩, 하드웨어 통신, 모션/I/O 제어, 상태 머신과 시퀀스, WPF/MVVM HMI, SECS/GEM과 머신비전 연동까지 다룹니다.

## 게임 프로그래밍

- **[raylib로 배우는 게임 프로그래밍](Raylib/Raylib기초/00-목차.md)** — C 언어 기반의 간단하고 빠른 게임/그래픽스 라이브러리 raylib로 게임 루프, 입력 처리, 텍스처/애니메이션, 충돌 감지, 오디오, 2D/3D 카메라, 셰이더, 게임 상태 관리, 파티클, 실전 미니 게임 제작까지 다룹니다.

---

각 책은 기존 출판된 교재를 번역하거나 참고한 것이 아니라, 처음부터 새로 집필한 오리지널 콘텐츠입니다.
