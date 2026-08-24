# 8장. Task 조합과 예외 처리

[◀ 이전: 7장. Task와 TPL 기초](ch07-Task와TPL기초.md) | [📖 목차](00-목차.md) | [다음: 9장. async/await 기초 ▶](ch09-asyncawait기초.md)


7장에서는 `Task.Run()`으로 작업 하나를 시작하고, `Wait()`나 `Result`로 그 결과를 동기적으로 기다리는 방법을 살펴봤습니다. 하지만 실제 애플리케이션에서는 작업 하나로 끝나는 경우가 드뭅니다. 여러 개의 파일을 동시에 다운로드하고 모두 끝나기를 기다려야 하거나, 여러 서버에 같은 요청을 보내고 가장 먼저 응답하는 서버의 결과만 사용해야 하는 경우도 있습니다. 이번 장에서는 여러 `Task`를 조합하는 방법과, 그 과정에서 발생하는 예외를 다루는 방법을 알아봅니다.

## 8.1 ContinueWith()로 작업 체이닝하기

작업 A가 끝난 뒤에 그 결과를 이어받아 작업 B를 실행하고 싶다면 어떻게 해야 할까요? `Wait()`로 A를 기다린 다음 B를 시작하는 방법도 있지만, 이는 호출 스레드를 블로킹시킨다는 7장의 문제를 그대로 안고 갑니다. `Task`는 이런 상황을 위해 `ContinueWith()`라는 메서드를 제공합니다. `ContinueWith()`는 대상 `Task`가 완료된 직후 실행할 후속 작업을 등록하며, 호출 스레드를 블로킹하지 않습니다.

```csharp
using System;
using System.Threading.Tasks;

Task<int> firstTask = Task.Run(() =>
{
    Console.WriteLine("첫 번째 작업 실행 중...");
    return 10;
});

Task<int> continuationTask = firstTask.ContinueWith(antecedent =>
{
    // antecedent는 firstTask 자신을 가리킵니다.
    int previousResult = antecedent.Result;
    Console.WriteLine($"이전 결과 {previousResult}를 이어받아 계산합니다.");
    return previousResult * 2;
});

Console.WriteLine($"최종 결과: {continuationTask.Result}"); // 20
```

`ContinueWith()`에 전달하는 대리자는 완료된 `Task` 자신(위 예제의 `antecedent`)을 매개변수로 받습니다. 이를 통해 이전 작업의 결과값(`Result`)이나 상태(`IsFaulted`, `IsCanceled`)에 접근할 수 있습니다. `ContinueWith()`는 다시 새로운 `Task`를 반환하므로, 후속 작업을 계속 이어 붙이는 체인을 만들 수 있습니다.

```csharp
Task<string> pipeline = Task.Run(() => 5)
    .ContinueWith(t => t.Result + 10)
    .ContinueWith(t => t.Result * 3)
    .ContinueWith(t => $"최종 값: {t.Result}");

Console.WriteLine(pipeline.Result); // 최종 값: 45
```

`ContinueWith()`에는 `TaskContinuationOptions`를 지정해 후속 작업이 실행되는 조건을 세밀하게 제어하는 기능도 있습니다. 예를 들어 `OnlyOnFaulted`를 지정하면 이전 작업이 예외로 실패했을 때만 실행되는 후속 작업을, `OnlyOnRanToCompletion`을 지정하면 이전 작업이 정상적으로 끝났을 때만 실행되는 후속 작업을 등록할 수 있습니다.

```csharp
Task riskyTask = Task.Run(() => throw new InvalidOperationException("문제 발생"));

Task errorHandler = riskyTask.ContinueWith(
    t => Console.WriteLine($"오류 처리: {t.Exception?.InnerException?.Message}"),
    TaskContinuationOptions.OnlyOnFaulted);

Task successHandler = riskyTask.ContinueWith(
    t => Console.WriteLine("성공적으로 끝났습니다."),
    TaskContinuationOptions.OnlyOnRanToCompletion);

errorHandler.Wait();
```

### ContinueWith()의 한계

`ContinueWith()`는 분명 유용하지만, 체인이 길어질수록 코드의 가독성이 급격히 떨어집니다. 위의 파이프라인 예제처럼 세 단계만 이어 붙여도 각 단계마다 `t.Result`를 꺼내는 코드가 반복되고, 조건 분기나 예외 처리가 섞이기 시작하면 코드는 금세 읽기 힘든 형태가 됩니다. 실무에서는 종종 이런 방식을 "콜백 피라미드"라고 부르는데, 각 단계의 로직이 중첩된 람다식 안에 파묻혀서 전체 흐름을 한눈에 파악하기 어려워지기 때문입니다.

