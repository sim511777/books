# 13장. PLINQ

[◀ 이전: 12장. 병렬 프로그래밍(Parallel 클래스)](ch12-병렬프로그래밍Parallel클래스.md) | [📖 목차](00-목차.md) | [다음: 14장. 동시성 컬렉션 ▶](ch14-동시성컬렉션.md)


## 들어가며

12장에서는 `Parallel.For`와 `Parallel.ForEach`를 사용해 명령형 반복문을 병렬화하는 방법을 살펴봤습니다. 그런데 실무에서 데이터를 다룰 때는 `for`문보다 LINQ 쿼리를 훨씬 자주 사용합니다. 컬렉션을 필터링하고, 변환하고, 집계하는 코드는

```csharp
var result = orders
    .Where(o => o.Amount > 1000)
    .Select(o => Compute(o))
    .OrderBy(o => o)
    .ToList();
```

같은 형태로 작성되는 경우가 많습니다. 이런 LINQ 쿼리를 병렬로 실행하고 싶다면 어떻게 해야 할까요? 매번 `Parallel.ForEach`로 풀어 쓰는 것은 번거롭고 코드의 선언적 성격도 잃어버립니다.

**PLINQ(Parallel LINQ)**는 이 문제를 해결하기 위해 .NET이 제공하는, LINQ 쿼리를 위한 병렬 실행 엔진입니다. `System.Linq.ParallelEnumerable`에 정의된 확장 메서드들로 구성되며, 기존 LINQ 쿼리에 `AsParallel()` 호출 하나만 끼워 넣으면 쿼리 파이프라인 전체가 여러 스레드에 나뉘어 실행됩니다.

12장의 `Parallel` 클래스와 13장의 PLINQ는 같은 스레드 풀 기반 작업 분할 인프라 위에 서 있지만, 사용하는 계층이 다릅니다.

| | 대상 | 스타일 |
|---|---|---|
| `Parallel` 클래스 (12장) | `for`/`foreach` 반복문 | 명령형(imperative) |
| PLINQ (13장) | LINQ 쿼리 | 선언적(declarative) |

명령형 반복문을 이미 가지고 있다면 `Parallel.For`/`Parallel.ForEach`가 자연스럽고, LINQ 쿼리 체인으로 데이터를 가공하고 있다면 PLINQ가 자연스럽습니다. 내부적으로 PLINQ도 결국 여러 스레드에 데이터를 분할하고, 각 스레드가 부분 결과를 계산한 뒤 이를 병합하는 방식으로 동작한다는 점에서 근본 원리는 동일합니다.

## AsParallel로 쿼리를 병렬화하기

일반적인 LINQ 쿼리는 `IEnumerable<T>`를 기반으로 순차적으로 실행됩니다. 여기에 `AsParallel()`을 호출하면 소스가 `ParallelQuery<T>`로 감싸지고, 이후 이어지는 `Where`, `Select`, `OrderBy` 등의 연산자들은 `Enumerable`이 아니라 `ParallelEnumerable`에 정의된 병렬 버전으로 바인딩됩니다.

```csharp
using System.Linq;

int[] numbers = Enumerable.Range(1, 20_000_000).ToArray();

// 순차 LINQ
var sequentialResult = numbers
    .Where(n => IsPrime(n))
    .Select(n => n * n)
    .ToArray();

// PLINQ
var parallelResult = numbers
    .AsParallel()
    .Where(n => IsPrime(n))
    .Select(n => n * n)
    .ToArray();

static bool IsPrime(int n)
{
    if (n < 2) return false;
    for (int i = 2; (long)i * i <= n; i++)
    {
        if (n % i == 0) return false;
    }
    return true;
}
```

`AsParallel()`을 호출하는 순간부터 PLINQ는 소스 컬렉션을 여러 청크(chunk)로 분할하고, 스레드 풀의 여러 워커 스레드에 각 청크를 배분합니다. 각 스레드는 `Where`와 `Select`를 자신이 맡은 청크에 대해 독립적으로 실행하고, 마지막에 `ToArray()`가 이 부분 결과들을 하나로 병합합니다.

