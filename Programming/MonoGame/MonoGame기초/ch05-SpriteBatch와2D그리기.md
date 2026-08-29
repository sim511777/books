# 5장. SpriteBatch와 2D 그리기

[◀ 이전: 4장. 콘텐츠 파이프라인(Content Pipeline)](ch04-콘텐츠파이프라인.md) | [📖 목차](00-목차.md) | [다음: 6장. 입력 처리 ▶](ch06-입력처리.md)


지금까지 우리는 `Game` 클래스의 뼈대를 세우고, `LoadContent()`에서 콘텐츠 파이프라인을 통해 텍스처를 불러오는 방법까지 살펴봤습니다. 이제 그 텍스처를 실제로 화면에 그려볼 차례입니다. MonoGame에서 2D 그리기의 중심에는 `SpriteBatch`라는 클래스가 있습니다. 이번 장에서는 `SpriteBatch`가 왜 필요한지, 어떻게 사용하는지, 그리고 `Draw()` 메서드가 제공하는 다양한 오버로드를 하나씩 뜯어보겠습니다.

## SpriteBatch란 무엇인가

게임 화면에는 보통 수십, 수백 개의 스프라이트가 동시에 그려집니다. 배경, 캐릭터, 적, 총알, UI 아이콘까지 모두 합치면 한 프레임에 그려야 할 이미지가 상당히 많아집니다. 만약 이런 이미지를 하나하나 그릴 때마다 GPU에 개별적으로 그리기 명령을 보낸다면, 그리기 호출(draw call) 자체의 오버헤드 때문에 성능이 크게 떨어질 수 있습니다.

`SpriteBatch`는 이 문제를 해결하기 위한 클래스입니다. 이름 그대로 "스프라이트를 묶어서(batch) 처리"합니다. 여러 개의 `Draw()` 호출을 즉시 GPU로 보내지 않고 내부 버퍼에 쌓아두었다가, 가능한 경우 같은 텍스처를 사용하는 그리기 요청들을 하나의 큰 그리기 호출로 묶어서 GPU에 전달합니다. 개발자 입장에서는 그저 그리고 싶은 것마다 `Draw()`를 호출하면 되고, 배치 처리에 대한 세부 사항은 `SpriteBatch`가 알아서 처리해줍니다.

`SpriteBatch`는 사용하기 전에 생성해야 합니다. 보통 `Game` 클래스의 `LoadContent()`에서 `GraphicsDevice`를 넘겨받아 한 번만 생성해두고 게임 전체에서 재사용합니다.

```csharp
protected override void LoadContent()
{
    _spriteBatch = new SpriteBatch(GraphicsDevice);
    _playerTexture = Content.Load<Texture2D>("player");
}
```

`SpriteBatch`의 생성자는 `GraphicsDevice` 하나만 받습니다. `GraphicsDevice`는 3장에서 다뤘듯이 `LoadContent()` 시점에는 이미 초기화가 끝나 있으므로 안전하게 접근할 수 있습니다.

## Begin과 End 사이에 그리기

`SpriteBatch`를 사용하는 가장 중요한 규칙은 다음과 같습니다.

> 모든 `Draw()` 호출은 반드시 `spriteBatch.Begin()`과 `spriteBatch.End()` 사이에 있어야 한다.

`Begin()`은 새로운 배치 작업을 시작한다고 GPU 파이프라인에 알리는 역할을 하고, `End()`는 지금까지 쌓인 그리기 요청들을 실제로 GPU에 전송(flush)하는 역할을 합니다. `Begin()` 없이 `Draw()`를 호출하거나, `End()`를 호출하지 않고 다음 프레임으로 넘어가면 `InvalidOperationException`이 발생하거나 화면에 아무것도 그려지지 않는 문제가 생깁니다.

이 패턴은 항상 `Game` 클래스의 `Draw()` 메서드 안에서 사용합니다. 가장 기본적인 형태는 다음과 같습니다.

```csharp
protected override void Draw(GameTime gameTime)
{
    GraphicsDevice.Clear(Color.CornflowerBlue);

    _spriteBatch.Begin();

    _spriteBatch.Draw(_playerTexture, _playerPosition, Color.White);

    _spriteBatch.End();

    base.Draw(gameTime);
}
```

