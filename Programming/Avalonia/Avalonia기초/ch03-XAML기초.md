# 3장. XAML 기초

[◀ 이전: 2장. 첫 프로그램과 프로젝트 구조](ch02-첫프로그램과프로젝트구조.md) | [📖 목차](00-목차.md) | [다음: 4장. 레이아웃 시스템 ▶](ch04-레이아웃시스템.md)


2장에서는 `dotnet new avalonia.app` 명령으로 프로젝트를 만들고, `MainWindow.axaml`과 `MainWindow.axaml.cs`라는 두 파일이 짝을 이루고 있다는 것을 확인했습니다. 이 장에서는 그 `.axaml` 파일 안에 적힌 XAML이라는 언어의 문법을 하나씩 뜯어보고, 그것이 어떻게 C# 코드와 연결되는지를 살펴봅니다.

## XAML이란 무엇인가

XAML(eXtensible Application Markup Language, "자멀"이라고 읽습니다)은 XML 문법을 그대로 따르는 마크업 언어입니다. 태그, 속성, 중첩 구조 같은 XML의 규칙을 빌려 와서 UI를 이루는 객체들의 트리 구조를 선언적으로 기술합니다.

"선언적"이라는 말은 "명령적"과 대비됩니다. 예를 들어 화면에 텍스트가 담긴 버튼을 하나 놓고 싶다고 해봅시다. C# 코드로 명령적으로 작성하면 다음과 같은 모습이 됩니다.

```csharp
var button = new Button();
button.Content = "확인";
button.Width = 120;
this.Content = button;
```

객체를 만들고, 속성을 하나씩 대입하고, 부모에 붙이는 절차를 순서대로 나열합니다. 반면 XAML은 결과물의 모양 자체를 트리 구조로 적습니다.

```xml
<Button Content="확인" Width="120" />
```

XAML 파서는 이 한 줄을 읽고 `Button` 클래스의 인스턴스를 만든 뒤, `Content` 속성에 `"확인"`을, `Width` 속성에 `120`을 대입하는 코드를 대신 생성해 줍니다. 즉 XAML 엘리먼트 하나는 결국 특정 C# 클래스의 인스턴스 하나에 대응되고, 엘리먼트의 속성은 그 클래스의 속성(프로퍼티)에 대응됩니다. 이 대응 관계만 기억하면 XAML 문서를 읽을 때 "이 태그는 어떤 클래스일까, 이 속성은 그 클래스의 어떤 프로퍼티일까"라고 되짚어 생각할 수 있습니다.

트리 구조로 UI를 적는 방식은 중첩 관계, 즉 "이 패널 안에 이 컨트롤들이 들어 있다"는 계층 구조를 눈으로 훑어보기에 C# 코드보다 훨씬 유리합니다. 실제로 어느 정도 규모가 있는 화면의 코드비하인드를 전부 C#으로 작성해 보면, 들여쓰기와 변수 이름만으로 트리 구조를 유지하는 일이 금방 번거로워진다는 것을 체감하게 됩니다. XAML은 이 문제를 XML의 태그 중첩으로 자연스럽게 해결합니다.

## 루트 엘리먼트와 네임스페이스

`.axaml` 파일은 하나의 최상위(root) 엘리먼트로 시작합니다. 창을 만드는 파일이라면 `Window`, 재사용 가능한 화면 조각을 만드는 파일이라면 `UserControl`이 root가 되는 경우가 가장 흔합니다. 2장에서 만든 `MainWindow.axaml`을 다시 열어 보면 다음과 비슷한 모습일 것입니다.

```xml
<Window xmlns="https://github.com/avaloniaui"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        x:Class="AvaloniaApp.MainWindow"
        Title="AvaloniaApp">
    <TextBlock Text="Hello, Avalonia!" />
</Window>
```

XML 문서는 태그 이름이 겹칠 수 있기 때문에 네임스페이스(namespace)로 태그가 어느 어휘 집합에 속하는지 구분합니다. 위 코드에서 `xmlns` 두 개가 바로 그 선언입니다.

