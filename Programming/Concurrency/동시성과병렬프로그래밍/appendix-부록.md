# 부록

[◀ 이전: 18장. 좋은 동시성 코드 작성 습관과 디버깅](ch18-좋은동시성코드작성습관과디버깅.md) | [📖 목차](00-목차.md)


이 부록은 책 전체에서 다룬 주요 클래스와 키워드를 한눈에 찾아볼 수 있는 요약표, 실무에서 동시성 문제를 의심할 때 사용할 수 있는 점검 목록, 참고할 만한 공식 문서 안내, 그리고 이 책을 마친 뒤 이어갈 만한 다음 학습 단계로 구성됩니다.

## 주요 클래스와 키워드 요약표

| 이름 | 네임스페이스 | 한 줄 설명 | 관련 장 |
|---|---|---|---|
| `Thread` | `System.Threading` | 운영체제 스레드를 직접 생성하고 제어하는 가장 저수준의 API | 2장 |
| `lock` 키워드 | C# 언어 키워드 | `Monitor.Enter`/`Exit`를 감싸 상호 배제 구역을 표현하는 구문 | 3장 |
| `Monitor` | `System.Threading` | `lock`의 기반이 되는 상호 배제 및 대기/신호(`Wait`/`Pulse`) 메커니즘 | 3장, 5장 |
| `Mutex` | `System.Threading` | 프로세스 간에도 공유할 수 있는 운영체제 수준의 상호 배제 커널 객체 | 4장 |
| `Semaphore` / `SemaphoreSlim` | `System.Threading` | 동시에 자원에 접근할 수 있는 스레드 수를 제한하는 카운팅 동기화 도구 | 4장 |
| `ReaderWriterLockSlim` | `System.Threading` | 읽기는 동시에 여럿을, 쓰기는 하나만 허용하는 동기화 도구 | 4장 |
| `ManualResetEventSlim` / `AutoResetEvent` | `System.Threading` | 신호를 받을 때까지 스레드를 대기시키는 이벤트 기반 동기화 도구 | 5장 |
| `CountdownEvent` | `System.Threading` | 지정한 횟수만큼 신호를 받을 때까지 대기하는 카운트다운 동기화 도구 | 5장 |
| `ThreadPool` | `System.Threading` | 런타임이 미리 만들어 재사용하는 워커 스레드 집합 | 6장 |
| `Task` / `Task<TResult>` | `System.Threading.Tasks` | 비동기 또는 병렬로 실행되는 작업 단위를 표현하는 추상화 | 7장, 8장 |
| `Task.WhenAll` / `Task.WhenAny` | `System.Threading.Tasks` | 여러 `Task`의 완료를 조합해서 기다리는 정적 메서드 | 8장 |
| `async` / `await` | C# 언어 키워드 | 비동기 코드를 동기 코드처럼 순차적으로 작성할 수 있게 해주는 언어 지원 | 9장, 10장 |
| `SynchronizationContext` | `System.Threading` | `await` 이후 이어지는 코드가 어느 스레드에서 실행될지를 결정하는 문맥 | 9장, 10장 |
| `ConfigureAwait` | `System.Threading.Tasks` | `await` 이후 원래 `SynchronizationContext`로 돌아갈지 여부를 지정 | 10장 |
| `CancellationToken` / `CancellationTokenSource` | `System.Threading` | 협조적 취소 신호를 전달하고 관찰하는 표준 메커니즘 | 11장 |
| `IProgress<T>` / `Progress<T>` | `System.Threading`, `System` | 비동기 작업의 진행률을 호출자에게 보고하는 표준 인터페이스와 구현체 | 11장 |
| `Parallel.For` / `Parallel.ForEach` | `System.Threading.Tasks` | 반복 작업을 여러 코어에 자동으로 분배해 실행하는 데이터 병렬 API | 12장 |
| PLINQ (`AsParallel`) | `System.Linq` | LINQ 쿼리를 병렬로 실행하는 확장 메서드 기반 API | 13장 |
| `ConcurrentDictionary<TKey, TValue>` | `System.Collections.Concurrent` | 여러 스레드가 안전하게 동시 접근할 수 있는 딕셔너리 | 14장 |
| `ConcurrentQueue<T>` / `ConcurrentStack<T>` / `ConcurrentBag<T>` | `System.Collections.Concurrent` | 스레드 안전한 큐, 스택, 순서 없는 컬렉션 | 14장 |
| `BlockingCollection<T>` | `System.Collections.Concurrent` | 생산자-소비자 패턴을 위한 경계(bounded) 지정 가능한 블로킹 컬렉션 | 14장 |
| `IAsyncEnumerable<T>` / `await foreach` | `System.Collections.Generic`, C# 언어 키워드 | 비동기적으로 여러 값을 순차 생성하고 소비하는 비동기 스트림 | 15장 |
| `Channel<T>` | `System.Threading.Channels` | 생산자와 소비자 사이에서 데이터를 안전하게 주고받는 비동기 큐 | 16장 |
| `volatile` 키워드 | C# 언어 키워드 | 필드에 대한 읽기/쓰기가 항상 최신 메모리 값을 반영하도록 보장 | 17장 |
| `Interlocked` | `System.Threading` | 증가/감소/교환/조건부 교체를 CPU 원자적 명령어로 수행 | 17장 |
| `record` / `init` 접근자 | C# 언어 기능 | 불변 객체를 간결하게 정의해 동기화 없이 스레드 안전성을 확보 | 17장 |

