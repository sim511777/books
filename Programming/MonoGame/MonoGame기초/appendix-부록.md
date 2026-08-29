# 부록

[◀ 이전: 18장. 실전 프로젝트: 미니 게임 만들기](ch18-실전프로젝트미니게임만들기.md) | [📖 목차](00-목차.md)


## A. MonoGame 설치와 프로젝트 생성 요약

2장에서 다룬 개발 환경 설정과 첫 프로젝트 생성 과정을 빠르게 다시 확인할 수 있도록 핵심 명령만 간단히 정리합니다. 자세한 설명과 각 단계의 배경은 2장을 참고하세요.

**1) .NET SDK 설치 확인**

```bash
dotnet --version
```

버전 번호가 출력되면 .NET SDK가 정상적으로 설치되어 있는 것입니다. 너무 오래된 버전이라면 최신 LTS(Long-Term Support) 버전으로 업데이트해두는 것이 좋습니다.

**2) MonoGame 프로젝트 템플릿 설치**

```bash
dotnet new install MonoGame.Templates.CSharp
```

NuGet을 통해 MonoGame 공식 프로젝트 템플릿 모음을 내려받아 `dotnet new` 명령에 등록합니다. 한 번만 설치해두면 이후로는 어디서든 새 MonoGame 프로젝트를 즉시 생성할 수 있습니다. 설치된 템플릿 목록은 `dotnet new list`로 확인할 수 있으며, `mgdesktopgl`, `mgandroid`, `mgios` 등이 포함되어 있습니다.

**3) 새 프로젝트 생성**

```bash
dotnet new mgdesktopgl -o MyGame
```

`mgdesktopgl`은 Windows, macOS, Linux에서 공통으로 동작하는 OpenGL 기반 데스크톱 템플릿입니다. `-o MyGame` 옵션은 `MyGame`이라는 폴더를 만들고 그 안에 프로젝트 파일을 생성하라는 뜻입니다.

**4) 빌드 및 실행**

```bash
dotnet run
```

프로젝트 폴더(`.csproj` 파일이 있는 위치)에서 실행하면 빌드 후 곧바로 게임 창이 뜹니다.

**5) 배포용 산출물 만들기**

```bash
dotnet publish -c Release -r win-x64 --self-contained true
```

17장에서 다룬 대로, 배포할 때는 `dotnet run` 대신 `dotnet publish`에 런타임 식별자(RID)와 `--self-contained` 옵션을 지정합니다.

## B. 이 책에서 사용한 주요 클래스/API 요약

이 책 전체에서 등장한 MonoGame 및 관련 .NET 타입 중 특히 자주 쓰인 것들을 장별로 정리했습니다. 정확한 멤버 목록과 오버로드는 각 장의 본문이나 MonoGame 공식 문서를 참고하세요.

| 장 | 클래스/API | 설명 |
|---|---|---|
| 2, 3장 | `Game` | 게임 루프와 수명주기 메서드(`Initialize`, `LoadContent`, `Update`, `Draw`, `UnloadContent`)를 제공하는 기반 클래스 |
| 2, 3장 | `GraphicsDeviceManager` | 화면 해상도, 전체화면 여부 등 그래픽스 관련 설정을 관리한다 |
| 3, 5장 | `GraphicsDevice` | 화면 지우기(`Clear`), 뷰포트 조회 등 실제 그래픽 장치에 접근하는 통로 |
| 3장 | `GameTime` | `ElapsedGameTime`(프레임 간 경과 시간), `TotalGameTime`(누적 경과 시간)을 담은 시간 정보 |
| 4장 | `ContentManager` (`Game.Content`) | `Content.Load<T>("이름")` 형태로 MGCB가 빌드한 `.xnb` 에셋을 불러온다 |
| 5장 | `SpriteBatch` | 여러 2D 그리기 호출을 모아 효율적으로 GPU에 전달하는 클래스 |
| 4, 5장 | `Texture2D` | 이미지 데이터를 담는 텍스처 타입 |
| 5, 9장 | `Rectangle` | 사각형 영역을 표현하며, `Intersects`로 AABB 충돌 판정을 제공한다 |
| 5장 | `Color` | 색상 값을 표현하며, 곱셈으로 틴트나 알파 조절이 가능하다 |
| 5장 | `SpriteEffects` | `Draw` 시 텍스처를 좌우/상하로 반전시키는 열거형 |
| 6장 | `Keyboard`, `KeyboardState` | 키보드 입력을 폴링 방식으로 조회한다 |
| 6장 | `Mouse`, `MouseState` | 마우스 위치와 버튼 상태를 조회한다 |
| 6장 | `GamePad`, `GamePadState` | 게임패드 입력을 조회한다 |
| 7장 | `Vector2`, `Vector3` | 위치·속도·방향 등을 표현하는 값 타입 벡터 구조체 |
| 7장 | `MathHelper` | `Clamp`, `ToRadians`, `Lerp`, `TwoPi` 등 게임 수학에 자주 쓰이는 정적 헬퍼와 상수 |
| 8장 | `Rectangle`(소스 사각형) | 스프라이트시트에서 애니메이션 프레임 하나를 잘라내는 데 사용된다 |
| 9장 | `Rectangle.Intersects` | 두 사각형의 겹침 여부를 판정하는 AABB 충돌 검사 |
| 9장 | `Vector2.Distance` / `Vector2.DistanceSquared` | 두 점 사이의 거리를 계산해 원형 충돌 판정에 사용한다 |
| 10장 | `SpriteFont` | MGCB로 빌드한 폰트 에셋으로, 텍스트를 화면에 그릴 수 있게 한다 |
| 10장 | `SpriteBatch.DrawString` / `SpriteFont.MeasureString` | 문자열을 그리거나, 그렸을 때의 크기를 미리 계산한다 |
| 11장 | `SoundEffect` | 짧은 효과음을 즉시 재생하는 타입 |
| 11장 | `SoundEffectInstance` | 재생 상태를 세밀하게 제어(반복 재생, 볼륨 등)할 수 있는 효과음 인스턴스 |
| 11장 | `Song`, `MediaPlayer` | 배경음악을 스트리밍 방식으로 재생하는 타입과 이를 제어하는 정적 클래스 |
| 12장 | `Matrix` | 이동·회전·크기 변환을 표현하며, `SpriteBatch.Begin`에 전달해 카메라 좌표계를 구현한다 |
| 13장 | `BasicEffect` | 3D 정점을 렌더링할 때 사용하는 기본 셰이더 효과 |
| 13장 | `VertexPositionColor`, `VertexPositionTexture` | 3D 정점의 위치와 색상/텍스처 좌표를 담는 구조체 |
| 13장 | `GraphicsDevice.DrawUserPrimitives` | 정점 배열로부터 직접 3D 도형을 그리는 메서드 |
| 16장 | `System.Text.Json` / `System.IO.File` | 게임 상태를 JSON 등으로 직렬화해 파일로 저장하고 불러오는 데 사용하는 .NET 표준 API |
| 17장 | `dotnet publish` (CLI) | 런타임 식별자(RID)와 `--self-contained` 옵션으로 배포용 산출물을 생성하는 명령 |

