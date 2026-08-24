# 10장. async/await 심화

[◀ 이전: 9장. async/await 기초](ch09-asyncawait기초.md) | [📖 목차](00-목차.md) | [다음: 11장. 취소와 진행률 보고 ▶](ch11-취소와진행률보고.md)


9장에서는 `async`/`await`가 왜 등장했는지, 그리고 `await`를 만났을 때 실행 흐름이 어떻게 바뀌는지를 살펴보았습니다. 이번 장에서는 그 흐름을 실제로 가능하게 만드는 컴파일러의 변환 방식(상태 머신), `await` 이후 코드가 어느 스레드에서 재개되는지를 결정하는 `SynchronizationContext`, 그리고 이를 둘러싼 가장 악명 높은 함정인 데드락을 다룹니다. 마지막으로 지금까지 배운 내용을 종합한 실전 예제로 장을 마무리합니다.

## 10.1 컴파일러는 async 메서드를 어떻게 변환하는가: 상태 머신

9.4절에서 "`await` 지점에서 실행 위치와 지역 변수가 저장되었다가 나중에 그 지점부터 재개된다"고 설명했습니다. 이것이 어떻게 가능한지 이해하려면, 컴파일러가 `async` 메서드를 컴파일 시점에 완전히 다른 모양의 코드로 바꿔 쓴다는 사실을 알아야 합니다. 이를 **상태 머신(state machine)** 변환이라고 부릅니다.

다음과 같은 간단한 `async` 메서드가 있다고 합시다.

```csharp
async Task<int> AddAsync(int a, int b)
{
    Console.WriteLine("계산 시작");
    int delayResult = await GetDelayedValueAsync();
    Console.WriteLine("계산 재개");
    return a + b + delayResult;
}
```

컴파일러는 이 메서드를 대략 다음과 같은 형태로(실제 생성 코드는 훨씬 복잡하지만, 개념을 이해하기 위해 단순화한 형태로) 바꿉니다.

```csharp
// 개념을 보여주기 위해 단순화한 의사 코드입니다.
// 실제 컴파일러 출력은 구조체 기반이며 더 많은 최적화를 포함합니다.
struct AddAsyncStateMachine : IAsyncStateMachine
{
    public int a;
    public int b;
    public int state;                       // 현재 어느 await 지점에 있는지
    public TaskAwaiter<int> awaiter;        // 대기 중인 awaiter (재개에 필요한 정보)
    public AsyncTaskMethodBuilder<int> builder; // 반환할 Task<int>를 관리

    public void MoveNext()
    {
        int result;
        switch (state)
        {
            case 0: // 메서드가 처음 시작될 때
                Console.WriteLine("계산 시작");
                awaiter = GetDelayedValueAsync().GetAwaiter();

                if (!awaiter.IsCompleted)
                {
                    state = 1;
                    // 이 awaiter가 완료되면 MoveNext()를 다시 호출하도록 콜백 등록
                    builder.AwaitUnsafeOnCompleted(ref awaiter, ref this);
                    return; // 여기서 호출자에게 제어가 돌아간다
                }
                goto case 1;

            case 1: // await 이후로 재개될 때 진입하는 지점
                int delayResult = awaiter.GetResult();
                Console.WriteLine("계산 재개");
                result = a + b + delayResult;
                builder.SetResult(result); // 반환한 Task<int>를 완료 상태로 만든다
                return;
        }
    }

    // ... SetStateMachine 등 인터페이스의 나머지 멤버
}
```

이 의사 코드에서 짚어야 할 핵심은 다음과 같습니다.