- `xmlns="https://github.com/avaloniaui"`: 기본 네임스페이스입니다. 접두사 없이 쓰이는 모든 태그(`Window`, `TextBlock`, `Button`, `StackPanel` 등)는 이 네임스페이스, 즉 Avalonia의 컨트롤 라이브러리에서 찾는다는 뜻입니다.
- `xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"`: `x:` 접두사로 시작하는 특수한 지시어들을 위한 네임스페이스입니다. `x:Class`, `x:Name`, `x:Key` 같은 것들이 여기에 속합니다. 이 이름들은 컨트롤이 아니라 XAML 컴파일러 자신에게 보내는 지시에 가깝습니다.

`xmlns:x`의 값이 `microsoft.com` 도메인으로 되어 있는 것이 의아할 수 있는데, 이는 XAML이라는 언어 자체가 WPF에서 처음 정의되어 널리 쓰이기 시작했고, `x:` 네임스페이스의 스키마 자체는 그때 정의된 것을 그대로 재사용하기 때문입니다. Avalonia는 이 XAML 문법 표준을 그대로 채택하면서, 컨트롤이 속한 기본 네임스페이스만 자신의 것(`https://github.com/avaloniaui`)으로 바꾸었습니다. 실제로 이 URI들은 어딘가에 접속해서 내려받는 문서가 아니라 단순한 식별자 문자열이므로, 인터넷 연결 여부와 무관하게 항상 그대로 적으면 됩니다.

`x:Class="AvaloniaApp.MainWindow"`는 이 XAML 파일이 어떤 C# 클래스와 연결되는지를 지정합니다. 이 속성이 왜 필요한지는 이 장 마지막 절에서 자세히 다룹니다.

프로젝트 안에 여러 개의 창이나 사용자 컨트롤을 만들다 보면 매번 같은 두 줄의 `xmlns`를 타이핑하게 되는데, 대부분의 IDE(Visual Studio, Rider, VS Code + Avalonia 확장)는 새 Avalonia 파일을 만들 때 이 선언을 자동으로 채워 줍니다.

## 엘리먼트와 속성 문법

XAML에서 UI 요소 하나를 나타내는 가장 기본적인 형태는 엘리먼트(태그)이고, 그 세부 설정은 속성(attribute)으로 표현합니다.

```xml
<Button Content="저장" Width="100" Height="32" IsEnabled="False" />
```

이 한 줄은 `Button` 객체를 만들고 `Content`, `Width`, `Height`, `IsEnabled` 네 개의 프로퍼티를 초기화하는 것과 같습니다. 속성 값은 항상 문자열로 적지만, XAML 파서는 대상 프로퍼티의 실제 타입(`string`, `double`, `bool` 등)에 맞게 문자열을 변환해 줍니다. 이런 변환을 담당하는 요소를 타입 컨버터(type converter)라고 부르는데, 지금 단계에서는 "속성 값은 항상 따옴표로 감싼 문자열이지만 실제로는 알맞은 타입으로 바뀐다" 정도만 알아 두면 충분합니다.

자식 엘리먼트를 가질 수 있는 컨트롤은 태그를 열고 닫는 사이에 다른 엘리먼트를 중첩시킵니다.

```xml
<StackPanel>
    <TextBlock Text="이름" />
    <TextBox Watermark="이름을 입력하세요" />
    <Button Content="제출" />
</StackPanel>
```

`StackPanel`은 자식을 여러 개 담을 수 있는 컨테이너이므로 `Children`이라는 컬렉션 프로퍼티에 세 엘리먼트가 순서대로 추가됩니다. 다만 `Children`이라는 이름을 XAML에 직접 적는 일은 거의 없는데, `StackPanel`이나 `Panel`을 상속한 컨트롤들은 자식 컬렉션이 "콘텐츠 프로퍼티(content property)"로 지정되어 있어서 태그 안에 바로 중첩하는 것만으로 자동으로 그 컬렉션에 추가되기 때문입니다. 자식을 하나만 가질 수 있는 컨트롤(`Border`, `ScrollViewer` 등)도 마찬가지로 콘텐츠 프로퍼티가 지정되어 있어서, 태그 사이에 엘리먼트를 하나 적으면 그 프로퍼티에 대입됩니다.