여기서 중요한 전제 조건이 있습니다. `Where`나 `Select`에 전달하는 람다식은 **부수 효과(side effect)가 없고, 다른 원소의 처리와 독립적**이어야 합니다. 만약 람다 내부에서 공유 변수를 수정한다면 3장에서 다룬 경쟁 조건(race condition)이 그대로 재현됩니다.

```csharp
// 위험한 코드 - 여러 스레드가 동시에 total을 수정한다
int total = 0;
numbers.AsParallel().ForAll(n => total += n); // 경쟁 조건 발생

// 올바른 코드 - PLINQ의 집계 기능을 사용한다
long total = numbers.AsParallel().Sum(n => (long)n);
```

`Sum`, `Count`, `Aggregate` 같은 집계 연산자는 PLINQ 내부에서 스레드별 부분 합계를 안전하게 계산한 뒤 병합하도록 구현되어 있으므로, 직접 공유 변수를 갱신하는 대신 이런 내장 연산자를 사용하는 것이 안전하고 간결합니다.

### AsSequential로 되돌리기

쿼리 체인의 일부만 병렬로 실행하고 나머지는 순차적으로 처리하고 싶다면 `AsSequential()`로 다시 일반 `IEnumerable<T>`로 전환할 수 있습니다.

```csharp
var result = numbers
    .AsParallel()
    .Where(n => IsPrime(n))     // 병렬로 실행
    .AsSequential()
    .Select(n => LogAndTransform(n)) // 이후는 순차로 실행 (예: 로그 순서 보장)
    .ToArray();
```

## AsOrdered로 순서 유지하기

PLINQ의 기본 동작에서 결과 순서는 **보장되지 않습니다**. `numbers`가 1부터 순서대로 정렬되어 있었더라도, `AsParallel().Select(...).ToArray()`의 결과는 원본 순서와 다를 수 있습니다.

```csharp
var numbers = Enumerable.Range(1, 20);

var result = numbers.AsParallel().Select(n => n).ToArray();
// 예: 5, 1, 8, 2, 9, 3, ... 처럼 뒤섞여 나올 수 있다
```

이유는 병렬 실행의 본질에 있습니다. 원본 시퀀스를 여러 청크로 나누어 서로 다른 스레드에 배분하면, 각 스레드가 자신의 청크를 처리하는 속도는 스레드 스케줄링, CPU 코어 상황, 각 원소의 계산 비용에 따라 달라집니다. 먼저 끝난 스레드의 결과가 먼저 결과 스트림에 합류하기 때문에, 원본의 순서 정보를 스레드들이 굳이 유지하려 하지 않는 한 순서는 뒤섞일 수밖에 없습니다.

PLINQ가 기본적으로 순서를 포기하는 이유는 **성능** 때문입니다. 순서를 유지하려면 각 원소가 원본에서 몇 번째 위치였는지 추적하고, 병합 시점에 그 위치대로 재조립하는 추가 작업이 필요합니다. 순서가 중요하지 않은 대다수의 집계·필터링 작업에서 이 비용은 불필요한 오버헤드입니다.

원본 순서를 유지해야 한다면 `AsOrdered()`를 명시적으로 호출합니다.

```csharp
var result = numbers
    .AsParallel()
    .AsOrdered()
    .Where(n => IsPrime(n))
    .Select(n => n * n)
    .ToArray();
// 항상 원본 순서(2, 3, 5, 7, 11, ...)를 유지한 채 정렬되어 나온다
```

`AsOrdered()`는 계산 자체는 여전히 병렬로 수행하되, 결과를 병합할 때 원본 인덱스를 기준으로 재정렬합니다. 병렬성은 유지하면서 순서 보장 비용만 추가로 지불하는 셈입니다. 다시 순서를 신경 쓰지 않아도 되는 지점부터는 `AsUnordered()`를 호출해 오버헤드를 줄일 수 있습니다.

```csharp
var result = numbers
    .AsParallel()
    .AsOrdered()
    .Where(n => IsPrime(n))   // 여기까지는 순서가 필요
    .AsUnordered()
    .Select(n => Compute(n))  // 이후 순서는 상관없음 - 오버헤드 제거
    .ToArray();
```