- 메서드의 지역 변수(`a`, `b`, `delayResult`가 필요로 하는 값들)는 지역 스택 변수가 아니라 **상태 머신 구조체의 필드**로 옮겨집니다. 스택 프레임은 메서드가 일시 중단되고 호출자에게 제어가 돌아가는 순간 사라지지만, 필드는 구조체(또는 힙에 박스화된 객체)에 남아 있으므로 나중에 재개될 때도 값을 잃지 않습니다.
- `state` 필드는 "지금 어느 `await` 지점에 있는가"를 기록합니다. `await`가 여러 번 등장하는 메서드라면 `case`가 그만큼 늘어나며, 매번 재개될 때 이 필드 값을 보고 올바른 지점으로 점프합니다.
- `awaiter.IsCompleted`가 `false`라면(즉, 아직 작업이 끝나지 않았다면) `AwaitUnsafeOnCompleted()`로 "이 작업이 끝나면 `MoveNext()`를 다시 호출해 달라"는 콜백을 등록하고 즉시 `return`합니다. 이 `return`이 바로 9장에서 말한 "호출자에게 제어가 돌아가는" 순간입니다.
- 나중에 대상 작업이 완료되어 콜백이 실행되면, 같은 `MoveNext()` 메서드가 **다시 호출**되고, 이번에는 `state` 필드 값(`1`)에 따라 `case 1`로 곧장 진입하여 중단되었던 지점부터 이어서 실행됩니다.
- 메서드 본문이 끝나 `return`에 도달하면 `builder.SetResult()`가 호출되어, 애초에 이 메서드를 호출했을 때 즉시 반환되었던 `Task<int>`가 그제서야 완료 상태로 바뀝니다.

정리하면, `async` 메서드를 호출하는 순간 실제로 일어나는 일은 "아직 완료되지 않은 `Task`를 만들어서 즉시 반환하고, 상태 머신 객체 하나를 준비해서 `MoveNext()`를 최초 한 번 실행하는 것"입니다. 그 이후 `await`를 만날 때마다 실행이 잠시 멈추고, 대상 작업의 완료 콜백이 `MoveNext()`를 다시 호출해 주는 것을 반복하며 메서드 전체가 마치 하나의 흐름처럼 이어지는 것처럼 보이게 만듭니다. `async`/`await`가 "새로운 스레드를 실행하는 마법"이 아니라 "코드를 재작성해서 콜백 체인으로 바꿔주는 컴파일러의 트릭"이라는 사실이 이 구조를 통해 분명해집니다.

## 10.2 SynchronizationContext란 무엇인가

10.1절에서 "`MoveNext()`가 다시 호출된다"고 했는데, 이때 **어느 스레드가** `MoveNext()`를 호출하는지가 매우 중요한 문제입니다. 콘솔 애플리케이션이라면 어느 스레드가 재개하든 크게 상관없는 경우가 많지만, UI 애플리케이션(WPF, Windows Forms, MAUI 등)에서는 사정이 다릅니다.

대부분의 UI 프레임워크는 "UI 컨트롤은 반드시 그 UI를 생성한 스레드에서만 조작할 수 있다"는 규칙을 가지고 있습니다. 만약 `await` 이후의 코드가 스레드 풀의 임의의 스레드에서 재개된다면, 다음과 같은 코드는 크래시하거나 예외를 던질 수 있습니다.

```csharp
private async void OnLoadButtonClick(object sender, EventArgs e)
{
    string content = await File.ReadAllTextAsync("data.txt");

    // await 이후 이 코드가 UI 스레드가 아닌 다른 스레드에서 실행된다면
    // "잘못된 스레드에서 컨트롤에 접근했습니다" 같은 예외가 발생한다.
    textBox.Text = content;
}
```

이 문제를 해결하기 위해 .NET은 **`SynchronizationContext`**라는 추상화를 제공합니다. `SynchronizationContext`는 "작업 하나를 특정 스레드(또는 특정 실행 환경)로 넘겨서 실행시키는 방법"을 캡슐화한 객체입니다. WPF와 Windows Forms는 각각 자신만의 `SynchronizationContext` 구현체(`DispatcherSynchronizationContext`, `WindowsFormsSynchronizationContext`)를 가지고 있으며, 이 구현체는 "넘겨받은 작업을 UI 스레드의 메시지 큐에 넣고, UI 스레드가 그 큐를 순서대로 처리하도록" 만들어져 있습니다.