또한 `ContinueWith()`는 기본적으로 어떤 스레드에서 후속 작업이 실행될지에 대한 보장이 명확하지 않고(옵션으로 조정은 가능하지만), 예외 처리도 `t.Exception`을 직접 검사해야 하는 등 번거로운 부분이 많습니다.

9장에서 배울 `async`/`await` 구문은 바로 이 문제를 해결하기 위해 등장합니다. 위의 파이프라인 예제를 `await`로 다시 쓰면 다음과 같은 모습이 됩니다. (아직 문법을 배우지 않았으니 참고로만 봐 두세요.)

```csharp
// 9장 예고: async/await를 사용하면 아래처럼 순차적인 코드로 표현할 수 있습니다.
async Task<string> RunPipelineAsync()
{
    int step1 = await Task.Run(() => 5);
    int step2 = step1 + 10;
    int step3 = step2 * 3;
    return $"최종 값: {step3}";
}
```

체인 형태의 `ContinueWith()`보다 위에서 아래로 읽히는 이 코드가 훨씬 이해하기 쉽습니다. `ContinueWith()`를 배우는 이유는 이 방식을 실무에서 널리 쓰기 위해서라기보다, `await`가 내부적으로 해결해 주는 문제가 무엇인지 체감하기 위해서라고 볼 수 있습니다.

## 8.2 여러 작업 기다리기: WhenAll과 WhenAny

여러 개의 독립적인 작업을 동시에 시작하고 그 결과를 조합해야 하는 상황은 매우 흔합니다. TPL은 이를 위해 `Task.WhenAll()`과 `Task.WhenAny()`라는 두 정적 메서드를 제공합니다.

### Task.WhenAll()

`Task.WhenAll()`은 전달받은 모든 `Task`가 완료될 때까지 기다리는 새로운 `Task`를 반환합니다. 결과가 있는 `Task<TResult>`들을 전달하면, 각 작업의 결과를 원래 순서대로 담은 배열을 반환하는 `Task<TResult[]>`를 얻을 수 있습니다.

```csharp
using System;
using System.Threading.Tasks;

Task<int> DownloadSizeAsync(string url)
{
    return Task.Run(() =>
    {
        System.Threading.Thread.Sleep(500); // 다운로드를 흉내내는 지연
        return url.Length * 100; // 예시용 임의 계산
    });
}

Task<int> task1 = DownloadSizeAsync("https://example.com/a");
Task<int> task2 = DownloadSizeAsync("https://example.com/bb");
Task<int> task3 = DownloadSizeAsync("https://example.com/ccc");

Task<int[]> allTask = Task.WhenAll(task1, task2, task3);
int[] sizes = allTask.Result; // 세 작업이 모두 끝나야 반환됩니다.

Console.WriteLine($"합계: {sizes[0] + sizes[1] + sizes[2]}");
```

여기서 눈여겨볼 점은 `task1`, `task2`, `task3`가 각각 `Task.Run()`을 호출하는 시점에 이미 동시에(concurrently) 시작된다는 것입니다. `Task.WhenAll()`은 이미 진행 중인 작업들을 "묶어서 기다리는" 역할만 할 뿐, 작업을 순차적으로 실행시키지 않습니다. 만약 세 작업이 각각 500밀리초씩 걸린다면, 순차적으로 실행했을 때는 총 1500밀리초가 걸리지만 `Task.WhenAll()`로 묶어서 기다리면 (스레드 풀에 여유가 있다는 전제하에) 약 500밀리초 만에 세 작업 모두 끝납니다.

`Task.WaitAll()`이라는 정적 메서드도 있는데, 이는 `Task.WhenAll()`과 달리 `Task`를 반환하지 않고 호출 스레드를 직접 블로킹시켜 모든 작업이 끝날 때까지 기다립니다. `Task.WhenAll()`은 그 자체가 또 다른 `Task`이므로 `await`와 함께 사용하기에 적합하고, `Task.WaitAll()`은 7장에서 설명한 `Wait()`와 같은 블로킹 방식의 위험성을 그대로 갖고 있습니다. 이 책에서는 `await Task.WhenAll(...)` 형태(9장 이후)를 기본으로 권장합니다.