> **주의**: `OrderBy()`는 `AsOrdered()`와 다른 개념입니다. `OrderBy()`는 값 자체를 기준으로 정렬하는 연산자이고, `AsOrdered()`는 원본 시퀀스에 존재하던 순서를 결과까지 그대로 전달하겠다는 선언입니다. `OrderBy()`를 쓰면 그 결과는 항상 정렬 기준에 따라 순서가 있는 상태가 되므로 별도로 `AsOrdered()`를 붙일 필요가 없습니다.

## WithDegreeOfParallelism으로 병렬도 제한하기

기본적으로 PLINQ는 `Environment.ProcessorCount`, 즉 시스템의 논리 프로세서 수만큼 병렬도를 사용합니다. 하지만 다음과 같은 상황에서는 병렬도를 명시적으로 제한하고 싶을 수 있습니다.

- 같은 프로세스에서 다른 CPU 집약적 작업도 동시에 실행 중이라 PLINQ가 모든 코어를 독차지하면 안 되는 경우
- 외부 리소스(예: 제한된 수의 DB 커넥션, 외부 API 동시 호출 제한)와 연동되어 있어 동시 실행 개수 자체를 제어해야 하는 경우
- 성능 테스트를 위해 병렬도를 고정하고 싶은 경우

```csharp
var result = numbers
    .AsParallel()
    .WithDegreeOfParallelism(4) // 최대 4개의 스레드만 사용
    .Where(n => IsPrime(n))
    .ToArray();
```

`WithDegreeOfParallelism()`에 전달할 수 있는 값의 범위는 1부터 512까지입니다. 0 이하이거나 512를 초과하는 값을 전달하면 `ArgumentOutOfRangeException`이 발생합니다.

`WithDegreeOfParallelism(1)`을 지정하면 사실상 순차 실행과 비슷한 효과를 내지만, PLINQ의 분할·병합 오버헤드는 여전히 남아 있으므로 완전히 같지는 않습니다. 순차 실행이 필요하다면 `AsSequential()`을 쓰는 편이 낫습니다.

이 값을 얼마로 설정할지는 4장, 6장에서 다룬 스레드 풀과 동기화 자원의 특성을 함께 고려해서 결정해야 합니다. 예를 들어 쿼리 내부에서 동시 접속 5개로 제한된 외부 API를 호출한다면 `WithDegreeOfParallelism(5)` 근처로 맞추는 것이 합리적입니다.

## ForAll로 결과를 즉시 소비하기

`ToList()`나 `ToArray()`로 PLINQ 결과를 모으면, 앞서 설명한 것처럼 각 스레드의 부분 결과를 하나의 리스트/배열로 **병합**하는 단계가 필요합니다. 기본적으로 이 병합 과정에는 동기화 비용이 따르며, `AsOrdered()`가 걸려 있다면 순서 재정렬 비용까지 추가됩니다.

만약 쿼리의 목적이 "결과를 모아서 반환하는 것"이 아니라 "각 원소에 대해 어떤 동작(부수 효과)을 수행하는 것"이라면, 굳이 병합할 필요가 없습니다. 이런 경우를 위해 PLINQ는 `ForAll()`을 제공합니다.

```csharp
// ToList로 모은 뒤 순차적으로 순회 - 병합 + 순차 반복이라는 이중 비용
numbers.AsParallel()
    .Where(n => IsPrime(n))
    .Select(n => Process(n))
    .ToList()
    .ForEach(r => SaveToDatabase(r));

// ForAll - 각 스레드가 결과를 만들자마자 그 자리에서 바로 소비한다
numbers.AsParallel()
    .Where(n => IsPrime(n))
    .Select(n => Process(n))
    .ForAll(r => SaveToDatabase(r));
```

`ForAll()`은 파이프라인의 각 스레드가 자신이 처리한 원소를 곧바로 `ForAll()`에 전달된 델리게이트로 넘겨버립니다. 즉 "계산 → 병합 → 순회"의 3단계가 아니라 "계산과 소비를 스레드별로 바로 실행"하는 구조가 되어, 결과를 하나로 모으는 동기화 지점이 사라집니다. 대신 대가로 `ForAll()`은 순서를 보장하지 않으며 - `AsOrdered()`가 걸려 있어도 `ForAll()`의 실행 순서 자체는 뒤섞일 수 있습니다 - 반환값도 없습니다(`void`).

