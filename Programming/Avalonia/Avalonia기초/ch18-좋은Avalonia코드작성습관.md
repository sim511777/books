# 18장. 좋은 Avalonia 코드 작성 습관

[◀ 이전: 17장. 플랫폼별 배포](ch17-플랫폼별배포.md) | [📖 목차](00-목차.md) | [다음: 부록 ▶](appendix-부록.md)


## 들어가며

지금까지 17개 장에 걸쳐 Avalonia의 기본 문법부터 레이아웃, 데이터 바인딩, MVVM, 커맨드, 스타일, 템플릿, 커스텀 컨트롤, 컬렉션 바인딩, 네비게이션, 테마, 배포까지 폭넓게 살펴보았습니다. 이 마지막 장에서는 새로운 기능을 배우기보다는, 지금까지 배운 내용을 실제 프로젝트에 적용할 때 코드 품질을 좌우하는 몇 가지 습관을 정리해 보려 합니다.

좋은 Avalonia 코드를 쓰는 비결은 사실 대단히 특별한 기법에 있지 않습니다. 오히려 이 책 전체에서 강조해 온 원칙들 - View와 ViewModel의 분리, 재사용 가능한 리소스 구성, 데이터 바인딩을 통한 선언적 UI 작성 - 을 얼마나 일관되게 지키느냐가 관건입니다. 이번 장을 통해 그 원칙들을 습관으로 만드는 방법을 정리하고, 마지막으로 이 책 전체를 돌아보며 어디로 더 나아갈 수 있을지 살펴보겠습니다.

## 코드비하인드를 최소화하고 로직은 ViewModel에

8장에서 MVVM 패턴을 배우면서 View(.axaml과 그 코드비하인드 파일)와 ViewModel의 역할을 분리하는 것이 왜 중요한지 다루었습니다. 실전에서 가장 흔히 무너지는 원칙이 바로 이 부분입니다. 처음에는 작은 편의를 위해 코드비하인드에 로직 한 줄을 추가했다가, 시간이 지나면서 코드비하인드 파일이 점점 비대해지는 경우가 많습니다.

다음과 같은 코드비하인드는 경계해야 할 신호입니다.

```csharp
// 코드비하인드(MainWindow.axaml.cs)에 비즈니스 로직이 섞여 있는 예
public partial class MainWindow : Window
{
    public MainWindow()
    {
        InitializeComponent();
    }

    private void OnSaveButtonClick(object? sender, RoutedEventArgs e)
    {
        // 파일 저장, 유효성 검사 등 실제 로직이 View 코드에 직접 들어가 있음
        var text = MyTextBox.Text;
        if (string.IsNullOrWhiteSpace(text))
        {
            return;
        }
        File.WriteAllText("output.txt", text);
    }
}
```

이 코드는 당장은 동작하지만, UI 프레임워크에 직접 의존하는 코드 안에 파일 입출력과 유효성 검사 로직이 섞여 있어 단위 테스트를 작성하기 어렵고, 나중에 로직을 재사용하기도 힘듭니다. 같은 기능을 ViewModel과 커맨드(9장)로 옮기면 다음과 같이 정리됩니다.

```csharp
public class MainWindowViewModel : ViewModelBase
{
    private string? _text;
    public string? Text
    {
        get => _text;
        set => SetProperty(ref _text, value);
    }

    public ICommand SaveCommand { get; }

    public MainWindowViewModel()
    {
        SaveCommand = new RelayCommand(Save, () => !string.IsNullOrWhiteSpace(Text));
    }

    private void Save()
    {
        File.WriteAllText("output.txt", Text);
    }
}
```

```xml
<TextBox Text="{Binding Text}"/>
<Button Content="저장" Command="{Binding SaveCommand}"/>
```

코드비하인드에는 이제 `InitializeComponent()` 호출 외에 거의 아무것도 남지 않습니다. 물론 코드비하인드를 100% 없애야 한다는 절대적인 규칙은 아닙니다. 애니메이션 제어처럼 View 계층의 요소를 직접 다뤄야 하는 극히 일부의 경우는 코드비하인드에 남기는 것이 오히려 자연스럽습니다. 다만 "이 코드가 비즈니스 로직인가, 아니면 순수하게 화면 표현에 관한 코드인가"를 습관적으로 자문해 보고, 전자라면 ViewModel로 옮기는 것을 기본 원칙으로 삼는 것이 좋습니다.