빈 엘리먼트, 즉 자식이 없는 태그는 `<TextBlock Text="안녕" />`처럼 슬래시로 닫는 자기 닫힘(self-closing) 표기를 쓰는 것이 일반적입니다. XML 문법상 `<TextBlock Text="안녕"></TextBlock>`이라고 여는 태그와 닫는 태그를 따로 적어도 동일하게 동작하지만, 자식이 없는 경우 자기 닫힘 표기가 훨씬 간결합니다.

## 프로퍼티 엘리먼트 문법

지금까지 본 속성 문법은 값이 짧은 문자열일 때는 편리하지만, 값 자체가 복잡한 객체 트리여야 하는 경우에는 한 줄의 attribute로 표현할 수 없습니다. 예를 들어 버튼의 `Content`에 텍스트가 아니라 아이콘과 글자를 함께 배치한 패널을 넣고 싶다고 해봅시다. `Content="..."`처럼 attribute 안에 다른 태그를 넣을 방법은 XML 문법상 존재하지 않습니다.

이럴 때 쓰는 것이 프로퍼티 엘리먼트(property element) 문법입니다. `소유자타입.프로퍼티이름`이라는 형태의 태그를 만들어서, 그 사이에 값을 자식 엘리먼트로 적는 방식입니다.

```xml
<Button>
    <Button.Content>
        <StackPanel Orientation="Horizontal" Spacing="6">
            <TextBlock Text="⬇" />
            <TextBlock Text="다운로드" />
        </StackPanel>
    </Button.Content>
</Button>
```

`<Button.Content>`라는 태그는 실제로 존재하는 컨트롤이 아니라, "이 안에 있는 내용을 `Button`의 `Content` 프로퍼티에 대입하라"는 XAML 문법 규칙입니다. 이렇게 하면 `Content`처럼 `object` 타입인 프로퍼티에 문자열이 아니라 임의의 컨트롤 트리를 통째로 넣을 수 있습니다.

사실 `Button`의 `Content` 프로퍼티는 콘텐츠 프로퍼티로 지정되어 있으므로, 위 코드는 `<Button.Content>` 태그를 생략하고 다음처럼 더 짧게 쓸 수도 있습니다.

```xml
<Button>
    <StackPanel Orientation="Horizontal" Spacing="6">
        <TextBlock Text="⬇" />
        <TextBlock Text="다운로드" />
    </StackPanel>
</Button>
```

그렇다면 프로퍼티 엘리먼트 문법이 필요한 경우는 언제일까요. 바로 콘텐츠 프로퍼티가 아닌 다른 프로퍼티에 복잡한 값을 대입해야 할 때입니다. 10장에서 다룰 `Button.Styles`나, 컨트롤에 그림자·변환 효과를 주는 `Button.RenderTransform` 같은 프로퍼티들이 대표적인 예입니다.

```xml
<Border Background="White">
    <Border.BoxShadow>
        <BoxShadows>0 2 8 0 #40000000</BoxShadows>
    </Border.BoxShadow>
    <TextBlock Text="카드 형태의 콘텐츠" Margin="16" />
</Border>
```

`Border`는 `Child`가 콘텐츠 프로퍼티이므로 `TextBlock`은 바로 중첩할 수 있지만, `BoxShadow`는 콘텐츠 프로퍼티가 아니므로 `<Border.BoxShadow>`라는 프로퍼티 엘리먼트로 명시해야 합니다.

## x:Name으로 이름 붙이기