### Task.WhenAny()

`Task.WhenAny()`는 전달받은 여러 작업 중 가장 먼저 완료되는 작업 하나가 끝나는 순간 완료되는 `Task<Task>`(또는 `Task<Task<TResult>>`)를 반환합니다. 나머지 작업들은 계속 백그라운드에서 실행되며, 자동으로 취소되지는 않습니다.

```csharp
using System;
using System.Threading.Tasks;

Task<string> FetchFromMirrorAsync(string mirrorName, int delayMs)
{
    return Task.Run(() =>
    {
        System.Threading.Thread.Sleep(delayMs);
        return $"{mirrorName}에서 응답 받음";
    });
}

Task<string> mirrorA = FetchFromMirrorAsync("미러 A", 800);
Task<string> mirrorB = FetchFromMirrorAsync("미러 B", 300);
Task<string> mirrorC = FetchFromMirrorAsync("미러 C", 1200);

Task<Task<string>> firstCompleted = Task.WhenAny(mirrorA, mirrorB, mirrorC);
Task<string> winner = firstCompleted.Result;

Console.WriteLine(winner.Result); // "미러 B에서 응답 받음"이 출력될 가능성이 높습니다.
```

`Task.WhenAny()`는 여러 개의 미러 서버 중 가장 빠른 응답을 사용하는 경쟁(racing) 패턴이나, 실제 작업과 타임아웃을 나타내는 `Task.Delay()`를 함께 넘겨서 제한 시간 안에 끝나지 않으면 포기하는 타임아웃 패턴을 구현할 때 자주 사용됩니다.

```csharp
Task<string> slowOperation = Task.Run(() =>
{
    System.Threading.Thread.Sleep(3000);
    return "느린 작업의 결과";
});

Task timeoutTask = Task.Delay(TimeSpan.FromSeconds(1));

Task winnerTask = Task.WhenAny(slowOperation, timeoutTask).Result;

if (winnerTask == timeoutTask)
{
    Console.WriteLine("제한 시간을 초과했습니다.");
}
else
{
    Console.WriteLine($"작업 완료: {slowOperation.Result}");
}
```

이 타임아웃 패턴은 실무에서 매우 유용하지만 한 가지 주의할 점이 있습니다. `Task.WhenAny()`가 완료되어도 "패배한" 작업(`slowOperation`)은 자동으로 취소되지 않고 백그라운드에서 계속 실행됩니다. 진짜로 작업을 중단시키려면 `CancellationToken`을 함께 사용해야 하며, 이 내용은 11장에서 다룹니다.

## 8.3 AggregateException과 예외 처리

`Task` 안에서 실행되는 코드가 예외를 던지면 어떻게 될까요? 7장에서 잠깐 언급했듯이, `Task`가 실패했을 때 `Wait()`나 `Result`를 통해 그 사실을 확인하려고 하면 원본 예외가 아니라 `AggregateException`이라는 래퍼 예외가 던져집니다.

### 왜 하나로 감싸는가

이렇게 감싸는 이유를 이해하려면 `Task.WhenAll()`처럼 여러 작업을 하나로 묶는 상황을 생각해 보면 됩니다. 세 개의 작업을 `Task.WhenAll()`로 묶었는데 그중 두 개가 서로 다른 예외를 던지며 실패했다면, 그 결과를 기다리는 쪽에는 예외가 몇 개나, 어떤 순서로 전달되어야 할까요? .NET의 예외 처리 모델은 하나의 `try`/`catch` 블록에서 하나의 예외 객체만 잡는 것을 전제로 설계되어 있습니다. 여러 개의 예외를 하나의 자리에 담아 전달하려면 그것들을 모두 품을 수 있는 컨테이너가 필요한데, 그 컨테이너 역할을 하는 것이 바로 `AggregateException`입니다.

`AggregateException`은 `InnerExceptions`라는 `ReadOnlyCollection<Exception>` 프로퍼티를 가지고 있어서, 실패의 원인이 된 모든 예외를 순서대로 담고 있습니다. 단일 `Task` 하나만 실패한 경우에도 마찬가지로 `AggregateException`으로 감싸지며, 이때는 `InnerExceptions`에 원소가 하나뿐인 상태가 됩니다. `InnerException` 프로퍼티(단수형)는 `InnerExceptions[0]`에 대한 편의 접근자입니다.