`await`가 실제로 하는 일 중 하나가 바로 이것입니다. `await`는 메서드를 일시 중단하기 직전에 `SynchronizationContext.Current`를 캡처해 둡니다. 그리고 대상 작업이 완료되어 나머지 코드를 재개해야 할 때, 캡처해 두었던 `SynchronizationContext`가 존재한다면 그 컨텍스트를 통해 나머지 코드를 실행시킵니다. `SynchronizationContext`가 `null`이라면(예를 들어 콘솔 애플리케이션이나 대부분의 스레드 풀 작업 컨텍스트) 그냥 스레드 풀의 임의의 스레드에서 재개됩니다.

정리하면, 앞의 버튼 클릭 예제에서 `await File.ReadAllTextAsync("data.txt")` 이후 `textBox.Text = content;`가 안전하게 실행될 수 있는 이유는, `await`가 자동으로 "원래 있던 UI 스레드의 `SynchronizationContext`로 돌아가서 계속 실행해 달라"고 요청해 주기 때문입니다. 개발자가 `Invoke()`나 `Dispatcher.Invoke()`를 직접 호출하지 않아도 되는 것은 이 메커니즘 덕분입니다.

이 캡처와 복귀 과정은 공짜가 아닙니다. UI 스레드의 메시지 큐에 작업을 넣고 처리되기를 기다리는 과정에는 약간의 오버헤드가 있습니다. 라이브러리 코드처럼 UI로 돌아갈 필요가 전혀 없는 경우에는 이 오버헤드를 피할 방법이 필요한데, 그것이 다음 절의 주제입니다.

## 10.3 ConfigureAwait(false): 컨텍스트로 돌아갈 필요가 없을 때

`Task`와 `Task<T>`는 `ConfigureAwait(bool continueOnCapturedContext)`라는 메서드를 제공합니다. `await someTask.ConfigureAwait(false)`처럼 사용하면, `await`에게 "이 작업이 끝난 뒤 나머지 코드를 실행할 때, 원래의 `SynchronizationContext`(또는 `TaskScheduler`)로 돌아갈 필요 없이 그냥 지금 이 작업을 완료시킨 스레드(대개는 스레드 풀 스레드)에서 곧바로 이어서 실행해 달라"고 지시하는 것입니다.

```csharp
public async Task<string> FetchAndNormalizeAsync(string url)
{
    using HttpClient client = new HttpClient();

    // 이 메서드는 UI 컨트롤을 건드리지 않는 순수한 라이브러리 로직이다.
    // 원래의 컨텍스트로 돌아갈 필요가 없으므로 ConfigureAwait(false)를 사용한다.
    string raw = await client.GetStringAsync(url).ConfigureAwait(false);
    string normalized = raw.Trim().ToLowerInvariant();

    await Task.Delay(10).ConfigureAwait(false);

    return normalized;
}
```

이 메서드가 UI 스레드에서 호출되었다고 하더라도, `ConfigureAwait(false)`를 사용한 각 `await` 이후의 코드는 UI 스레드로 돌아가지 않고 스레드 풀 스레드에서 계속 실행됩니다. 이 메서드 안에는 UI 컨트롤을 다루는 코드가 전혀 없으므로 아무 문제가 없고, 오히려 UI 스레드의 메시지 큐를 거치는 오버헤드를 절약할 수 있습니다.

### 사용 기준

`ConfigureAwait(false)`를 언제 붙여야 하는지는 다음 기준으로 판단합니다.

