# 9장. 커맨드(ICommand)

[◀ 이전: 8장. MVVM 패턴](ch08-MVVM패턴.md) | [📖 목차](00-목차.md) | [다음: 10장. 스타일과 리소스 ▶](ch10-스타일과리소스.md)


6장에서는 버튼을 클릭했을 때 어떤 동작을 실행하기 위해 코드비하인드에서 `Click` 이벤트 핸들러를 작성하는 방법을 배웠다. 이 방식은 간단한 예제에서는 아무 문제가 없지만, 7장과 8장에서 살펴본 MVVM 패턴의 관점에서 보면 걸리는 부분이 있다. `Click` 이벤트 핸들러는 코드비하인드, 즉 View 클래스의 일부이기 때문에 버튼을 눌렀을 때 실행되는 로직이 View에 남아 있게 된다. ViewModel만 따로 떼어 테스트하고 싶어도, 정작 "저장" 버튼을 눌렀을 때 무슨 일이 일어나는지는 View 코드를 열어 봐야 알 수 있는 것이다.

이 장에서는 이 문제를 해결하는 Avalonia(그리고 WPF 계열 프레임워크 전반)의 표준적인 방법인 `ICommand`를 다룬다. 버튼의 동작을 이벤트 핸들러가 아니라 ViewModel의 속성으로 노출하고, XAML에서는 `Command` 속성으로 그 속성에 바인딩하는 방식이다.

## 9.1 왜 커맨드가 필요한가

8장에서 만든 ViewModel에는 이미 `INotifyPropertyChanged`를 구현한 속성들이 있었다. 여기에 "버튼을 누르면 실행되는 동작"까지 속성처럼 노출할 수 있다면, View는 여전히 XAML만으로 완결되고 ViewModel은 View를 전혀 몰라도 되는 상태를 유지할 수 있다.

코드비하인드 방식과 커맨드 방식을 나란히 놓고 비교해 보자.

```csharp
// 코드비하인드 방식 (6장)
private void OnSaveClick(object? sender, RoutedEventArgs e)
{
    // 저장 로직이 View 클래스 안에 들어 있다
    SaveToDatabase();
}
```

```xml
<Button Content="저장" Click="OnSaveClick" />
```

커맨드 방식에서는 "저장"이라는 동작 자체를 ViewModel이 소유하고, View는 그 동작을 가리키기만 한다.

```xml
<Button Content="저장" Command="{Binding SaveCommand}" />
```

`SaveCommand`는 ViewModel에 정의된 속성이며, 그 타입은 `.NET`에 이미 정의되어 있는 `System.Windows.Input.ICommand` 인터페이스다. Avalonia는 이 인터페이스를 표준으로 채택하고 있으므로, WPF나 다른 XAML 기반 프레임워크의 경험이 있다면 낯설지 않을 것이다.

## 9.2 ICommand 인터페이스

`ICommand`는 `System.Windows.Input` 네임스페이스에 정의된 아주 단순한 인터페이스로, 멤버는 세 개뿐이다.

```csharp
public interface ICommand
{
    void Execute(object? parameter);
    bool CanExecute(object? parameter);
    event EventHandler? CanExecuteChanged;
}
```

각 멤버의 역할은 다음과 같다.

- **`Execute(object? parameter)`**: 커맨드가 실제로 수행할 동작. 버튼이 클릭되면 Avalonia는 바인딩된 커맨드의 `Execute`를 호출한다.
- **`CanExecute(object? parameter)`**: 지금 이 커맨드를 실행할 수 있는 상태인지를 반환한다. 예를 들어 필수 입력 항목이 비어 있다면 `false`를 반환하여 저장을 막을 수 있다.
- **`CanExecuteChanged`**: `CanExecute`의 반환값이 바뀔 수 있는 시점에 발생시키는 이벤트다. 이 이벤트를 구독한 UI 요소(버튼 등)는 이벤트가 발생하면 `CanExecute`를 다시 호출해 자신의 활성화 상태를 갱신한다.

`parameter`는 `CommandParameter`라는 별도의 XAML 속성으로 함께 전달할 수 있는 추가 데이터다. 대부분의 간단한 커맨드는 이 매개변수를 사용하지 않지만, 리스트의 특정 항목을 삭제하는 커맨드처럼 "어떤 대상에 대해 실행할지"가 필요한 경우 유용하다(자세한 활용은 14장에서 다시 다룬다).

## 9.3 RelayCommand 직접 구현하기

`ICommand`를 구현하는 클래스를 매번 새로 만드는 것은 번거롭다. 커맨드마다 필요한 것은 결국 "무엇을 실행할지"와 "언제 실행 가능한지"뿐이므로, 이 두 가지를 생성자로 받아 처리하는 범용 클래스를 하나 만들어 두면 충분하다. 이런 클래스를 흔히 `RelayCommand` 또는 `DelegateCommand`라고 부른다.