## View와 ViewModel의 네이밍 컨벤션

프로젝트가 커지면 View와 ViewModel 파일이 수십, 수백 개로 늘어납니다. 이때 일관된 네이밍 규칙이 없다면 어떤 View가 어떤 ViewModel과 짝을 이루는지 찾는 데만 시간을 허비하게 됩니다. Avalonia와 .NET 커뮤니티에서 널리 쓰이는 관례는 View 이름 뒤에 `ViewModel`을 붙이는 방식입니다.

| View | ViewModel |
|---|---|
| `MainWindow` | `MainWindowViewModel` |
| `SettingsView` | `SettingsViewModel` |
| `UserListView` | `UserListViewModel` |
| `UserDetailView` | `UserDetailViewModel` |

이 규칙을 프로젝트 전체에서 지키면 다음과 같은 실질적인 이점이 있습니다.

- 파일 탐색기나 IDE의 파일 목록에서 이름만으로 짝을 바로 찾을 수 있습니다.
- 새 화면을 추가할 때 "이 View의 ViewModel 이름은 무엇으로 지어야 하나"를 고민할 필요가 없습니다.
- 11장에서 다룬 `DataTemplate` 기반의 뷰 자동 매핑(ViewModel 타입에 대응하는 View를 자동으로 찾아 연결하는 패턴)을 도입할 때도, 이름 규칙이 일관되어 있으면 매핑 로직을 단순하게 유지할 수 있습니다.

폴더 구조 역시 `Views` 폴더와 `ViewModels` 폴더를 나란히 두고, 그 안에서 같은 이름 접두어를 사용하는 것이 일반적인 관례입니다.

```
Views/
    MainWindow.axaml
    SettingsView.axaml
ViewModels/
    MainWindowViewModel.cs
    SettingsViewModel.cs
```

## 리소스와 스타일을 재사용 가능하게 분리하기

10장에서 스타일과 리소스 딕셔너리를 배웠습니다. 실전에서 가장 자주 나타나는 나쁜 습관은 색상 값이나 여백 크기 같은 값을 여러 XAML 파일에 그대로 반복해서 적어 넣는 것입니다.

```xml
<!-- 여러 파일에 같은 색상 값이 하드코딩되어 흩어져 있는 예 -->
<Border Background="#2D2D30" BorderBrush="#3F3F46"/>
```

이렇게 작성하면 나중에 브랜드 색상 하나를 바꾸는 작업조차 프로젝트 전체를 뒤져야 하는 큰 작업이 되어 버립니다. 대신 이런 값을 `App.axaml`이나 별도의 리소스 딕셔너리 파일에 이름 붙여 정의해 두고, 각 XAML에서는 그 이름을 참조하는 습관을 들여야 합니다.

```xml
<!-- App.axaml 또는 별도의 리소스 딕셔너리 -->
<Application.Resources>
    <SolidColorBrush x:Key="PanelBackgroundBrush" Color="#2D2D30"/>
    <SolidColorBrush x:Key="PanelBorderBrush" Color="#3F3F46"/>
    <Thickness x:Key="DefaultPanelMargin">12</Thickness>
</Application.Resources>
```

```xml
<!-- 실제 사용하는 곳 -->
<Border Background="{StaticResource PanelBackgroundBrush}"
        BorderBrush="{StaticResource PanelBorderBrush}"
        Margin="{StaticResource DefaultPanelMargin}"/>
```

이렇게 분리해 두면 16장에서 다룬 테마 전환 기능을 붙일 때도 리소스 값만 교체하면 되므로 훨씬 수월합니다. 반복되는 컨트롤 모양은 `Style`과 `ControlTemplate`(10장, 11장)로 뽑아내고, 반복되는 값은 리소스 딕셔너리로 뽑아내는 습관을 처음부터 들여 놓으면, 프로젝트가 커져도 일관된 디자인을 유지하는 비용이 크게 줄어듭니다.

## 디자인타임 데이터로 미리보기 확인하며 작업하기