XAML로 선언한 컨트롤을 코드비하인드(`.axaml.cs` 파일)에서 직접 다루고 싶을 때가 있습니다. 예를 들어 버튼 클릭 시 텍스트박스의 내용을 읽어 와야 한다면, C# 코드 쪽에서 그 텍스트박스를 가리킬 방법이 필요합니다. 이때 사용하는 것이 `x:Name` 속성입니다.

```xml
<StackPanel Margin="16" Spacing="8">
    <TextBox x:Name="NameTextBox" Watermark="이름을 입력하세요" />
    <Button Content="인사하기" Click="OnGreetClicked" />
    <TextBlock x:Name="GreetingText" />
</StackPanel>
```

`x:Name="NameTextBox"`를 적어 두면, Avalonia의 XAML 컴파일러가 빌드 시점에 `NameTextBox`라는 이름의 필드를 코드비하인드 클래스 안에 자동으로 만들어 주고, `InitializeComponent()`가 실행될 때 그 필드가 실제 `TextBox` 인스턴스를 가리키도록 연결합니다. 그 결과 코드비하인드에서는 다음과 같이 마치 원래부터 있던 필드인 것처럼 자연스럽게 접근할 수 있습니다.

```csharp
private void OnGreetClicked(object? sender, RoutedEventArgs e)
{
    var name = NameTextBox.Text;
    GreetingText.Text = $"안녕하세요, {name}님!";
}
```

이 필드는 컴파일 시점에 생성되므로, `x:Name`으로 지정한 이름을 잘못 입력하면 XAML 파일이 아니라 코드비하인드 쪽에서 "그런 이름의 필드가 없다"는 컴파일 오류로 나타납니다. 처음 이런 오류를 만나면 당황할 수 있지만, `x:Name`이 자동 생성 필드 이름이라는 것을 알면 원인을 바로 짐작할 수 있습니다.

`x:Name` 대신 `Name`이라는 속성도 존재합니다. 많은 컨트롤 클래스가 자체적으로 `Name`이라는 CLR 프로퍼티를 가지고 있고, `x:Name`은 그 값을 설정함과 동시에 코드비하인드 필드까지 생성해 주는 조금 더 강력한 버전이라고 이해하면 됩니다. 실무에서는 코드비하인드 접근이 필요한 대부분의 경우에 `x:Name`을 사용합니다.

이벤트 처리(`Click="OnGreetClicked"`)에 대해서는 6장에서 자세히 다룹니다. 여기서는 XAML의 속성 값이 항상 데이터일 필요는 없고, 이렇게 코드비하인드에 정의된 메서드 이름을 가리킬 수도 있다는 정도만 짚고 넘어갑니다.

## 마크업 확장 맛보기

지금까지 본 속성 값은 모두 `"저장"`, `"100"`처럼 있는 그대로의 리터럴 문자열이었습니다. 그런데 XAML 문서를 조금만 더 살펴보면 중괄호로 감싸인 독특한 문법을 자주 마주치게 됩니다.

```xml
<TextBlock Text="{Binding UserName}" />
<Border Background="{StaticResource AccentBrush}" />
```

`{ }`로 감싼 이 표현을 마크업 확장(markup extension)이라고 부릅니다. 일반적인 attribute 값이 "이 문자열을 그대로 대입하라"는 뜻이라면, 마크업 확장은 "이 표현을 평가해서 나온 결과를 대입하라"는 뜻입니다. 즉 값을 즉석에서 계산하거나 다른 곳에서 가져오는 일종의 함수 호출처럼 동작합니다.

- `{Binding UserName}`은 "현재 데이터 컨텍스트(DataContext)에 있는 `UserName`이라는 프로퍼티의 값을 여기에 연결하라"는 뜻입니다. 데이터가 바뀌면 화면도 자동으로 갱신되는 데이터 바인딩의 기초가 되는 문법이며, 7장에서 원리와 다양한 옵션을 자세히 다룹니다.
- `{StaticResource AccentBrush}`는 "리소스 사전에 `AccentBrush`라는 키로 등록된 값을 찾아서 여기에 대입하라"는 뜻입니다. 색상이나 브러시, 스타일 등을 한곳에 정의해 두고 여러 곳에서 재사용할 때 쓰며, 10장 스타일과 리소스에서 다룹니다.