```csharp
using System;
using System.Windows.Input;

namespace AvaloniaGuide.ViewModels;

public class RelayCommand : ICommand
{
    private readonly Action<object?> _execute;
    private readonly Func<object?, bool>? _canExecute;

    public RelayCommand(Action<object?> execute, Func<object?, bool>? canExecute = null)
    {
        _execute = execute ?? throw new ArgumentNullException(nameof(execute));
        _canExecute = canExecute;
    }

    // 매개변수가 필요 없는 커맨드를 위한 편의 생성자
    public RelayCommand(Action execute, Func<bool>? canExecute = null)
        : this(_ => execute(), canExecute is null ? null : _ => canExecute())
    {
    }

    public event EventHandler? CanExecuteChanged;

    public bool CanExecute(object? parameter) => _canExecute?.Invoke(parameter) ?? true;

    public void Execute(object? parameter) => _execute(parameter);

    public void RaiseCanExecuteChanged() => CanExecuteChanged?.Invoke(this, EventArgs.Empty);
}
```

동작 방식은 단순하다.

- `execute`로 전달된 델리게이트를 `Execute`가 그대로 호출한다.
- `canExecute`가 주어지지 않았다면 `CanExecute`는 항상 `true`를 반환한다("항상 실행 가능").
- ViewModel 쪽에서 `CanExecute`의 결과가 바뀔 만한 상황(예: 입력값이 변경됨)이 생기면 `RaiseCanExecuteChanged()`를 직접 호출해 UI에 재평가를 요청한다.

이제 이 `RelayCommand`를 ViewModel에서 사용해 보자. 사용자 이름을 입력받아 저장하는 간단한 예제다.

```csharp
using System.ComponentModel;
using System.Runtime.CompilerServices;

namespace AvaloniaGuide.ViewModels;

public class MainViewModel : INotifyPropertyChanged
{
    private string _userName = string.Empty;

    public string UserName
    {
        get => _userName;
        set
        {
            if (_userName == value)
                return;

            _userName = value;
            OnPropertyChanged();

            // UserName이 바뀔 때마다 SaveCommand의 실행 가능 여부를 다시 확인시킨다
            SaveCommand.RaiseCanExecuteChanged();
        }
    }

    public RelayCommand SaveCommand { get; }

    public MainViewModel()
    {
        SaveCommand = new RelayCommand(
            execute: Save,
            canExecute: () => !string.IsNullOrWhiteSpace(UserName));
    }

    private void Save()
    {
        // 실제로는 파일이나 데이터베이스에 저장하는 로직이 들어간다
        System.Diagnostics.Debug.WriteLine($"저장됨: {UserName}");
    }

    public event PropertyChangedEventHandler? PropertyChanged;

    protected void OnPropertyChanged([CallerMemberName] string? propertyName = null)
        => PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName));
}
```

이 코드에는 View, `Button`, `Click` 이벤트가 전혀 등장하지 않는다는 점에 주목하자. `MainViewModel`만 놓고도 `UserName`에 빈 문자열을 넣었을 때 `SaveCommand.CanExecute(null)`이 `false`를 반환하는지 단위 테스트로 검증할 수 있다. 이것이 커맨드를 사용하는 가장 큰 이유다.

## 9.4 XAML에서 Command 바인딩하기

ViewModel이 준비되었으니 View에서는 `Command` 속성을 바인딩하기만 하면 된다.

```xml
<Window xmlns="https://github.com/avaloniaui"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:vm="using:AvaloniaGuide.ViewModels"
        x:Class="AvaloniaGuide.Views.MainWindow"
        x:DataType="vm:MainViewModel"
        Title="커맨드 예제" Width="360" Height="200">
    <StackPanel Margin="16" Spacing="8">
        <TextBox Watermark="이름을 입력하세요"
                 Text="{Binding UserName}" />
        <Button Content="저장" Command="{Binding SaveCommand}" />
    </StackPanel>
</Window>
```

`Command="{Binding SaveCommand}"`는 7장에서 배운 것과 똑같은 바인딩 문법이다. 다만 바인딩 대상이 문자열이나 숫자 같은 값이 아니라 `ICommand` 객체라는 점이 다를 뿐이다. 추가로 전달할 값이 있다면 `CommandParameter`를 함께 사용한다.

```xml
<Button Content="삭제"
        Command="{Binding DeleteCommand}"
        CommandParameter="{Binding SelectedItem}" />
```

이 경우 `DeleteCommand.Execute`가 호출될 때 `SelectedItem`의 값이 `parameter` 인자로 전달된다. 리스트 항목을 대상으로 한 커맨드는 14장에서 더 자세히 다룬다.

## 9.5 CanExecute와 버튼 자동 비활성화

`Button`은 `ICommandSource`라는 역할을 겸하고 있어서, `Command`에 바인딩된 객체의 `CanExecuteChanged` 이벤트를 자동으로 구독한다. 그래서 개발자가 직접 `IsEnabled`를 조작하지 않아도 다음과 같은 흐름이 그대로 동작한다.