정리하면, "최종 결과 컬렉션이 필요하다"면 `ToList()`/`ToArray()`를, "각 원소마다 부수 효과만 일으키면 된다"면 `ForAll()`을 선택합니다. `SaveToDatabase`, 로그 기록, 캐시 갱신처럼 각 항목을 독립적으로 처리하고 끝내는 작업에는 `ForAll()`이 더 적합하고 더 빠릅니다.

`ForAll()`에 전달하는 델리게이트 내부에서 공유 자원에 접근한다면 여전히 스레드 안전성을 직접 책임져야 한다는 점은 변하지 않습니다. 14장에서 다룰 동시성 컬렉션이나 4장의 동기화 프리미티브가 여기서 필요할 수 있습니다.

## 예외 처리

PLINQ 쿼리 실행 중 여러 스레드에서 예외가 발생할 수 있으므로, 8장에서 다룬 `Task`의 예외 처리와 마찬가지로 PLINQ도 이 예외들을 `AggregateException`으로 감싸서 던집니다.

```csharp
try
{
    var result = numbers
        .AsParallel()
        .Select(n => 100 / n) // n이 0이면 DivideByZeroException
        .ToArray();
}
catch (AggregateException ex)
{
    foreach (var inner in ex.Flatten().InnerExceptions)
    {
        Console.WriteLine($"오류: {inner.Message}");
    }
}
```

8장에서 소개한 `Flatten()`은 여기서도 동일하게 중첩된 예외들을 평탄화하는 데 사용할 수 있습니다.

## PLINQ가 항상 더 빠르지는 않다

PLINQ를 도입하면 무조건 성능이 좋아질 것이라 기대하기 쉽지만, 실제로는 순차 LINQ보다 **느려지는 경우도 흔합니다.** 병렬화에는 다음과 같은 고정 비용이 따르기 때문입니다.

- 소스 시퀀스를 여러 청크로 분할하는 비용
- 각 스레드를 스레드 풀에서 가져오고 작업을 배분하는 비용
- 부분 결과를 병합하는 비용(특히 `AsOrdered()`가 걸려 있다면 재정렬 비용까지)
- 스레드 간 데이터 이동에 따른 캐시 지역성(cache locality) 저하

간단한 예로, 정수 100개를 두 배로 만드는 작업처럼 원소당 계산 비용이 극히 낮은 경우 PLINQ는 거의 항상 순차 LINQ보다 느립니다. 분할·배분·병합에 드는 오버헤드가 실제 계산 시간보다 훨씬 크기 때문입니다.

```csharp
// 이런 경우 PLINQ는 오히려 손해다
var result = Enumerable.Range(1, 100)
    .AsParallel()
    .Select(n => n * 2)
    .ToArray();
```

### PLINQ를 고려해야 하는 기준

경험적으로 다음 조건을 **모두** 만족할 때 PLINQ 도입을 검토할 가치가 있습니다.

1. **데이터 양이 충분히 크다.** 최소 수천 개 이상의 원소, 이상적으로는 수만~수백만 개 단위일 때 분할 비용을 상쇄할 수 있습니다.
2. **원소당 처리 비용이 계산량이 많다(CPU-bound).** 소수 판별, 암호화 해시 계산, 복잡한 수치 연산처럼 원소 하나를 처리하는 데 걸리는 시간 자체가 유의미해야 합니다. I/O를 기다리는 작업(파일 읽기, 네트워크 호출)은 PLINQ보다 9~11장에서 다룬 `async`/`await` 기반 비동기 처리가 훨씬 적합합니다. PLINQ는 CPU 바운드 작업을 위한 도구입니다.
3. **각 원소의 처리가 서로 독립적이다.** 앞 원소의 결과가 뒤 원소의 처리에 영향을 주지 않아야 합니다. 누적 합, 이전 상태에 의존하는 계산 등은 병렬화하기 어렵거나 별도의 설계가 필요합니다.
4. **순서가 중요하지 않거나, 순서 보장 비용을 감수할 만하다.** 순서가 반드시 필요하면서 원소당 계산 비용은 낮은 경우, `AsOrdered()`의 오버헤드가 병렬화 이득을 잠식할 수 있습니다.