마크업 확장이 XML 표준 문법이 아니라 XAML이 독자적으로 추가한 문법이라는 점도 알아 둘 만합니다. XML 파서 입장에서 `{Binding UserName}`은 그저 `"{Binding UserName}"`이라는 평범한 문자열일 뿐이고, 이 문자열을 중괄호로 시작하는 특수한 표현으로 해석해서 별도의 객체(예를 들어 `Binding` 객체)로 바꾸는 일은 XAML 파서, 더 정확히는 Avalonia의 마크업 확장 처리기가 담당합니다. 지금 단계에서는 "중괄호로 시작하는 속성 값은 리터럴이 아니라 평가되는 표현식이다"라는 감각만 잡아 두면 충분하며, 각 마크업 확장의 구체적인 동작은 해당 장에서 예제와 함께 다시 설명합니다.

## XAML과 코드비하인드의 결합: InitializeComponent

이 장을 시작하면서 `x:Class="AvaloniaApp.MainWindow"`라는 속성을 잠깐 짚고 넘어갔습니다. 이제 이 속성이 실제로 어떤 일을 하는지 알아볼 차례입니다.

2장에서 만든 `MainWindow.axaml.cs`를 다시 살펴보면 다음과 같은 모습입니다.

```csharp
using Avalonia.Controls;

namespace AvaloniaApp;

public partial class MainWindow : Window
{
    public MainWindow()
    {
        InitializeComponent();
    }
}
```

여기서 `partial` 키워드가 핵심입니다. C#의 `partial class`는 하나의 클래스 정의를 여러 파일에 나누어 작성할 수 있게 해 주는 기능입니다. `MainWindow.axaml.cs`에는 생성자와 이벤트 처리 메서드 같은 "직접 작성한 코드"가 담겨 있고, 나머지 절반은 빌드 과정에서 XAML 컴파일러가 `MainWindow.axaml` 파일을 읽어 자동으로 생성합니다. 이렇게 자동 생성된 코드는 프로젝트의 `obj` 폴더 아래에 `MainWindow.axaml.g.cs` 같은 이름으로 만들어지며, 대략 다음과 같은 내용을 담고 있습니다(실제 생성 코드는 훨씬 복잡하지만 핵심만 단순화하면 이런 형태입니다).

```csharp
public partial class MainWindow : Window
{
    internal TextBox NameTextBox;
    internal TextBlock GreetingText;

    public void InitializeComponent()
    {
        AvaloniaXamlLoader.Load(this);
        NameTextBox = this.FindControl<TextBox>("NameTextBox");
        GreetingText = this.FindControl<TextBlock>("GreetingText");
    }
}
```

즉 `x:Class="AvaloniaApp.MainWindow"`는 "이 XAML 문서가 기술하는 트리를, `AvaloniaApp` 네임스페이스에 있는 `MainWindow`라는 partial 클래스의 절반으로 취급하라"는 연결 고리입니다. 이 속성 덕분에 XAML 컴파일러는 어느 클래스에 코드를 생성해 넣어야 할지 알 수 있고, `x:Name`으로 지정한 이름들이 그 클래스의 필드로 노출됩니다.

`AvaloniaXamlLoader.Load(this)`는 XAML 문서를 실제로 파싱해서 `Window`, `StackPanel`, `TextBox` 같은 객체 트리를 만들고, 그 트리를 `this`(현재 만들어지고 있는 `MainWindow` 인스턴스)의 콘텐츠로 채워 넣는 역할을 합니다. 생성자 안에서 `InitializeComponent()`를 호출하는 순서가 중요한 이유도 여기에 있습니다. 이 호출이 끝나기 전까지는 XAML에 선언된 자식 컨트롤들이 아직 만들어지지 않은 상태이므로, `x:Name`으로 노출되는 필드에 접근하면 `null`이 됩니다.