순서를 정리하면 다음과 같습니다.

1. `GraphicsDevice.Clear()`로 화면을 한 가지 색으로 지운다.
2. `spriteBatch.Begin()`으로 배치를 시작한다.
3. 그리고 싶은 것을 모두 `spriteBatch.Draw()`로 그린다.
4. `spriteBatch.End()`로 배치를 끝내고 실제로 GPU에 전송한다.

`Begin()`과 `End()`는 한 프레임에 여러 번 짝지어 호출할 수도 있습니다. 예를 들어 블렌드 모드나 정렬 방식을 바꿔서 그려야 하는 그룹이 있다면 `End()`로 한 번 마무리하고 다른 옵션으로 `Begin()`을 다시 호출하면 됩니다. 다만 `Begin()`을 호출한 상태에서 또 `Begin()`을 호출하면 예외가 발생하므로, 반드시 짝을 맞춰야 합니다.

> **참고**: `Begin()`에는 정렬 모드(`SpriteSortMode`), 블렌드 상태(`BlendState`), 샘플러 상태(`SamplerState`), 변환 행렬(`Matrix`) 등을 지정하는 오버로드가 있습니다. 변환 행렬을 이용한 카메라 구현은 12장에서 자세히 다루고, 여기서는 인자 없는 기본 `Begin()`만 사용합니다.

## Draw() 메서드의 다양한 오버로드

`SpriteBatch.Draw()`는 상황에 따라 필요한 정보만 넘길 수 있도록 여러 오버로드를 제공합니다. 가장 단순한 형태부터 가장 복잡한 형태까지 순서대로 살펴보겠습니다.

### 1. 위치만 지정하기

가장 단순한 오버로드는 텍스처, 위치, 색상만 받습니다.

```csharp
public void Draw(Texture2D texture, Vector2 position, Color color);
```

`position`은 텍스처의 왼쪽 위 모서리가 놓일 화면 좌표입니다. `color`에 `Color.White`를 넘기면 텍스처 원본의 색을 그대로 그립니다.

```csharp
_spriteBatch.Draw(_playerTexture, new Vector2(100, 200), Color.White);
```

색상에 흰색이 아닌 다른 값을 넘기면 텍스처의 각 픽셀 색상에 곱셈으로 틴트(tint)가 적용됩니다. 예를 들어 `Color.Red`를 넘기면 텍스처가 붉은빛으로 물든 것처럼 보이고, 반투명하게 그리고 싶다면 `Color.White * 0.5f`처럼 알파 값을 조절한 색을 넘기면 됩니다.

### 2. 사각형으로 위치와 크기를 함께 지정하기

`position` 대신 `Rectangle`을 넘기는 오버로드도 있습니다. 이 경우 텍스처는 원본 크기와 무관하게 지정한 사각형 크기에 맞춰 늘어나거나 줄어들어 그려집니다.

```csharp
public void Draw(Texture2D texture, Rectangle destinationRectangle, Color color);
```

```csharp
_spriteBatch.Draw(_playerTexture, new Rectangle(100, 200, 64, 64), Color.White);
```

### 3. 소스 사각형으로 텍스처 일부만 그리기

텍스처 전체가 아니라 일부분만 그리고 싶을 때는 `sourceRectangle`을 지정하는 오버로드를 사용합니다.

```csharp
public void Draw(Texture2D texture, Vector2 position, Rectangle? sourceRectangle, Color color);
```

`sourceRectangle`은 텍스처 이미지 안에서 잘라낼 영역을 픽셀 좌표로 지정합니다. `null`을 넘기면 텍스처 전체를 사용합니다.

```csharp
// 텍스처의 (0, 0) 위치에서 32x32 크기만큼만 잘라서 그린다.
var sourceRect = new Rectangle(0, 0, 32, 32);
_spriteBatch.Draw(_spriteSheet, new Vector2(100, 200), sourceRect, Color.White);
```

