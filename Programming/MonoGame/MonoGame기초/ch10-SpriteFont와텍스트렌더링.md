# 10장. SpriteFont와 텍스트 렌더링

[◀ 이전: 9장. 충돌 감지](ch09-충돌감지.md) | [📖 목차](00-목차.md) | [다음: 11장. 오디오 ▶](ch11-오디오.md)


점수, 체력, 타이머, 대화창, 메뉴 항목 — 어떤 장르의 게임이든 화면에 글자를 띄워야 하는 순간이 반드시 찾아옵니다. 그런데 MonoGame을 처음 접하는 사람들이 은근히 자주 놀라는 지점이 하나 있습니다. "그냥 문자열 하나 화면에 찍는 게 왜 이렇게 번거롭지?"라는 질문입니다.

raylib 같은 일부 프레임워크는 실행 파일 안에 기본 폰트를 내장하고 있어서 `DrawText("Hello", 10, 10, 20, BLACK)` 한 줄이면 곧바로 글자가 나옵니다. 반면 MonoGame에는 그런 내장 폰트가 전혀 없습니다. 화면에 텍스트를 그리려면 반드시 폰트를 미리 이미지(텍스처 아틀라스)로 구워서 콘텐츠 파이프라인을 거쳐야 합니다. 4장에서 다룬 콘텐츠 파이프라인의 개념이 텍스트 렌더링에서 그대로, 그리고 가장 직접적으로 쓰이는 곳이 바로 이 장입니다.

번거로워 보이지만 이유가 있습니다. MonoGame은 특정 플랫폼의 텍스트 렌더링 API에 의존하지 않고, 텍스트를 처음부터 "글자 모양이 그려진 텍스처를 필요한 위치에 붙여 넣는" 방식으로 통일해서 처리합니다. 폰트 파일을 실행 중에 해석하는 대신 빌드 시점에 미리 문자별 이미지를 만들어 두기 때문에, 어떤 플랫폼에서도 동일하게 동작하고 실행 속도도 빠릅니다. 이번 장에서는 이 `SpriteFont` 시스템을 처음부터 끝까지 다뤄보겠습니다.

## 10.1 SpriteFont의 동작 원리

`SpriteFont`는 사실 특별한 것이 아닙니다. 내부적으로는 각 문자의 모양이 그려진 하나의 큰 텍스처(글리프 아틀라스)와, 그 텍스처 안에서 문자마다 어느 위치에 어떤 크기로 잘라야 하는지를 기록한 메타데이터의 조합입니다. `spriteBatch.DrawString`을 호출하면 내부적으로는 문자열의 각 글자에 해당하는 영역을 아틀라스에서 잘라내어 5장에서 배운 `SpriteBatch.Draw`를 문자 수만큼 반복 호출하는 것과 다르지 않습니다.

이 아틀라스와 메타데이터는 실행 중에 만들어지는 것이 아니라, 콘텐츠 파이프라인이 빌드 시점에 시스템에 설치된 트루타입(TrueType) 폰트로부터 미리 생성해 둡니다. 즉 게임을 실행하는 사용자의 컴퓨터에 해당 폰트가 설치되어 있지 않아도 전혀 문제가 없습니다. 필요한 문자 이미지는 이미 게임 콘텐츠 안에 구워져 있기 때문입니다.

## 10.2 .spritefont 파일 만들기

MGCB Editor(4장에서 콘텐츠 파이프라인을 설정할 때 사용했던 바로 그 도구)를 열어 프로젝트에 새 항목을 추가합니다.

1. MGCB Editor에서 콘텐츠를 추가할 폴더를 마우스 오른쪽 버튼으로 클릭하고 **Add → New Item...**을 선택합니다.
2. 항목 목록에서 **SpriteFont Description**을 선택하고 파일 이름을 지정합니다(예: `DefaultFont.spritefont`).
3. 프로젝트를 빌드(Build)하면 이 XML 설명 파일을 바탕으로 폰트 텍스처가 생성됩니다.

`.spritefont` 파일은 사실 평범한 XML 문서입니다. 기본 생성된 내용은 대략 다음과 같은 형태를 띱니다.