정리하면, `.axaml` 파일과 `.axaml.cs` 파일은 서로 다른 두 개의 클래스가 아니라 하나의 partial 클래스를 서로 다른 두 가지 방식(선언적인 XAML, 명령적인 C#)으로 나누어 적은 것입니다. `InitializeComponent()`가 그 둘을 하나로 잇는 다리 역할을 하며, 이 구조 덕분에 우리는 UI의 겉모습은 XAML로, 동작 로직은 C#으로 자연스럽게 역할을 나누어 작성할 수 있습니다.

## 요약

- XAML은 XML 문법을 사용해 객체 트리를 선언적으로 기술하는 마크업 언어이며, 엘리먼트는 클래스의 인스턴스에, 속성은 그 클래스의 프로퍼티에 대응됩니다.
- `.axaml` 파일은 `Window`나 `UserControl` 같은 root 엘리먼트로 시작하며, `xmlns`로 기본 네임스페이스(Avalonia 컨트롤)와 `x:` 네임스페이스(XAML 지시어)를 선언합니다.
- 짧은 값은 attribute 문법으로, 자식 트리 전체가 필요한 복잡한 값은 `소유자타입.프로퍼티` 형태의 프로퍼티 엘리먼트 문법으로 표현합니다. 콘텐츠 프로퍼티로 지정된 프로퍼티는 프로퍼티 엘리먼트 태그 없이 바로 중첩할 수 있습니다.
- `x:Name`은 코드비하인드에서 참조할 수 있도록 필드를 자동 생성해 주는 지시어이며, `InitializeComponent()` 호출 이후에만 유효한 값을 갖습니다.
- `{Binding ...}`, `{StaticResource ...}`처럼 중괄호로 시작하는 값은 마크업 확장이며, 리터럴이 아니라 평가되는 표현식입니다. 자세한 내용은 7장과 10장에서 다룹니다.
- `x:Class`로 지정된 클래스는 XAML 컴파일러가 생성하는 코드와 개발자가 작성한 코드비하인드가 합쳐지는 partial 클래스이며, `InitializeComponent()`가 XAML을 파싱해 실제 컨트롤 트리를 만들고 `x:Name` 필드를 연결해 줍니다.

## 연습문제

1. XAML에서 속성(attribute) 문법과 프로퍼티 엘리먼트 문법의 차이를 설명하고, 각각 어떤 상황에 적합한지 예를 들어 보세요.
2. `Window`의 root 엘리먼트에 `xmlns`와 `xmlns:x` 선언이 빠지면 어떤 문제가 생길지 추론해 보세요.
3. `x:Name="SubmitButton"`이 붙은 `Button`이 있을 때, 코드비하인드에서 이 버튼의 `IsEnabled`를 `false`로 바꾸는 코드를 작성해 보세요. 단, 이 코드는 `InitializeComponent()` 호출 이후에 실행되어야 합니다.
4. `Border` 컨트롤의 콘텐츠 프로퍼티는 `Child`입니다. `Border` 안에 `TextBlock` 하나를 넣는 XAML을, 프로퍼티 엘리먼트 문법을 명시적으로 사용한 버전과 생략한 버전 두 가지로 각각 작성해 보세요.
5. `{Binding}`과 `{StaticResource}`가 일반적인 attribute 값(리터럴 문자열)과 근본적으로 어떻게 다른지, 이 장에서 배운 내용을 바탕으로 자신의 말로 설명해 보세요.

---

[◀ 이전: 2장. 첫 프로그램과 프로젝트 구조](ch02-첫프로그램과프로젝트구조.md) | [📖 목차](00-목차.md) | [다음: 4장. 레이아웃 시스템 ▶](ch04-레이아웃시스템.md)