## C. 참고할 만한 공식 자료

더 깊이 공부하고 싶다면 다음 공식 자료들을 활용하는 것을 권합니다.

- **MonoGame 공식 웹사이트**: 다운로드 링크, 튜토리얼, 뉴스, 커뮤니티 안내 등을 제공합니다.
- **MonoGame 공식 문서 사이트**: API 레퍼런스와 시작 가이드, 플랫폼별 배포 안내 등을 체계적으로 정리해두고 있습니다.
- **MonoGame GitHub 저장소**: 프레임워크 자체의 소스 코드가 공개되어 있으며, 이슈 트래커를 통해 버그를 확인하거나 최신 개발 동향을 살펴볼 수 있습니다. 저장소 안의 예제(Samples) 프로젝트들도 함께 살펴볼 만합니다.
- **MonoGame 커뮤니티(Discord 등)**: 공식 웹사이트에 안내된 커뮤니티 채널을 통해 다른 개발자들과 질문을 주고받거나, 쇼케이스를 구경할 수 있습니다. 초대 링크는 시점에 따라 바뀔 수 있으므로 공식 웹사이트에서 최신 링크를 확인하는 것을 권합니다.

## D. 다음 학습 단계

이 책을 마쳤다면, 다음과 같은 방향으로 학습을 이어가볼 수 있습니다.

- **더 큰 게임 프로젝트에 도전하기**: 18장에서 만든 벽돌깨기를 확장하거나, 플랫포머·탑다운 슈팅·퍼즐 게임처럼 새로운 장르를 처음부터 직접 설계해보세요. 규모가 커질수록 지금까지 배운 상태 관리, 콘텐츠 파이프라인, 충돌 감지가 왜 필요한지 더 몸으로 느끼게 됩니다.
- **MonoGame.Extended 같은 커뮤니티 확장 라이브러리 탐색하기**: MonoGame 코어는 의도적으로 최소한의 기능만 제공하는데, `MonoGame.Extended` 같은 커뮤니티 라이브러리는 여기에 타일맵 로딩, 카메라 헬퍼, UI 위젯, 트위닝(tweening), 입력 리스너 같은 편의 기능을 더해줍니다. 이 책에서 손수 구현했던 기능들(파티클, 카메라 변환 등)을 라이브러리가 어떻게 추상화하고 있는지 비교해보는 것도 좋은 공부가 됩니다.
- **ECS(Entity Component System) 아키텍처 학습하기**: 이 책의 예제들은 필드와 배열, 상태 열거형을 직접 다루는 비교적 단순한 구조를 사용했습니다. 게임의 개체 종류와 상호작용이 크게 늘어나면 ECS라는 설계 방식이 코드를 훨씬 유연하게 만들어줍니다. MonoGame과 함께 쓸 수 있는 ECS 라이브러리를 도입해보거나, 간단한 ECS를 직접 구현해보는 것도 좋은 다음 걸음이 됩니다.
- **플랫폼을 넓혀보기**: 17장에서 개념만 소개한 Android/iOS 배포를 실제로 진행해보거나, 13장에서 다룬 3D 기초를 발전시켜 간단한 3D 게임에 도전해보는 것도 자연스러운 다음 단계입니다.

---

[◀ 이전: 18장. 실전 프로젝트: 미니 게임 만들기](ch18-실전프로젝트미니게임만들기.md) | [📖 목차](00-목차.md)