```xml
<?xml version="1.0" encoding="utf-8"?>
<XnaContent xmlns:Graphics="Microsoft.Xna.Framework.Content.Pipeline.Graphics">
  <Asset Type="Graphics:FontDescription">
    <FontName>Arial</FontName>
    <Size>24</Size>
    <Spacing>0</Spacing>
    <UseKerning>true</UseKerning>
    <Style>Regular</Style>

    <CharacterRegions>
      <CharacterRegion>
        <Start>&#32;</Start>
        <End>&#126;</End>
      </CharacterRegion>
    </CharacterRegions>
  </Asset>
</XnaContent>
```

각 항목의 의미는 다음과 같습니다.

- `FontName`: 빌드 머신에 설치되어 있어야 하는 시스템 폰트 이름입니다. "Arial", "맑은 고딕"처럼 운영체제에 설치된 폰트 이름을 그대로 적습니다.
- `Size`: 폰트 크기(포인트 단위)입니다. 이 크기로 문자 텍스처가 구워지므로, 게임에서 이보다 훨씬 크게 확대해서 그리면 이미지가 흐려질 수 있습니다.
- `Spacing`: 문자 사이의 추가 간격(픽셀)입니다.
- `UseKerning`: 커닝(문자 쌍에 따른 미세 간격 조정) 적용 여부입니다.
- `Style`: `Regular`, `Bold`, `Italic`, `Bold, Italic`을 지정할 수 있습니다.
- `CharacterRegions`: 실제로 텍스처에 구워 넣을 문자의 유니코드 범위입니다. `&#32;`(공백)부터 `&#126;`(물결표 `~`)까지가 기본값으로, 이는 ASCII 인쇄 가능 문자 전체를 포함합니다.

여기서 중요한 점은, `CharacterRegions`에 포함되지 않은 문자를 `DrawString`으로 그리려고 하면 빌드된 폰트에 해당 글리프가 존재하지 않아 예외가 발생한다는 것입니다. 기본값은 영문과 숫자, 기본 기호만 포함하므로 한글을 그리려면 반드시 범위를 추가해야 합니다. 이 내용은 10.6절에서 자세히 다룹니다.

## 10.3 폰트 로드하기

`.spritefont` 파일을 빌드하고 나면, 다른 콘텐츠와 마찬가지로 `Content.Load<T>`로 불러올 수 있습니다. 이때 타입 매개변수는 `SpriteFont`입니다.

```csharp
public class Game1 : Game
{
    private GraphicsDeviceManager graphics;
    private SpriteBatch spriteBatch;
    private SpriteFont font;

    public Game1()
    {
        graphics = new GraphicsDeviceManager(this);
        Content.RootDirectory = "Content";
    }

    protected override void LoadContent()
    {
        spriteBatch = new SpriteBatch(GraphicsDevice);
        font = Content.Load<SpriteFont>("DefaultFont");
    }
}
```

`Content.Load<SpriteFont>`에 넘기는 문자열은 확장자를 뺀 에셋 이름입니다. MGCB에서 지정한 출력 경로(폴더 구조를 만들었다면 `"Fonts/DefaultFont"`처럼 상대 경로 포함)와 정확히 일치해야 합니다. 이는 텍스처나 사운드를 로드할 때와 동일한 규칙으로, 4장에서 다룬 콘텐츠 파이프라인의 경로 규칙을 그대로 따릅니다.

## 10.4 DrawString으로 텍스트 그리기

폰트를 로드했다면 `SpriteBatch.DrawString`으로 화면에 텍스트를 그릴 수 있습니다. 가장 단순한 오버로드는 다음과 같은 시그니처를 가집니다.

```csharp
public void DrawString(SpriteFont spriteFont, string text, Vector2 position, Color color);
```

`Draw`와 마찬가지로 `Begin`과 `End` 사이에서 호출해야 합니다.

```csharp
protected override void Draw(GameTime gameTime)
{
    GraphicsDevice.Clear(Color.CornflowerBlue);

    spriteBatch.Begin();
    spriteBatch.DrawString(font, "Score: 1200", new Vector2(20, 20), Color.White);
    spriteBatch.End();

    base.Draw(gameTime);
}
```

`DrawString`에는 5장에서 다룬 `Draw`의 확장 오버로드와 대칭되는, 회전과 크기 조절, 원점, 그리기 깊이까지 지정할 수 있는 버전도 있습니다.

```csharp
public void DrawString(
    SpriteFont spriteFont,
    string text,
    Vector2 position,
    Color color,
    float rotation,
    Vector2 origin,
    float scale,
    SpriteEffects effects,
    float layerDepth);
```