이 기준에 확신이 서지 않는다면 가장 확실한 방법은 **직접 측정**하는 것입니다. `Stopwatch`로 순차 버전과 PLINQ 버전의 실행 시간을 비교하고, 실제 운영 환경과 유사한 데이터 크기로 벤치마크한 뒤 도입을 결정하는 것이 안전합니다. PLINQ는 "쓰면 무조건 빨라지는 마법"이 아니라, 조건이 맞을 때만 효과를 내는 도구라는 점을 기억해야 합니다.

## 요약

- PLINQ는 LINQ 쿼리를 병렬로 실행하는 엔진으로, 12장의 `Parallel` 클래스가 명령형 반복문을 병렬화하는 것과 대응되는 선언적 스타일의 병렬화 수단이다.
- `AsParallel()`을 호출하면 이후의 LINQ 연산자들이 병렬 버전으로 바인딩되며, `AsSequential()`로 다시 순차 실행으로 되돌릴 수 있다.
- PLINQ는 기본적으로 결과 순서를 보장하지 않는다. 순서가 필요하면 `AsOrdered()`를, 더 이상 필요 없어지면 `AsUnordered()`를 사용해 오버헤드를 조절한다.
- `WithDegreeOfParallelism(n)`으로 사용할 스레드 수를 명시적으로 제한할 수 있으며, 외부 리소스 제약이 있을 때 특히 유용하다.
- 결과 컬렉션이 필요 없이 각 원소에 대한 부수 효과만 필요하다면 `ToList()` 후 순회하는 대신 `ForAll()`을 사용해 병합 비용을 없앨 수 있다.
- PLINQ 실행 중 발생한 예외는 `AggregateException`으로 감싸져 전달되며, `Flatten()`으로 평탄화해서 처리할 수 있다.
- PLINQ는 데이터가 충분히 크고, 원소당 계산 비용이 있으며(CPU-bound), 각 원소 처리가 독립적일 때 이득을 본다. 이 조건이 맞지 않으면 순차 LINQ보다 오히려 느려질 수 있으므로 실제 측정을 통해 도입 여부를 결정해야 한다.

## 연습문제

1. 정수 배열에서 소수만 걸러내는 쿼리를 순차 LINQ와 PLINQ 두 가지 버전으로 작성하고, 배열 크기를 100개, 10,000개, 1,000,000개로 바꿔가며 `Stopwatch`로 실행 시간을 비교해 보세요. 어느 크기부터 PLINQ가 더 빨라지는지 확인하세요.
2. `AsOrdered()`를 사용한 쿼리와 사용하지 않은 쿼리의 실행 시간을 비교하고, 순서 보장에 어느 정도의 비용이 드는지 측정해 보세요.
3. `numbers.AsParallel().ForAll(n => total += n)`처럼 공유 변수를 직접 누적하는 코드가 왜 위험한지 설명하고, `Sum()` 또는 `Aggregate()`를 사용해 안전하게 다시 작성하세요.
4. `WithDegreeOfParallelism(2)`와 `WithDegreeOfParallelism(Environment.ProcessorCount)`로 동일한 쿼리를 실행해 시간 차이를 비교하고, 병렬도를 제한하는 것이 유용한 실제 시나리오를 하나 제시해 보세요.
5. I/O 바운드 작업(예: 여러 URL에서 데이터 다운로드)을 PLINQ로 병렬화하는 것이 왜 적절하지 않은지, 9장과 11장에서 배운 `async`/`await` 기반 접근과 비교해서 설명해 보세요.

---

[◀ 이전: 12장. 병렬 프로그래밍(Parallel 클래스)](ch12-병렬프로그래밍Parallel클래스.md) | [📖 목차](00-목차.md) | [다음: 14장. 동시성 컬렉션 ▶](ch14-동시성컬렉션.md)