- **라이브러리(재사용 가능한 클래스 라이브러리) 코드**에서, `await` 이후의 코드가 UI 컨트롤 접근이나 `HttpContext.Current`처럼 특정 컨텍스트에 의존하는 작업을 전혀 하지 않는다면 `ConfigureAwait(false)`를 붙이는 것이 일반적인 권장 사항입니다. 라이브러리는 어떤 종류의 애플리케이션(UI 앱, 콘솔 앱, ASP.NET Core 등)에서 호출될지 알 수 없으므로, 불필요한 컨텍스트 복귀를 피해 두면 성능에 도움이 되고 11장 이후에 다룰 여러 조합 패턴에서도 예측 가능하게 동작합니다.
- **애플리케이션 최상위 코드(이벤트 핸들러, 컨트롤러 액션 등)**에서는 `ConfigureAwait(false)`를 붙이지 않는 것이 일반적입니다. 이런 코드는 애초에 UI 컨트롤을 갱신하거나 요청 컨텍스트 정보를 사용할 가능성이 높기 때문입니다.
- ASP.NET Core는 애초에 요청을 처리할 때 `SynchronizationContext`를 사용하지 않으므로(요청 스레드가 UI 스레드처럼 하나로 고정되어 있지 않습니다), `ConfigureAwait(false)`의 실질적인 효과가 없습니다. 다만 여러 프로젝트 유형에 공유되는 라이브러리 코드라면 습관적으로 붙여 두어도 무해합니다.

`ConfigureAwait(false)`를 붙였다고 해서 코드의 정확성이 갑자기 깨지는 것은 아니지만, 한 메서드 안에서 어떤 `await`는 컨텍스트로 돌아가고 어떤 `await`는 돌아가지 않는 상황이 섞이면 그 다음에 실행되는 코드가 어느 스레드에서 실행될지 예측하기 어려워질 수 있습니다. 그래서 실무에서는 "이 메서드는 라이브러리 로직이니 모든 `await`에 일관되게 `ConfigureAwait(false)`를 붙인다"는 식으로 메서드 단위, 나아가 어셈블리 단위로 일관성을 지키는 것이 좋습니다.

## 10.4 데드락 함정: .Result와 .Wait()의 위험성

`SynchronizationContext`의 존재를 모른 채 `async` 메서드를 "동기적으로" 기다리려고 하면 데드락에 빠질 수 있습니다. 이는 `async`/`await`를 다루면서 가장 많이 보고되는 실전 버그 중 하나입니다. 구체적인 시나리오를 코드로 살펴보겠습니다.

```csharp
// WPF나 Windows Forms처럼 SynchronizationContext가 있는 UI 스레드에서 호출된다고 가정
private void OnLoadButtonClick(object sender, EventArgs e)
{
    // 위험: async 메서드를 .Result로 동기적으로 기다린다
    string content = LoadDataAsync().Result;
    textBox.Text = content;
}

private async Task<string> LoadDataAsync()
{
    await Task.Delay(1000); // 예: 네트워크 호출을 흉내
    return "완료된 데이터";
}
```

이 코드는 UI 스레드에서 실행되면 그대로 멈춰버립니다(데드락). 그 이유를 단계별로 따라가 보겠습니다.

1. `OnLoadButtonClick()`이 UI 스레드에서 실행되며 `LoadDataAsync().Result`를 호출합니다. `.Result`는 `Task<string>`이 완료될 때까지 **현재 스레드를 블록**합니다. 즉, UI 스레드는 여기서 멈춰서 결과를 기다립니다.
2. 한편 `LoadDataAsync()` 내부에서는 `await Task.Delay(1000)`을 만납니다. 이 `await`는 10.2절에서 설명한 대로 현재의 `SynchronizationContext`(UI 스레드의 컨텍스트)를 캡처해 두고, 1초 뒤 타이머가 끝나면 "그 컨텍스트로 돌아가서 나머지 코드(`return "완료된 데이터";`)를 실행해 달라"고 예약한 뒤 제어를 반환합니다.
3. 1초가 지나 `Task.Delay(1000)`이 완료되면, 캡처해 둔 `SynchronizationContext`를 통해 "`return` 문을 실행해 줘"라는 작업이 UI 스레드의 메시지 큐에 들어갑니다. 하지만 UI 스레드는 **1번 단계에서 `.Result`를 기다리며 이미 블록되어 있으므로** 이 메시지 큐를 처리할 수 없습니다.
4. 결과적으로 `LoadDataAsync()`의 나머지 코드는 영원히 실행될 기회를 얻지 못하고, `LoadDataAsync().Result`를 기다리던 UI 스레드도 영원히 풀려나지 못합니다. 서로가 서로를 기다리는 전형적인 데드락입니다.