예를 들어 텍스트를 서서히 커지게 하는 연출은 `scale` 매개변수를 매 프레임 조금씩 늘리는 것만으로 구현할 수 있습니다.

```csharp
spriteBatch.DrawString(
    font,
    "GAME OVER",
    position,
    Color.Red,
    0f,
    Vector2.Zero,
    pulseScale,
    SpriteEffects.None,
    0f);
```

## 10.5 MeasureString으로 텍스트 크기 재기

텍스트를 화면 중앙에 배치하거나, 여러 줄의 텍스트를 정렬하거나, 텍스트 배경 상자의 크기를 텍스트 길이에 맞춰 자동으로 조절하고 싶을 때가 있습니다. 이럴 때 텍스트를 실제로 그리기 전에 그 크기를 미리 알아야 합니다. `SpriteFont.MeasureString`이 이 역할을 합니다.

```csharp
public Vector2 MeasureString(string text);
```

반환되는 `Vector2`의 `X`는 텍스트의 너비(픽셀), `Y`는 높이(픽셀)입니다. 이 값을 이용하면 화면 중앙 정렬을 다음과 같이 구현할 수 있습니다.

```csharp
private void DrawCenteredText(SpriteBatch spriteBatch, SpriteFont font, string text, Color color)
{
    Vector2 textSize = font.MeasureString(text);

    int screenWidth = GraphicsDevice.Viewport.Width;
    int screenHeight = GraphicsDevice.Viewport.Height;

    Vector2 position = new Vector2(
        (screenWidth - textSize.X) / 2f,
        (screenHeight - textSize.Y) / 2f);

    spriteBatch.DrawString(font, text, position, color);
}
```

같은 원리를 응용하면 버튼 텍스트를 버튼 사각형 안에서 중앙 정렬하거나, 텍스트 뒤에 딱 맞는 크기의 반투명 배경 상자를 그리는 것도 가능합니다.

```csharp
private void DrawTextWithBackground(SpriteBatch spriteBatch, Texture2D pixel,
    SpriteFont font, string text, Vector2 position, Color textColor, Color backColor)
{
    Vector2 size = font.MeasureString(text);
    int padding = 8;

    Rectangle backRect = new Rectangle(
        (int)position.X - padding,
        (int)position.Y - padding,
        (int)size.X + padding * 2,
        (int)size.Y + padding * 2);

    spriteBatch.Draw(pixel, backRect, backColor);
    spriteBatch.DrawString(font, text, position, textColor);
}
```

여기서 `pixel`은 1x1 크기의 흰색 텍스처로, 색을 입혀 임의 크기의 사각형을 그리는 용도로 흔히 쓰이는 기법입니다. 점수나 타이머처럼 매 프레임 문자열 내용이 바뀌는 텍스트는 `MeasureString`도 매번 다시 호출해야 정확한 크기를 얻을 수 있다는 점에 유의하세요. 텍스트 내용이 고정되어 있다면 굳이 매 프레임 다시 잴 필요 없이 한 번 계산한 크기를 캐시해 두는 편이 효율적입니다.

## 10.6 실무 팁: 한글 폰트 지원하기

기본으로 생성되는 `.spritefont`의 `CharacterRegions`는 ASCII 범위(`&#32;`부터 `&#126;`까지)만 포함하기 때문에, 이 상태로 한글 문자열을 `DrawString`에 넘기면 빌드된 폰트에 해당 글리프가 없어 런타임에 예외가 발생합니다. 한글을 지원하려면 `CharacterRegions`에 한글 유니코드 범위를 직접 추가해야 합니다.

현대 한글(완성형 한글 음절, 가~힣)의 유니코드 범위는 `U+AC00`부터 `U+D7A3`까지입니다. `.spritefont` XML에 다음과 같이 범위를 추가합니다.

```xml
<CharacterRegions>
  <CharacterRegion>
    <Start>&#32;</Start>
    <End>&#126;</End>
  </CharacterRegion>
  <CharacterRegion>
    <Start>&#44032;</Start>  <!-- U+AC00, '가' -->
    <End>&#55203;</End>      <!-- U+D7A3, '힣' -->
  </CharacterRegion>
</CharacterRegions>
```

XML의 문자 참조는 10진수를 사용하므로 `U+AC00`은 44032, `U+D7A3`은 55203으로 환산해서 적어야 합니다. 필요하다면 자음·모음 낱자(예: 반각 자모, `U+3131`~`U+318E`)나 특수 문장 부호 범위를 추가로 넣을 수도 있습니다.