## 동시성 문제 진단 체크리스트

코드를 리뷰하거나 원인을 알 수 없는 버그를 조사할 때, 다음 질문들을 순서대로 점검해보면 동시성 문제인지 여부와 그 성격을 좁혀나가는 데 도움이 됩니다.

**공유 상태 존재 여부**

- [ ] 이 필드, 속성, 컬렉션은 하나의 인스턴스를 통해 둘 이상의 실행 흐름에서 접근될 수 있는가?
- [ ] 정적(static) 필드나 싱글턴 인스턴스처럼 애플리케이션 전체에서 공유되는 상태인가?

**접근 주체 확인**

- [ ] 여러 스레드가 동시에 접근하는가, 아니면 여러 `Task`나 콜백이 같은 `SynchronizationContext`를 공유해서 순차적으로만 접근하는가?
- [ ] "지금은 한 스레드에서만 호출된다"는 가정에 의존하고 있다면, 그 가정이 코드 변경이나 배포 환경 변화로 깨질 가능성은 없는가?

**읽기/쓰기 패턴**

- [ ] 접근하는 스레드 중 하나라도 값을 변경하는가, 아니면 모두 읽기 전용인가? (모두 읽기 전용이고 초기화 이후 절대 바뀌지 않는다면 동기화가 필요 없을 수 있습니다.)
- [ ] 읽기와 쓰기가 섞여 있다면, 그 접근이 `lock`, 동시성 컬렉션, `Interlocked`, 불변성 중 어떤 방식으로 보호되고 있는가?
- [ ] "값을 읽고 -> 계산하고 -> 다시 쓰는" 것처럼 여러 단계로 이루어진 연산인가? 이런 연산은 단순히 `volatile`이나 개별 원자적 읽기/쓰기만으로는 보호되지 않습니다.

**비동기 코드 특유의 함정**

- [ ] `async` 메서드가 예외를 던질 수 있는데 반환 타입이 `void`로 되어 있지는 않은가?
- [ ] `.Result`, `.Wait()`, `.GetAwaiter().GetResult()`처럼 비동기 작업을 동기적으로 기다리는 코드가 `SynchronizationContext`가 있는 환경에서 실행되지는 않는가?
- [ ] 취소 토큰을 받는 작업이 실제로 그 토큰을 관찰하고 있는가, 아니면 매개변수로만 전달받고 무시하고 있는가?

**동시성 자체를 없앨 수 있는가**

- [ ] 이 데이터를 아예 불변으로 만들어 동기화 자체를 없앨 수는 없는가?
- [ ] `.NET`이 이미 제공하는 동시성 컬렉션이나 `Channel<T>`로 직접 만든 동기화 로직을 대체할 수는 없는가?