이 문제는 `.Wait()`를 사용해도 똑같이 발생하며, `SynchronizationContext`가 존재하는 환경(주로 UI 애플리케이션이나 구버전 ASP.NET(.NET Framework)의 요청 컨텍스트)에서 재현됩니다. 반면 콘솔 애플리케이션이나 ASP.NET Core처럼 `SynchronizationContext`가 없는 환경에서는 재개할 때 그냥 스레드 풀의 다른 스레드가 사용되므로 이 특정 데드락은 발생하지 않습니다. 하지만 그렇다고 해서 `.Result`나 `.Wait()`를 안전하게 써도 된다는 뜻은 아닙니다. 환경이 바뀌면 언제든 같은 문제가 재발할 수 있는 잠재적 지뢰이기 때문입니다.

### 해결책: async면 끝까지 async로

가장 확실한 해결책은 애초에 동기적으로 기다리지 않는 것입니다. 이벤트 핸들러 자체를 `async void`로 바꾸어 `await`를 사용하면 문제가 사라집니다.

```csharp
private async void OnLoadButtonClick(object sender, EventArgs e)
{
    // .Result 대신 await를 사용한다.
    // UI 스레드는 여기서 블록되지 않고 즉시 풀려난다.
    string content = await LoadDataAsync();
    textBox.Text = content;
}
```

이제 UI 스레드는 `await LoadDataAsync()`를 만나는 즉시 제어를 돌려받아 다른 이벤트를 처리할 수 있는 상태로 돌아갑니다. `LoadDataAsync()` 내부의 `await Task.Delay(1000)`이 끝나면 캡처해 두었던 `SynchronizationContext`를 통해 UI 스레드의 메시지 큐에 재개 작업이 들어가고, 이번에는 UI 스레드가 블록되어 있지 않으므로 그 메시지 큐를 정상적으로 처리하여 나머지 코드를 실행할 수 있습니다.

이것이 바로 **"async면 끝까지 async로(async all the way)"** 라는 원칙입니다. 한 번 `async` 메서드를 도입했다면, 그 메서드를 호출하는 쪽도 `await`로 기다려야 하고, 그 호출자의 호출자도 마찬가지로 `await`를 사용해야 합니다. 호출 체인의 중간에 `.Result`나 `.Wait()`, `GetAwaiter().GetResult()`처럼 비동기 작업을 동기적으로 기다리는 코드를 끼워 넣는 순간, 데드락의 위험을 항상 안고 가게 됩니다.

부득이하게 동기 코드에서 비동기 메서드의 결과가 꼭 필요한 특수한 상황(예를 들어 `Main` 메서드가 `async Task`를 지원하지 않는 아주 오래된 프로젝트 형식이거나, 완전히 동기적인 인터페이스를 구현해야 하는 경우)이라면 `ConfigureAwait(false)`를 호출 체인 전체에 일관되게 적용하여 `SynchronizationContext`로 돌아가지 않도록 만드는 우회책이 알려져 있습니다. 하지만 이는 데드락의 근본 원인을 피해가는 임시방편일 뿐, 근본적인 해결책은 처음부터 동기적으로 기다리는 코드를 만들지 않는 것입니다. 취소와 진행률 보고를 다루는 11장, 그리고 좋은 비동기 코드 습관을 정리하는 18장에서도 이 원칙이 반복해서 등장할 것입니다.