```csharp
using System;
using System.Threading.Tasks;

Task task1 = Task.Run(() => throw new InvalidOperationException("작업 1 실패"));
Task task2 = Task.Run(() => throw new ArgumentException("작업 2 실패"));
Task task3 = Task.Run(() => 42); // 이 작업은 성공합니다.

try
{
    Task.WaitAll(task1, task2, task3);
}
catch (AggregateException ex)
{
    Console.WriteLine($"총 {ex.InnerExceptions.Count}개의 예외가 발생했습니다.");
    foreach (Exception inner in ex.InnerExceptions)
    {
        Console.WriteLine($" - {inner.GetType().Name}: {inner.Message}");
    }
}
```

### 중첩된 AggregateException 풀기: Flatten()

`ContinueWith()`를 여러 단계로 체이닝하거나, `Task` 안에서 또 다른 `Task`의 결과를 기다리는 구조를 만들다 보면 `AggregateException` 안에 또 다른 `AggregateException`이 들어 있는 중첩 구조가 만들어질 수 있습니다. 이런 중첩 구조를 순회하면서 실제 원인이 되는 예외를 하나씩 찾아내는 일은 번거롭습니다. `AggregateException`은 이를 위해 `Flatten()` 메서드를 제공합니다. `Flatten()`은 중첩된 모든 `AggregateException`을 재귀적으로 풀어헤쳐서, 실제 원인이 되는 예외들만 담긴 새로운 `AggregateException`을 반환합니다.

```csharp
using System;
using System.Threading.Tasks;

Task outerTask = Task.Run(() =>
{
    Task innerTask1 = Task.Run(() => throw new InvalidOperationException("내부 오류 1"));
    Task innerTask2 = Task.Run(() => throw new ArgumentException("내부 오류 2"));
    Task.WaitAll(innerTask1, innerTask2); // 여기서 AggregateException이 발생
});

try
{
    outerTask.Wait(); // 바깥쪽에서 또 한 번 AggregateException으로 감싸짐
}
catch (AggregateException ex)
{
    AggregateException flattened = ex.Flatten();
    Console.WriteLine($"평탄화 후 예외 개수: {flattened.InnerExceptions.Count}");
    foreach (Exception inner in flattened.InnerExceptions)
    {
        Console.WriteLine($" - {inner.GetType().Name}: {inner.Message}");
    }
}
```

`Flatten()`을 사용하면 중첩 깊이와 상관없이 실제 예외들을 한 단계의 컬렉션으로 다룰 수 있어서, 예외 종류에 따라 분기 처리하는 코드를 훨씬 단순하게 작성할 수 있습니다. `AggregateException`에는 `Handle()`이라는 메서드도 있는데, 각 내부 예외를 처리했는지 여부를 `bool`로 반환하는 콜백을 전달하면 처리되지 않은 예외만 다시 `AggregateException`으로 던져줍니다.

```csharp
try
{
    outerTask.Wait();
}
catch (AggregateException ex)
{
    ex.Flatten().Handle(inner =>
    {
        if (inner is InvalidOperationException)
        {
            Console.WriteLine($"InvalidOperationException 처리됨: {inner.Message}");
            return true; // 이 예외는 처리했으니 다시 던지지 않습니다.
        }
        return false; // 처리하지 못한 예외는 다시 AggregateException에 담겨 던져집니다.
    });
}
```

### await를 사용하면 달라지는 점 (9장 예고)

지금까지 살펴본 `AggregateException`은 `Wait()`나 `Result`처럼 동기적으로 결과를 꺼낼 때 나타나는 동작입니다. 9장에서 배울 `await` 키워드를 사용하면 이야기가 조금 달라집니다. `await`로 실패한 `Task`를 기다리면, `AggregateException`으로 감싸지지 않고 `InnerExceptions`의 **첫 번째** 예외가 원본 타입 그대로 던져집니다. 즉 `InvalidOperationException`이 발생했다면 `catch (InvalidOperationException)`으로 바로 잡을 수 있다는 뜻입니다.

```csharp
// 9장 예고: await는 AggregateException이 아니라 원본 예외 타입을 그대로 전파합니다.
async Task RunAsync()
{
    try
    {
        await Task.Run(() => throw new InvalidOperationException("작업 실패"));
    }
    catch (InvalidOperationException ex)
    {
        // AggregateException이 아니라 InvalidOperationException을 바로 잡습니다.
        Console.WriteLine($"예외 처리: {ex.Message}");
    }
}
```