## 참고할 만한 공식 문서

이 책에서 다룬 내용을 더 깊이 확인하고 싶다면 마이크로소프트의 공식 학습 문서를 참고하는 것이 가장 정확합니다. 아래는 검색해서 찾아볼 만한 문서 주제를 이름으로만 안내합니다. 모두 `learn.microsoft.com`(Microsoft Learn) 사이트의 .NET 문서 영역에서 찾을 수 있습니다.

- .NET의 "Managed Threading(관리되는 스레딩)" 개념 문서 - 스레드, 동기화 프리미티브, 스레드 풀의 기본 개념
- "Task Parallel Library(TPL)" 문서 - `Task`, `Task<TResult>`, 작업 조합과 예외 처리
- "Asynchronous programming with async and await" 문서 - async/await의 동작 원리와 모범 사례
- "Parallel programming in .NET" 문서 - `Parallel` 클래스와 PLINQ
- "System.Threading.Channels" 네임스페이스 문서 - Channel 기반 생산자-소비자 패턴
- "Thread-safe collections" 문서 - `System.Collections.Concurrent` 네임스페이스의 컬렉션들
- ".NET 메모리 모델" 관련 문서 및 C# 언어 사양의 `volatile` 키워드 설명
- "TPL Dataflow" 문서 - `System.Threading.Tasks.Dataflow` 네임스페이스

공식 문서는 API의 정확한 동작과 예외 상황, 버전별 변경 사항을 확인하는 데 특히 유용합니다. 이 책에서 설명한 개념과 원리를 바탕에 두고 공식 문서를 함께 참고하면, 새로운 .NET 버전에서 추가되는 세부 사항도 어렵지 않게 소화할 수 있을 것입니다.

## 다음 학습 단계 제안

이 책을 마쳤다면 다음과 같은 순서로 학습을 이어가는 것을 권합니다.

1. **직접 만든 프로젝트에 적용해보기**. 이론을 읽는 것과 실제로 경쟁 상태를 만들어보고, 그것을 `lock`과 `Interlocked`와 불변성 각각으로 고쳐보는 것은 완전히 다른 경험입니다. 작은 콘솔 프로젝트에서 의도적으로 버그를 만들고 고쳐보는 연습을 추천합니다.
2. **18장에서 언급한 심화 주제 중 하나를 골라 깊이 파보기**. TPL Dataflow로 파이프라인을 구성해보거나, 락-프리 자료구조를 직접 구현해보면서 `Interlocked.CompareExchange`의 재시도 루프를 몸에 익히는 것도 좋은 다음 단계입니다.
3. **성능 프로파일링 도구 익히기**. 동시성 코드는 정확성뿐 아니라 성능이 목적인 경우가 많습니다. Visual Studio의 진단 도구나 `dotnet-trace` 같은 커맨드라인 도구로 실제 스레드 경합과 대기 시간을 측정하는 방법을 익혀두면, 18장에서 다룬 "측정 후 최적화" 원칙을 실전에서 적용할 수 있습니다.
4. **오픈소스 코드 읽기**. .NET 런타임과 `System.Collections.Concurrent`, `System.Threading.Channels`의 실제 구현은 모두 공개되어 있습니다. 이 책에서 배운 개념이 실제 프로덕션 코드에서 어떻게 응용되는지 확인해보는 것은 수준을 한 단계 끌어올리는 좋은 방법입니다.

동시성과 병렬 프로그래밍은 한 번 배우고 끝나는 주제가 아니라, 새로운 문제를 만날 때마다 이 책에서 익힌 사고방식, 즉 "공유 상태가 있는가", "그것을 어떻게 안전하게 조율할 것인가"를 반복해서 적용해나가는 여정입니다. 이 책이 그 여정의 튼튼한 출발점이 되었기를 바랍니다.

---

[◀ 이전: 18장. 좋은 동시성 코드 작성 습관과 디버깅](ch18-좋은동시성코드작성습관과디버깅.md) | [📖 목차](00-목차.md)