여기서 반드시 기억해야 할 실무상의 함정이 하나 있습니다. 한글 음절 전체 범위(약 11,172자)를 통째로 포함시키면 빌드되는 텍스처 아틀라스가 매우 커지고, 빌드 시간도 눈에 띄게 길어집니다. 실제 게임에서 사용할 문자가 제한적이라면(예: 특정 대사 스크립트에 등장하는 글자만), 전체 범위 대신 실제로 쓰이는 문자만 개별 `CharacterRegion`으로 좁혀 넣는 것이 빌드 성능과 최종 텍스처 메모리 사용량 양쪽에 유리합니다. 반대로 사용자가 자유롭게 입력하는 텍스트(채팅, 닉네임 등)를 그려야 한다면 전체 범위를 포함하는 것이 안전합니다.

또한 `FontName`에 지정하는 폰트가 한글 글리프를 실제로 지원하는지도 확인해야 합니다. "Arial"처럼 한글이 없는 서양 폰트를 지정한 채로 한글 유니코드 범위만 추가하면, 콘텐츠 빌드 과정에서 해당 문자를 찾지 못해 오류가 발생하거나 의도치 않은 대체 글리프(네모 문자 등)가 구워질 수 있습니다. "맑은 고딕", "나눔고딕"처럼 한글을 지원하는 폰트를 `FontName`에 지정해야 합니다.

## 요약

- MonoGame에는 내장 기본 폰트가 없으며, 텍스트를 그리려면 반드시 콘텐츠 파이프라인을 통해 `.spritefont`를 빌드해 `SpriteFont` 에셋으로 만들어야 합니다.
- MGCB Editor에서 **Add → New Item → SpriteFont Description**으로 `.spritefont` 파일을 추가하며, 이 파일은 폰트 이름, 크기, 스타일, 문자 범위를 지정하는 XML 문서입니다.
- `Content.Load<SpriteFont>("이름")`으로 폰트를 로드하고, `spriteBatch.Begin()`과 `End()` 사이에서 `DrawString(font, text, position, color)`로 텍스트를 그립니다.
- `font.MeasureString(text)`는 텍스트가 차지할 너비와 높이를 `Vector2`로 반환하며, 이를 이용해 중앙 정렬이나 텍스트에 꼭 맞는 배경 상자 같은 레이아웃을 구현할 수 있습니다.
- 한글을 그리려면 `.spritefont`의 `CharacterRegions`에 한글 유니코드 범위(`U+AC00`~`U+D7A3` 등, 10진수로 44032~55203)를 추가해야 하며, 한글을 지원하는 `FontName`을 지정해야 합니다. 전체 범위를 포함하면 텍스처와 빌드 시간이 크게 늘어나므로 실제 사용 문자에 맞춰 범위를 좁히는 것이 실무에서 유리합니다.

## 연습문제

1. 새 `.spritefont` 파일을 만들어 MGCB Editor로 빌드하고, `DrawString`으로 "Hello, MonoGame!"이라는 문자열을 화면 좌상단에 출력해 보세요.
2. `MeasureString`을 이용해 임의의 문자열을 화면 정중앙에 정렬해서 그리는 `DrawCenteredText` 메서드를 작성하고, 문자열 내용을 바꿔가며 항상 중앙에 위치하는지 확인해 보세요.
3. `.spritefont`의 `CharacterRegions`에 한글 범위를 추가하지 않은 채로 한글 문자열을 `DrawString`에 전달했을 때 어떤 예외가 발생하는지 직접 확인하고, 범위를 추가한 뒤 다시 실행해 정상적으로 그려지는지 비교해 보세요.
4. 9장에서 만든 충돌 감지 예제를 확장하여, 두 오브젝트가 충돌했을 때 화면에 "Collision!" 같은 텍스트를 일정 시간 동안 표시하는 기능을 추가해 보세요.
5. 점수를 나타내는 정수 변수를 매 프레임 `DrawString`으로 출력하되, `MeasureString`을 이용해 자릿수가 늘어나도(예: 9에서 10으로) 텍스트가 항상 화면 오른쪽 끝에 붙어 오른쪽 정렬되도록 구현해 보세요.

---

[◀ 이전: 9장. 충돌 감지](ch09-충돌감지.md) | [📖 목차](00-목차.md) | [다음: 11장. 오디오 ▶](ch11-오디오.md)