다만 `Task.WhenAll()`로 여러 작업을 묶은 경우에는 주의가 필요합니다. `await Task.WhenAll(...)`도 첫 번째 예외만 원본 타입으로 던지기 때문에, 두 번째 이후의 예외 정보는 `catch` 블록의 예외 객체만으로는 알 수 없습니다. 나머지 예외까지 모두 확인하고 싶다면, `await`가 예외를 던지기 전에 `Task.WhenAll()`이 반환한 `Task` 객체 자체의 `Exception` 프로퍼티를 조회해야 합니다. 이 프로퍼티는 여전히 `AggregateException` 타입이며, 그 안에 실패한 모든 작업의 예외가 담겨 있습니다. 이 내용은 9장에서 `async`/`await` 문법을 배운 뒤 다시 자세히 다룹니다.

## 8.4 여러 작업 중 하나라도 실패했을 때의 처리 패턴

여러 작업을 동시에 실행하다 보면 "일부는 성공하고 일부는 실패하는" 상황을 자주 마주치게 됩니다. 이런 상황을 다루는 몇 가지 일반적인 패턴을 살펴보겠습니다.

### 패턴 1: 실패해도 계속 진행하고, 끝난 뒤 결과를 취합하기

각 작업이 독립적이어서 하나가 실패해도 나머지 작업의 결과는 그대로 활용하고 싶다면, 각 작업의 예외를 개별적으로 처리하도록 만들 수 있습니다.

```csharp
using System;
using System.Collections.Generic;
using System.Threading.Tasks;

Task<int> ProcessItemAsync(int id)
{
    return Task.Run(() =>
    {
        if (id == 3)
        {
            throw new InvalidOperationException($"항목 {id} 처리 실패");
        }
        return id * id;
    });
}

List<Task<int>> tasks = new();
for (int i = 1; i <= 5; i++)
{
    tasks.Add(ProcessItemAsync(i));
}

Task<int[]> allTask = Task.WhenAll(tasks);

try
{
    int[] results = allTask.Result;
    Console.WriteLine("모든 항목이 성공했습니다.");
}
catch (AggregateException)
{
    // Task.WhenAll이 반환한 Task의 Exception 프로퍼티에서 전체 실패 목록을 확인합니다.
    Console.WriteLine("일부 항목이 실패했습니다:");
    foreach (Exception ex in allTask.Exception!.Flatten().InnerExceptions)
    {
        Console.WriteLine($" - {ex.Message}");
    }

    Console.WriteLine("성공한 항목들:");
    foreach (Task<int> task in tasks)
    {
        if (task.IsCompletedSuccessfully)
        {
            Console.WriteLine($" - 결과: {task.Result}");
        }
    }
}
```

이 패턴에서는 `Task.WhenAll()`이 예외를 던지더라도 개별 `Task` 목록(`tasks`)을 그대로 유지하고 있으므로, 각 작업의 `IsCompletedSuccessfully`나 `IsFaulted`를 확인해 성공한 것과 실패한 것을 나눠서 처리할 수 있습니다.

### 패턴 2: 하나라도 실패하면 즉시 전체를 실패로 간주하기

반대로, 여러 작업이 하나의 트랜잭션처럼 묶여 있어서 하나라도 실패하면 전체를 실패로 처리해야 하는 경우도 있습니다. 이때는 `Task.WhenAll()`이 실패했다는 사실 자체가 곧 "요청이 실패했다"는 신호가 되므로, 앞서 살펴본 것처럼 `AggregateException`을 잡아서 상위 계층에 적절한 예외나 결과를 전달하면 됩니다.

```csharp
async Task<bool> SaveAllOrNothingAsync(IEnumerable<int> ids)
{
    // 9장 예고: 실무 코드에서는 이렇게 async/await로 작성하는 것이 일반적입니다.
    var tasks = new List<Task>();
    foreach (int id in ids)
    {
        tasks.Add(ProcessItemAsync(id));
    }

    try
    {
        await Task.WhenAll(tasks);
        return true;
    }
    catch
    {
        // 하나라도 실패했다면 전체를 실패로 간주하고, 필요하다면 이미 성공한 항목을 되돌립니다.
        return false;
    }
}
```

### 패턴 3: WhenAny로 첫 실패 또는 첫 성공에 반응하기