이 기능은 하나의 큰 이미지(스프라이트 시트) 안에 여러 프레임의 애니메이션이나 여러 종류의 타일을 모아두고, 매번 필요한 부분만 잘라서 그리는 데 사용됩니다. 8장에서 스프라이트 시트를 이용한 애니메이션을 다룰 때 이 `sourceRectangle`이 핵심적인 역할을 하게 되니 기억해두시기 바랍니다.

### 4. 회전, 원점, 스케일까지 포함한 전체 오버로드

가장 많은 정보를 받는 오버로드는 다음과 같습니다.

```csharp
public void Draw(
    Texture2D texture,
    Vector2 position,
    Rectangle? sourceRectangle,
    Color color,
    float rotation,
    Vector2 origin,
    float scale,
    SpriteEffects effects,
    float layerDepth);
```

각 매개변수의 의미는 다음과 같습니다.

- **`rotation`**: 회전 각도를 라디안 단위로 지정합니다. 도(degree) 단위 값이 있다면 `MathHelper.ToRadians()`로 변환해서 넘겨야 합니다. 양수 값은 시계 방향으로 회전합니다.
- **`origin`**: 회전과 스케일의 기준점이자, `position`이 가리키는 지점이기도 합니다. 텍스처 내부 좌표계 기준으로 지정합니다. 예를 들어 32x32 텍스처의 정중앙을 기준으로 회전시키고 싶다면 `origin`을 `(16, 16)`으로 지정합니다. 기본값인 `Vector2.Zero`(왼쪽 위 모서리)를 기준으로 회전시키면 텍스처가 마치 시계추처럼 모서리를 축으로 빙글빙글 도는 것처럼 보이므로, 자연스러운 회전을 원한다면 중심점을 원점으로 지정하는 것이 일반적입니다.
- **`scale`**: 텍스처를 확대/축소하는 배율입니다. `1.0f`이면 원본 크기 그대로이고, `2.0f`이면 두 배 크기로 그립니다. `float` 하나로 가로세로를 동일하게 조절하는 오버로드와, `Vector2`로 가로세로를 각각 다르게 조절하는 오버로드가 모두 존재합니다.
- **`effects`**: `SpriteEffects.None`, `SpriteEffects.FlipHorizontally`, `SpriteEffects.FlipVertically` 값으로 텍스처를 좌우 또는 상하로 뒤집어서 그릴 수 있습니다. 캐릭터가 왼쪽을 볼 때와 오른쪽을 볼 때 텍스처를 따로 만들지 않고 하나의 텍스처를 좌우 반전해서 재사용할 때 유용합니다.
- **`layerDepth`**: 0.0f에서 1.0f 사이의 값으로, 여러 스프라이트가 겹칠 때 그리는 순서(깊이)를 지정합니다. `Begin()`에 `SpriteSortMode.BackToFront`나 `FrontToBack`을 지정했을 때 의미를 가지며, 기본 정렬 모드에서는 호출 순서대로 그려집니다.

다음은 텍스처를 화면 중앙 근처에 두 배 크기로, 45도 회전시켜 그리는 예제입니다.

```csharp
Vector2 origin = new Vector2(_playerTexture.Width / 2f, _playerTexture.Height / 2f);
float rotation = MathHelper.ToRadians(45f);

_spriteBatch.Draw(
    _playerTexture,
    _playerPosition,
    null,               // 텍스처 전체 사용
    Color.White,
    rotation,
    origin,
    2.0f,               // 두 배 확대
    SpriteEffects.None,
    0f);
```

이 오버로드는 인자가 많아서 처음에는 부담스러울 수 있지만, 실제로는 필요한 값만 채우고 나머지는 관습적인 기본값(`sourceRectangle`은 `null`, `rotation`은 `0f`, `origin`은 `Vector2.Zero`, `scale`은 `1f`, `effects`는 `SpriteEffects.None`, `layerDepth`는 `0f`)을 넣으면 되므로, 몇 번 사용해보면 금방 익숙해집니다.

## 도형이 아니라 텍스처를 그린다는 것

raylib나 Dear ImGui 같은 라이브러리에 익숙하다면 `DrawRectangle()`이나 `DrawCircle()`처럼 도형을 직접 그리는 함수를 기대했을 수도 있습니다. 하지만 MonoGame의 `SpriteBatch`에는 그런 도형 그리기 함수가 없습니다. MonoGame은 철저하게 텍스처(이미지) 기반으로 동작하는 프레임워크이기 때문입니다.