## 10.5 종합 실전 예제: 파일을 비동기로 읽고 처리하기

지금까지 다룬 내용(그대로 `await`하기, `ConfigureAwait(false)`, 예외 처리, async 끝까지 유지하기)을 모두 반영한 실전 예제를 작성해 보겠습니다. 여러 개의 텍스트 파일을 비동기로 읽어서 각 파일의 줄 수를 세고, 그 결과를 요약 파일에 기록하는 프로그램입니다.

```csharp
using System;
using System.Collections.Generic;
using System.IO;
using System.Threading.Tasks;

class FileWordCountSummarizer
{
    // 라이브러리 성격의 로직: UI 컨텍스트로 돌아갈 필요가 없으므로
    // 모든 await에 ConfigureAwait(false)를 일관되게 적용한다.
    public static async Task<int> CountLinesAsync(string path)
    {
        try
        {
            string content = await File.ReadAllTextAsync(path).ConfigureAwait(false);
            return content.Split('\n').Length;
        }
        catch (IOException ex)
        {
            Console.WriteLine($"'{path}' 읽기 실패: {ex.Message}");
            return 0;
        }
    }

    public static async Task<string> SummarizeAllAsync(IEnumerable<string> paths)
    {
        var lines = new List<string>();

        foreach (string path in paths)
        {
            // I/O 바운드 작업이므로 Task.Run 없이 그대로 await한다.
            int lineCount = await CountLinesAsync(path).ConfigureAwait(false);
            lines.Add($"{Path.GetFileName(path)}: {lineCount}줄");
        }

        return string.Join(Environment.NewLine, lines);
    }

    // 애플리케이션 진입점: 여기서는 컨텍스트를 신경 쓸 필요가 없는
    // 콘솔 애플리케이션이므로 ConfigureAwait(false) 없이 그대로 사용해도 무방하다.
    public static async Task Main(string[] args)
    {
        string[] inputFiles = { "report1.txt", "report2.txt", "report3.txt" };

        Console.WriteLine("파일 분석을 시작합니다...");

        // "async면 끝까지 async로" 원칙에 따라 .Result나 .Wait()를 쓰지 않고
        // Main 메서드 자체를 async Task로 선언해서 끝까지 await로 이어간다.
        string summary = await SummarizeAllAsync(inputFiles);

        await File.WriteAllTextAsync("summary.txt", summary);

        Console.WriteLine("분석이 완료되었습니다. summary.txt를 확인하세요.");
        Console.WriteLine(summary);
    }
}
```

이 예제에서 각 설계 결정이 앞서 배운 원칙과 어떻게 연결되는지 정리하면 다음과 같습니다.

- `CountLinesAsync()`와 `SummarizeAllAsync()`는 재사용 가능한 순수 로직이므로 `ConfigureAwait(false)`를 일관되게 사용했습니다(10.3절).
- 파일 읽기는 전형적인 I/O 바운드 작업이므로 `Task.Run()`으로 감싸지 않고 `File.ReadAllTextAsync()`를 그대로 `await`했습니다(9.5절).
- `CountLinesAsync()` 내부에서 발생할 수 있는 `IOException`은 `try`/`catch`로 잡아서 하나의 파일 실패가 전체 프로그램을 중단시키지 않도록 처리했습니다. `async Task<T>` 메서드이기 때문에 이 예외 처리가 호출자 쪽 `try`/`catch`를 거치지 않고도 메서드 내부에서 자연스럽게 가능합니다(9.3절).
- `Main` 메서드는 `.Result`나 `.Wait()`를 전혀 사용하지 않고 `async Task Main()`으로 선언하여 처음부터 끝까지 `await`로 흐름을 이어갔습니다(10.4절의 "async면 끝까지 async로" 원칙).
- .NET Core/.NET 5 이상에서는 `Main` 메서드가 `async Task`를 반환하도록 선언하는 것이 언어 차원에서 정식으로 지원되므로, 콘솔 애플리케이션에서도 `.Result`나 `.Wait()`로 우회할 필요가 전혀 없습니다.