여러 작업 중 어느 하나가 먼저 끝나는 순간 즉시 반응해야 하는 경우에는 `Task.WhenAny()`를 반복적으로 사용하는 방법도 있습니다. 완료된 작업을 목록에서 제거하고 나머지에 대해 다시 `Task.WhenAny()`를 호출하는 방식으로, 작업들이 끝나는 순서대로 하나씩 처리할 수 있습니다.

```csharp
List<Task<int>> remaining = new(tasks);

while (remaining.Count > 0)
{
    Task<int> completed = await Task.WhenAny(remaining);
    remaining.Remove(completed);

    if (completed.IsFaulted)
    {
        Console.WriteLine($"작업 실패: {completed.Exception!.Flatten().InnerException?.Message}");
    }
    else
    {
        Console.WriteLine($"작업 성공, 결과: {completed.Result}");
    }
}
```

이 패턴은 완료되는 순서대로 결과를 즉시 화면에 표시하거나 로그를 남겨야 하는 상황, 예를 들어 여러 파일을 병렬로 다운로드하면서 각 파일이 끝날 때마다 진행률을 갱신해야 하는 경우에 유용합니다. 진행률 보고에 대한 더 체계적인 방법은 11장에서 `IProgress<T>`와 함께 다룹니다.

## 요약

- `ContinueWith()`는 `Task`가 완료된 뒤 실행할 후속 작업을 등록하는 방법이지만, 체인이 길어질수록 가독성이 떨어집니다. 9장의 `async`/`await`는 이 문제를 순차적인 코드 형태로 해결합니다.
- `Task.WhenAll()`은 여러 작업이 모두 끝나기를 기다리고, `Task.WhenAny()`는 가장 먼저 끝나는 작업 하나를 알려줍니다. 두 메서드 모두 그 자체가 `Task`이므로 블로킹 없이 조합할 수 있습니다.
- `Task`에서 발생한 예외는 여러 작업의 예외를 하나로 모아 전달할 수 있도록 `AggregateException`으로 감싸집니다. `InnerExceptions`로 전체 목록을, `Flatten()`으로 중첩 구조를 풀어낸 목록을 얻을 수 있습니다.
- `Wait()`/`Result`는 예외를 `AggregateException`으로 던지지만, `await`(9장)는 원본 예외 타입을 그대로 전파합니다. 다만 `Task.WhenAll()`을 `await`할 때는 첫 번째 예외만 전파되므로, 전체 실패 목록이 필요하면 `Task`의 `Exception` 프로퍼티를 별도로 확인해야 합니다.
- 여러 작업 중 일부가 실패하는 상황은 "실패해도 계속 진행", "하나라도 실패하면 전체 실패", "완료되는 순서대로 처리" 같은 패턴으로 대응할 수 있습니다.

## 연습문제

1. `ContinueWith()`를 네 단계 이상 이어 붙인 파이프라인을 작성하고, 중간 단계에서 예외가 발생했을 때 `TaskContinuationOptions.OnlyOnFaulted`로 등록한 오류 처리 후속 작업이 실행되는지 확인해 보세요.
2. 서로 다른 지연 시간을 가진 세 개의 `Task<int>`를 `Task.WhenAll()`로 묶어 실행하고, 전체 소요 시간이 가장 긴 작업의 시간과 비슷한지 측정해 보세요. (`Stopwatch` 클래스를 활용하세요.)
3. `Task.WhenAny()`와 `Task.Delay()`를 조합해 3초 타임아웃을 구현하고, 실제 작업이 타임아웃보다 느릴 때와 빠를 때 각각 어떤 메시지가 출력되는지 확인해 보세요.
4. 세 개의 작업 중 두 개가 서로 다른 예외를 던지도록 만들고, `Task.WaitAll()`로 기다렸을 때 `AggregateException.InnerExceptions`에 몇 개의 예외가 담기는지, 그리고 `Flatten()`을 호출했을 때와 호출하지 않았을 때 결과가 어떻게 다른지 비교해 보세요.
5. 8.4절의 "패턴 1"을 참고해, 10개의 항목을 처리하는 작업 목록을 만들고 그중 3개가 무작위로 실패하도록 구현해 보세요. 실패한 항목의 메시지와 성공한 항목의 결과를 모두 출력하는 코드를 작성해 보세요.

---

[◀ 이전: 7장. Task와 TPL 기초](ch07-Task와TPL기초.md) | [📖 목차](00-목차.md) | [다음: 9장. async/await 기초 ▶](ch09-asyncawait기초.md)