Avalonia는 XAML 에디터(Visual Studio, Rider, VS Code의 Avalonia 확장 등)에서 실제로 애플리케이션을 실행하지 않고도 화면 모양을 미리 볼 수 있는 디자인타임 미리보기 기능을 제공합니다. 이 기능을 제대로 활용하려면 `d:` 네임스페이스로 시작하는 디자인타임 속성을 XAML에 함께 적어 두는 습관이 필요합니다.

```xml
<UserControl xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
             xmlns:vm="using:MyApp.ViewModels"
             mc:Ignorable="d"
             xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
             d:DesignWidth="400" d:DesignHeight="300"
             d:DataContext="{d:DesignInstance Type=vm:UserDetailViewModel}"
             x:Class="MyApp.Views.UserDetailView">
    <StackPanel Margin="16" Spacing="8">
        <TextBlock Text="{Binding UserName}" FontSize="18" FontWeight="Bold"/>
        <TextBlock Text="{Binding Email}"/>
    </StackPanel>
</UserControl>
```

`d:DesignWidth`와 `d:DesignHeight`는 미리보기 캔버스의 크기를 지정해, 실제 창 크기와 비슷한 조건에서 레이아웃이 어떻게 잡히는지 확인할 수 있게 해줍니다. `d:DataContext`(정적인 디자인타임 인스턴스를 지정하는 `{d:DesignInstance ...}` 구문과 함께 사용)는 실제 애플리케이션을 실행하지 않고도 바인딩된 속성 이름이 올바른지, 대략 어떤 값이 들어올 때 화면이 어떻게 보이는지를 에디터 안에서 바로 확인할 수 있게 해줍니다.

`mc:Ignorable="d"`를 함께 지정해 두면 이 디자인타임 전용 속성들은 실제 빌드 및 실행 시에는 무시되므로, 런타임 동작에 아무런 영향을 주지 않으면서 개발 편의성만 얻을 수 있습니다. 화면 하나를 만들 때마다 매번 애플리케이션 전체를 실행해서 확인하는 대신, 이렇게 디자인타임 데이터를 미리 채워 두고 에디터의 미리보기 창을 활용하는 습관을 들이면 반복 작업 시간을 크게 줄일 수 있습니다.

## 이 책을 돌아보며: 지금까지 배운 것들

여기서 잠시 멈춰서, 이 책 전체에서 다룬 내용을 한 번 정리해 보겠습니다.

1장에서는 Avalonia가 어떤 프레임워크인지, 왜 크로스플랫폼 데스크톱 UI 개발에 좋은 선택이 되는지 살펴보았습니다. 2장과 3장에서는 첫 프로젝트를 만들고 XAML의 기본 문법을 익혔으며, 4장과 5장에서는 레이아웃 패널과 기본 컨트롤들을 다뤘습니다. 6장에서는 이벤트 처리 방식을 배웠고, 그 흐름은 곧바로 7장의 데이터 바인딩과 8장의 MVVM 패턴으로 이어졌습니다.

돌이켜 보면 **7장의 데이터 바인딩과 8장의 MVVM 패턴**이야말로 이 책 전체를 관통하는 두 개의 기둥이었습니다. 9장의 커맨드, 10장의 스타일과 리소스, 11장의 컨트롤 템플릿과 데이터 템플릿, 12장과 13장의 사용자 정의 컨트롤, 14장의 리스트와 컬렉션 바인딩, 15장의 네비게이션과 다이얼로그, 16장의 테마 - 이 모든 주제는 결국 "View와 ViewModel을 데이터 바인딩으로 느슨하게 연결하고, 그 위에서 UI를 선언적으로 조립한다"는 하나의 원칙 위에 세워져 있었습니다. 이번 18장에서 다룬 코드비하인드 최소화, 네이밍 컨벤션, 리소스 분리, 디자인타임 데이터 활용 같은 습관들도 결국 이 원칙을 실전에서 무너뜨리지 않기 위한 구체적인 실천 방법이라고 할 수 있습니다.

어떤 장이 유독 어렵게 느껴졌다면, 그 장의 내용이 대부분 데이터 바인딩과 MVVM이라는 기초 위에 쌓여 있다는 점을 기억하고 7장과 8장을 다시 훑어보는 것도 좋은 복습 방법입니다.

## 더 공부할 방향