이처럼 하나의 프로그램 안에서도 "이 코드가 라이브러리 로직인가, 애플리케이션의 진입점인가"에 따라 `ConfigureAwait(false)`의 적용 여부가 달라질 수 있다는 점, 그리고 처음부터 끝까지 비동기 체인을 유지하면 데드락을 걱정할 필요가 전혀 없다는 점을 기억해 두시기 바랍니다.

## 요약

- 컴파일러는 `async` 메서드를 **상태 머신**으로 변환합니다. 지역 변수는 상태 머신 객체의 필드로 옮겨지고, `await` 지점마다 현재 상태를 기록해 두었다가, 대상 작업이 완료되면 그 상태를 보고 중단되었던 지점부터 실행을 재개합니다.
- `SynchronizationContext`는 "특정 스레드(또는 특정 실행 환경)로 작업을 넘겨 실행시키는" 메커니즘입니다. UI 애플리케이션에서 `await` 이후의 코드가 자동으로 UI 스레드로 돌아오는 것은 `await`가 이 컨텍스트를 캡처했다가 복귀시켜 주기 때문입니다.
- `ConfigureAwait(false)`는 원래의 `SynchronizationContext`로 돌아갈 필요가 없을 때 사용하며, 주로 UI 컨트롤이나 요청 컨텍스트에 의존하지 않는 라이브러리 코드에 일관되게 적용하는 것이 좋습니다.
- `SynchronizationContext`가 있는 환경(대표적으로 UI 스레드)에서 `async` 메서드의 결과를 `.Result`나 `.Wait()`로 동기적으로 기다리면, 그 UI 스레드가 블록되어 있는 동안 `await` 이후의 재개 작업이 같은 UI 스레드로 돌아가지 못해 데드락에 빠질 수 있습니다.
- 이 데드락을 피하는 원칙은 "async면 끝까지 async로"입니다. 한 번 `async`/`await`를 도입했다면 호출 체인 전체를 `await`로 이어가고, `.Result`나 `.Wait()`로 도중에 끊지 않아야 합니다.

## 연습문제

1. 상태 머신 변환 관점에서, `await`가 두 번 등장하는 `async` 메서드는 몇 개의 `case`(상태)를 가지게 될지 설명하고, 그 이유를 서술하세요.
2. `SynchronizationContext`가 `null`인 환경(예: 콘솔 애플리케이션)에서는 10.4절에서 설명한 데드락이 왜 발생하지 않는지 설명하세요.
3. 다음 라이브러리 메서드에 `ConfigureAwait(false)`를 적용해야 하는지 판단하고, 이유를 설명하세요.
   ```csharp
   public async Task<byte[]> DownloadImageAsync(string url)
   {
       using var client = new HttpClient();
       return await client.GetByteArrayAsync(url);
   }
   ```
4. 다음 코드는 WPF 애플리케이션의 버튼 클릭 이벤트 핸들러에서 호출될 때 데드락에 빠집니다. 원인을 설명하고 두 가지 이상의 방법으로 수정하세요.
   ```csharp
   private void OnRefreshClick(object sender, RoutedEventArgs e)
   {
       var data = FetchDataAsync().Result;
       listView.ItemsSource = data;
   }
   ```
5. 10.5절의 `SummarizeAllAsync()`는 파일들을 순서대로(하나씩) 읽습니다. 만약 여러 파일을 동시에 읽고 싶다면 어떤 방식으로 바꿀 수 있을지 개략적으로 서술하세요(힌트: 8장에서 다룬 여러 `Task`를 동시에 기다리는 방법을 떠올려 보세요).

---

[◀ 이전: 9장. async/await 기초](ch09-asyncawait기초.md) | [📖 목차](00-목차.md) | [다음: 11장. 취소와 진행률 보고 ▶](ch11-취소와진행률보고.md)