1. 앱이 시작되면 `Button`은 `SaveCommand.CanExecute(null)`을 호출해 초기 활성화 상태를 결정한다. `UserName`이 비어 있으므로 `false`가 반환되고, 버튼은 비활성화된 채로 표시된다.
2. 사용자가 `TextBox`에 글자를 입력하면 `UserName` 속성의 setter가 실행되고, `OnPropertyChanged()`에 이어 `SaveCommand.RaiseCanExecuteChanged()`가 호출된다.
3. `RaiseCanExecuteChanged()`는 `CanExecuteChanged` 이벤트를 발생시키고, 이를 구독하고 있던 `Button`은 즉시 `CanExecute`를 다시 호출한다.
4. 이번에는 `UserName`이 채워져 있으므로 `true`가 반환되고, `Button`은 스스로 `IsEnabled`를 `true`로 바꾸며 다시 클릭 가능한 상태가 된다.

즉 `RaiseCanExecuteChanged()`를 호출하는 시점을 잘 챙겨주기만 하면, 버튼의 활성화/비활성화 처리는 별도의 코드 없이 프레임워크가 알아서 해 준다. 이 패턴은 "필수 항목을 모두 채워야 등록 버튼이 눌린다", "선택된 항목이 있어야 삭제 버튼이 눌린다"와 같은 흔한 UI 요구사항을 매우 자연스럽게 표현해 준다.

## 9.6 보일러플레이트 줄이기: CommunityToolkit.Mvvm

지금까지 만든 `RelayCommand`는 프로젝트마다 거의 똑같이 반복해서 작성하게 되는 코드다. 커맨드뿐 아니라 8장에서 다룬 `INotifyPropertyChanged` 구현 역시 매 속성마다 비슷한 코드가 반복된다는 것을 눈치챘을 것이다.

실무에서는 이런 반복 작업을 줄이기 위해 마이크로소프트가 제공하는 `CommunityToolkit.Mvvm` NuGet 패키지를 널리 사용한다. 이 라이브러리는 `[RelayCommand]`라는 특성(attribute)을 메서드에 붙이기만 하면 소스 제너레이터가 컴파일 시점에 `ICommand` 속성을 자동으로 생성해 주는 기능을 제공한다. 이 책에서는 지금까지처럼 `ICommand`가 내부적으로 어떻게 동작하는지 직접 구현해 보며 이해하는 데 집중하고, `CommunityToolkit.Mvvm`의 구체적인 사용법은 다루지 않는다. 다만 기본 동작 원리를 알고 있으면 이런 라이브러리가 생성해 주는 코드가 정확히 무엇을 대신해 주는 것인지 쉽게 파악할 수 있으므로, 이름 정도는 기억해 두고 넘어가자.

## 요약

- 코드비하인드의 `Click` 이벤트는 로직을 View에 묶어 두지만, `ICommand`는 그 로직을 ViewModel의 속성으로 노출시켜 MVVM 구조를 유지하게 해 준다.
- `ICommand`는 `Execute`, `CanExecute`, `CanExecuteChanged` 세 멤버로 구성된 단순한 인터페이스다.
- `RelayCommand`(또는 `DelegateCommand`)는 실행 로직과 실행 가능 조건을 생성자로 받아 `ICommand`를 구현하는 범용 클래스로, 직접 만들어 재사용할 수 있다.
- XAML에서는 `Command="{Binding SaveCommand}"`로 커맨드를 바인딩하고, 필요하면 `CommandParameter`로 추가 값을 전달한다.
- `Button`은 `CanExecuteChanged` 이벤트를 자동으로 구독하므로, `CanExecute`가 `false`를 반환하면 별도 코드 없이 스스로 비활성화된다. 조건이 바뀔 때는 `RaiseCanExecuteChanged()`를 호출해 재평가를 요청해야 한다.
- `CommunityToolkit.Mvvm`의 `[RelayCommand]` 소스 제너레이터는 이런 보일러플레이트를 크게 줄여 주는 실무용 도구다.

## 연습문제

1. `ICommand` 인터페이스의 세 멤버(`Execute`, `CanExecute`, `CanExecuteChanged`)가 각각 어떤 역할을 하는지 자신의 말로 설명해 보라.
2. 이번 장의 `RelayCommand`를 참고하여, 매개변수를 받는 제네릭 버전 `RelayCommand<T>`를 설계해 보라. `Execute(object? parameter)`에서 전달받은 `parameter`를 `T`로 변환하는 부분을 어떻게 처리할지 고민해 보자.
3. `UserName`과 `Email` 두 속성이 모두 채워져 있어야만 활성화되는 `RegisterCommand`를 작성해 보라. 두 속성 중 어느 하나라도 바뀔 때 `RaiseCanExecuteChanged()`가 호출되도록 구성해야 한다.
4. 코드비하인드의 `Click` 이벤트 방식과 `ICommand` 방식 중 어느 쪽이 단위 테스트를 작성하기 쉬운지, 그 이유와 함께 설명해 보라.
5. `CommandParameter`를 사용해 리스트에서 특정 항목을 삭제하는 `DeleteCommand`를 설계한다면, `Execute`와 `CanExecute`는 각각 어떤 매개변수를 어떻게 활용해야 할지 구상해 보라.

---

[◀ 이전: 8장. MVVM 패턴](ch08-MVVM패턴.md) | [📖 목차](00-목차.md) | [다음: 10장. 스타일과 리소스 ▶](ch10-스타일과리소스.md)