이 책은 Avalonia의 기초를 다지는 데 초점을 맞추었지만, Avalonia와 그 주변 생태계에는 더 깊이 파고들 만한 주제가 많이 남아 있습니다. 다음 학습 단계로 관심을 가져볼 만한 방향을 이름 수준에서 소개합니다.

- **ReactiveUI** - Avalonia와 함께 자주 쓰이는 반응형 MVVM 프레임워크로, 옵저버블 스트림을 활용해 더 정교한 데이터 흐름과 비동기 처리를 표현할 수 있습니다.
- **CommunityToolkit.Mvvm** - 소스 제너레이터를 활용해 `INotifyPropertyChanged` 구현이나 커맨드 선언 같은 반복적인 MVVM 코드를 크게 줄여주는 라이브러리입니다.
- **애니메이션과 트랜지션 시스템** - Avalonia는 속성 값 변화에 애니메이션을 적용하거나 화면 전환에 시각적 효과를 주는 기능을 제공합니다. 정적인 화면을 넘어 동적인 사용자 경험을 만들고 싶다면 살펴볼 가치가 있습니다.
- **접근성(Accessibility)** - 스크린 리더 지원이나 키보드만으로의 조작성 등, 더 많은 사용자가 애플리케이션을 사용할 수 있도록 돕는 기능들입니다.

이 주제들은 이 책에서 다진 데이터 바인딩과 MVVM 기초 위에서 훨씬 수월하게 익힐 수 있습니다. 부록에서는 개발 환경 설정을 더 자세히 정리하고, 이 책에서 사용한 주요 문법을 표로 요약하며, 앞으로 참고할 만한 자료를 안내합니다.

## 요약

- MVVM을 실전에서 지키는 핵심은 코드비하인드에 비즈니스 로직을 남기지 않고, 데이터와 동작은 최대한 ViewModel과 커맨드로 옮기는 습관입니다.
- `MainWindow`/`MainWindowViewModel`처럼 View 이름 뒤에 `ViewModel`을 붙이는 네이밍 컨벤션을 지키면, 프로젝트 규모가 커져도 View와 ViewModel의 짝을 쉽게 찾을 수 있습니다.
- 색상, 여백 같은 값과 반복되는 컨트롤 모양은 리소스 딕셔너리와 스타일로 분리해 두면, 나중에 디자인을 일괄 변경하거나 테마를 전환하는 작업이 훨씬 쉬워집니다.
- `d:DesignWidth`, `d:DesignHeight`, `d:DataContext`(`{d:DesignInstance ...}`) 같은 디자인타임 속성을 활용하면, 애플리케이션을 매번 실행하지 않고도 XAML 에디터의 미리보기에서 화면 모양을 바로 확인할 수 있습니다.
- 이 책 전체에서 7장의 데이터 바인딩과 8장의 MVVM 패턴은 다른 모든 장의 기반이 되는 핵심 개념이며, 앞으로 ReactiveUI, CommunityToolkit.Mvvm, 애니메이션/트랜지션, 접근성 등으로 학습을 넓혀갈 수 있습니다.

## 연습문제

1. 코드비하인드에 비즈니스 로직이 섞여 있는 것이 왜 문제가 되는지, 테스트 용이성과 유지보수성 관점에서 설명해 보세요.
2. `SettingsView`라는 이름의 View가 있다면, 네이밍 컨벤션에 따라 그 ViewModel의 이름은 무엇이어야 할지 답해 보세요.
3. 색상 값을 XAML 여러 곳에 직접 하드코딩하는 대신 리소스 딕셔너리로 분리했을 때 얻을 수 있는 이점을 한 가지 이상 설명해 보세요.
4. `d:DesignWidth`, `d:DataContext` 같은 디자인타임 속성이 실제 애플리케이션 실행 결과에 영향을 주지 않는 이유를 `mc:Ignorable="d"`와 연결지어 설명해 보세요.
5. 이 책에서 배운 18개 장의 내용 중, 7장 데이터 바인딩과 8장 MVVM 패턴이 다른 장들의 기반이 되는 이유를 자신의 언어로 정리해 보세요.

---

[◀ 이전: 17장. 플랫폼별 배포](ch17-플랫폼별배포.md) | [📖 목차](00-목차.md) | [다음: 부록 ▶](appendix-부록.md)