그렇다면 단색 사각형 하나를 그리고 싶을 때는 어떻게 할까요? MonoGame 커뮤니티에서 널리 쓰이는 방법은 1x1 크기의 흰색 텍스처를 하나 만들어두고, 이를 원하는 크기로 확대해서 그리는 것입니다.

```csharp
Texture2D pixel = new Texture2D(GraphicsDevice, 1, 1);
pixel.SetData(new[] { Color.White });

// 이후 그리기 코드에서
_spriteBatch.Draw(pixel, new Rectangle(50, 50, 200, 30), Color.Red);
```

1x1 흰색 텍스처를 `Rectangle` 오버로드로 늘려서 그리면, 지정한 크기의 단색 사각형처럼 보입니다. `Color` 인자로 원하는 색을 곱해주면 되므로 색상도 자유롭게 바꿀 수 있습니다. 체력 바, 디버그용 히트박스 표시, 간단한 UI 배경 등을 그릴 때 이 기법이 자주 쓰입니다. 도형 전용 API가 없는 대신, 텍스처라는 하나의 개념으로 이미지든 단색 도형이든 동일한 방식으로 처리할 수 있다는 점이 MonoGame다운 설계라고 할 수 있습니다.

## 요약

- `SpriteBatch`는 여러 개의 2D 그리기 호출을 모아서 효율적으로 GPU에 전달하는 MonoGame의 핵심 클래스이며, `GraphicsDevice`를 인자로 받아 생성한다.
- 모든 `Draw()` 호출은 반드시 `spriteBatch.Begin()`과 `spriteBatch.End()` 사이, 그리고 `Game.Draw()` 메서드 안에서 이루어져야 한다.
- `Draw()`는 위치만 받는 가장 단순한 오버로드부터, 목적지 사각형, 소스 사각형(텍스처 일부만 그리기), 회전, 원점, 스케일, 반전 효과, 레이어 깊이까지 포함하는 전체 오버로드까지 다양하게 제공된다.
- `sourceRectangle`로 텍스처의 일부만 잘라 그리는 기능은 8장에서 다룰 스프라이트 시트 애니메이션의 기반이 된다.
- `origin`은 회전과 스케일의 기준점이므로, 자연스러운 회전을 위해서는 보통 텍스처의 중심을 원점으로 지정한다.
- MonoGame은 텍스처 기반 프레임워크이며, 단색 도형은 1x1 흰색 텍스처를 확대해서 그리는 관용적인 방법으로 표현한다.

## 연습문제

1. `SpriteBatch.Begin()`과 `End()` 사이에 그리기 호출을 넣어야 하는 이유를 GPU 그리기 호출의 관점에서 설명해보세요.
2. 텍스처를 화면 중앙에 원본 크기 그대로, 회전 없이 그리는 코드를 `Draw()`의 전체 오버로드를 사용해 작성해보세요. `origin`을 어떻게 설정해야 텍스처의 중심이 화면 중앙에 오는지 생각해보세요.
3. `sourceRectangle`을 사용해 128x128 크기의 스프라이트 시트에서 오른쪽 아래 32x32 칸 하나만 그리는 코드를 작성해보세요.
4. `SpriteEffects.FlipHorizontally`를 사용하면 캐릭터가 왼쪽을 보는 텍스처만으로 오른쪽을 보는 모습도 표현할 수 있습니다. 이 방식의 장점과, 반대로 텍스처를 따로 준비하는 것이 더 나을 수 있는 상황을 각각 하나씩 들어보세요.
5. 1x1 흰색 텍스처를 이용해 화면 왼쪽 위에 너비 200, 높이 20인 초록색 체력 바 배경과, 그 위에 현재 체력 비율만큼 너비가 줄어드는 빨간색 체력 바를 겹쳐 그리는 코드를 작성해보세요.

---

[◀ 이전: 4장. 콘텐츠 파이프라인(Content Pipeline)](ch04-콘텐츠파이프라인.md) | [📖 목차](00-목차.md) | [다음: 6장. 입력 처리 ▶](ch06-입력처리.md)
