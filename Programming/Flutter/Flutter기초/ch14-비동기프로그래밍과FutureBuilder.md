# 14장. 비동기 프로그래밍과 FutureBuilder

📖 [◀ 목차](00-목차.md) | [◀ 이전: 13장. 상태 관리 심화](ch13-상태관리심화.md) | [다음: 15장. 네트워크 통신 ▶](ch15-네트워크통신.md)

---

지금까지 만들어본 화면들은 데이터가 이미 준비되어 있다고 가정하고 곧바로 화면에 그려왔습니다. 하지만 실제 앱은 서버에서 데이터를 내려받거나, 기기에 저장된 파일을 읽거나, 데이터베이스를 조회하는 등 시간이 걸리는 작업을 자주 수행해야 합니다. 이런 작업은 결과가 도착할 때까지 화면을 멈춰 세울 수 없고, 기다리는 동안에도 사용자에게 무언가 진행되고 있다는 신호를 보여줘야 합니다. 이 장에서는 Dart의 비동기 프로그래밍 도구를 짧게 복습한 뒤, Flutter가 이런 상황을 위해 제공하는 `FutureBuilder` 위젯을 자세히 다룹니다.

## Future와 async/await 복습

Dart 기초를 다룬 책에서 이미 `Future`와 `async`/`await` 문법을 배웠으므로, 여기서는 핵심만 짧게 되짚고 곧바로 UI에 적용하는 데 집중하겠습니다.

`Future<T>`는 지금 당장은 값이 없지만 미래의 어느 시점에 `T` 타입의 값(또는 에러)을 만들어낼 것이라는 약속을 나타내는 객체입니다. 네트워크 요청, 파일 입출력, 일정 시간 뒤 실행되는 타이머 등은 모두 결과를 `Future`로 반환합니다.

```dart
Future<String> fetchGreeting() async {
  await Future.delayed(const Duration(seconds: 2));
  return '안녕하세요, Flutter!';
}
```

`fetchGreeting`처럼 `async` 키워드가 붙은 함수는 내부에서 `await`를 사용해 다른 `Future`가 완료될 때까지 기다렸다가 다음 코드를 실행할 수 있습니다. `await Future.delayed(...)`는 실제 코드를 2초간 멈추는 것이 아니라, 2초 뒤에 이어서 실행되도록 예약해두고 그동안 다른 작업(예: 화면을 그리는 이벤트 루프)이 계속 진행되도록 양보한다는 점이 중요합니다. 바로 이 특성 때문에 Flutter 앱에서 비동기 작업을 수행하는 동안에도 앱이 멈추지 않고 계속 반응할 수 있습니다.

문제는 이 함수를 호출하는 쪽입니다. `fetchGreeting()`을 호출하는 순간에는 아직 문자열이 존재하지 않고, 2초가 지나야 값이 도착합니다. 그 사이 화면에는 무엇을 그려야 할까요? 그리고 값이 도착하면 화면을 어떻게 다시 그려야 할까요?

## 비동기 작업과 화면 갱신이라는 문제

가장 단순하게 생각하면 6장에서 배운 `setState`로 직접 처리할 수 있습니다.

```dart
class GreetingScreen extends StatefulWidget {
  const GreetingScreen({super.key});

  @override
  State<GreetingScreen> createState() => _GreetingScreenState();
}

class _GreetingScreenState extends State<GreetingScreen> {
  String? _greeting;
  bool _isLoading = true;

  @override
  void initState() {
    super.initState();
    _loadGreeting();
  }

  Future<void> _loadGreeting() async {
    final result = await fetchGreeting();
    setState(() {
      _greeting = result;
      _isLoading = false;
    });
  }

  @override
  Widget build(BuildContext context) {
    if (_isLoading) {
      return const Center(child: CircularProgressIndicator());
    }
    return Center(child: Text(_greeting!));
  }
}
```

이 코드는 동작은 하지만, 로딩 상태(`_isLoading`)와 결과값(`_greeting`)을 각각 별도의 필드로 관리해야 하고, 에러가 발생하는 경우까지 고려하면 `_hasError` 같은 필드가 하나 더 늘어납니다. `initState`에서 비동기 함수를 호출하고 완료되면 `setState`를 부르는 패턴도 매번 반복해서 작성해야 합니다. 게다가 위젯이 화면에서 사라진 뒤에 `Future`가 완료되어 `setState`가 호출되면 오류가 발생할 수 있어, 이를 막기 위한 추가 처리까지 필요해집니다.

Flutter는 "하나의 `Future`가 완료되기를 기다렸다가 그 결과에 따라 다른 위젯을 보여준다"는 이 패턴이 워낙 흔하기 때문에, 이를 전담하는 위젯을 코어 프레임워크에 포함시켰습니다. 바로 `FutureBuilder`입니다.

## FutureBuilder로 비동기 상태를 선언적으로 다루기

`FutureBuilder<T>`는 감시할 `future`와, 그 `Future`의 현재 상태에 따라 어떤 위젯을 그릴지 결정하는 `builder` 함수를 받습니다.

```dart
FutureBuilder<String>(
  future: fetchGreeting(),
  builder: (BuildContext context, AsyncSnapshot<String> snapshot) {
    if (snapshot.connectionState == ConnectionState.waiting) {
      return const Center(child: CircularProgressIndicator());
    }
    if (snapshot.hasError) {
      return Center(child: Text('오류가 발생했습니다: ${snapshot.error}'));
    }
    if (snapshot.hasData) {
      return Center(child: Text(snapshot.data!));
    }
    return const SizedBox.shrink();
  },
)
```

`builder`는 `Future`가 아직 진행 중일 때, 완료됐을 때, 에러가 났을 때 등 상태가 바뀔 때마다 다시 호출되며, 그때마다 현재 상태를 담은 `AsyncSnapshot<T>` 객체를 전달받습니다. `snapshot`이 제공하는 정보는 크게 두 가지입니다.

- `snapshot.connectionState`: `Future`가 지금 어떤 단계에 있는지를 나타내는 `ConnectionState` 열거형 값입니다. 아직 `Future`가 연결되지 않은 `none`, 결과를 기다리는 중인 `waiting`, 완료된 `done` 등이 있습니다. 대개는 `ConnectionState.waiting`인지만 확인해 로딩 인디케이터를 보여줄지 판단합니다.
- `snapshot.hasData` / `snapshot.hasError`: `Future`가 완료되었을 때 정상적으로 값을 반환했는지, 아니면 예외를 던지며 실패했는지를 알려줍니다. `hasData`가 `true`이면 `snapshot.data`로 실제 결과값을 꺼낼 수 있고, `hasError`가 `true`이면 `snapshot.error`로 발생한 예외를 확인할 수 있습니다.

이렇게 상태 확인 순서를 정리하면, 위 코드는 "기다리는 중이면 로딩 인디케이터, 에러가 있으면 에러 메시지, 데이터가 있으면 결과 화면, 그 외의 경우는 빈 화면"이라는 흐름을 한눈에 읽을 수 있는 형태로 표현됩니다. `setState`를 직접 쓰던 앞의 예제와 비교하면, 로딩·에러·완료라는 세 가지 상태를 별도의 필드 없이 `snapshot` 하나로 통합해서 다룰 수 있다는 점이 가장 큰 차이입니다.

### future 매개변수를 다룰 때 주의할 점

`FutureBuilder`를 사용할 때 자주 하는 실수 중 하나는 `future` 매개변수에 `build` 메서드 안에서 새로 함수를 호출한 결과를 직접 전달하는 것입니다.

```dart
// 주의: build가 호출될 때마다 fetchGreeting()이 다시 실행된다.
@override
Widget build(BuildContext context) {
  return FutureBuilder<String>(
    future: fetchGreeting(),
    builder: (context, snapshot) {
      // ...
    },
  );
}
```

이렇게 작성하면 부모 위젯이 다시 빌드될 때마다(예를 들어 다른 상태가 바뀌어 `setState`가 호출될 때마다) `fetchGreeting()`이 매번 새로 호출되어, 이미 받아온 데이터를 무시하고 처음부터 다시 요청을 보내는 문제가 생깁니다. 이를 피하려면 `Future`를 `State`의 필드에 미리 만들어두고, `build`에서는 그 필드를 참조하도록 해야 합니다.

```dart
class GreetingScreen extends StatefulWidget {
  const GreetingScreen({super.key});

  @override
  State<GreetingScreen> createState() => _GreetingScreenState();
}

class _GreetingScreenState extends State<GreetingScreen> {
  late final Future<String> _greetingFuture;

  @override
  void initState() {
    super.initState();
    _greetingFuture = fetchGreeting();
  }

  @override
  Widget build(BuildContext context) {
    return FutureBuilder<String>(
      future: _greetingFuture,
      builder: (context, snapshot) {
        if (snapshot.connectionState == ConnectionState.waiting) {
          return const Center(child: CircularProgressIndicator());
        }
        if (snapshot.hasError) {
          return Center(child: Text('오류가 발생했습니다: ${snapshot.error}'));
        }
        return Center(child: Text(snapshot.data!));
      },
    );
  }
}
```

`_greetingFuture`를 `initState`에서 한 번만 생성해두면, 이후 `build`가 몇 번을 다시 호출되더라도 `FutureBuilder`는 동일한 `Future` 인스턴스를 계속 관찰하므로 불필요하게 요청이 반복되지 않습니다.

## StreamBuilder: 값이 한 번이 아니라 계속 흘러올 때

`FutureBuilder`는 한 번의 요청에 대해 한 번의 결과만 받는 상황에 알맞습니다. 하지만 실시간 채팅 메시지, 센서 값, 타이머처럼 시간이 지나면서 값이 여러 번 연속으로 전달되는 경우도 있습니다. 이런 경우 Dart는 `Future` 대신 `Stream`을 사용하며, Flutter는 이를 위한 짝으로 `StreamBuilder`를 제공합니다.

```dart
StreamBuilder<int>(
  stream: Stream.periodic(const Duration(seconds: 1), (count) => count),
  builder: (BuildContext context, AsyncSnapshot<int> snapshot) {
    if (snapshot.connectionState == ConnectionState.waiting) {
      return const Text('시작 대기 중...');
    }
    return Text('경과 시간: ${snapshot.data}초');
  },
)
```

사용법은 `FutureBuilder`와 거의 같습니다. `future` 대신 `stream` 매개변수를 받고, `builder`에 전달되는 `snapshot`도 동일하게 `connectionState`, `hasData`, `hasError`를 제공합니다. 차이는 `Stream`이 완료되기 전까지 여러 번에 걸쳐 새 값을 내보낼 수 있다는 점이며, `StreamBuilder`는 새 값이 도착할 때마다 자동으로 `builder`를 다시 호출해 화면을 갱신합니다. 즉 "결과를 딱 한 번만 받는가, 아니면 값이 계속 흘러 들어오는가"가 `FutureBuilder`와 `StreamBuilder` 중 무엇을 선택할지를 가르는 기준이 됩니다. 이 책에서는 대부분의 예제가 한 번의 요청과 한 번의 응답으로 이루어지므로 `FutureBuilder`를 중심으로 다루지만, 실시간으로 갱신되는 데이터를 다뤄야 할 때는 `StreamBuilder`가 있다는 것을 기억해두면 좋습니다.

## 다음 장을 위한 준비

이 장에서 배운 `FutureBuilder` 패턴은 다음 15장에서 다룰 네트워크 통신에서 곧바로 실전에 쓰이게 됩니다. HTTP 요청으로 서버에서 JSON 데이터를 받아오는 작업은 전형적인 비동기 작업이며, 요청을 보내는 동안 로딩 인디케이터를 보여주고, 응답이 도착하면 파싱된 데이터를 화면에 표시하고, 요청이 실패하면 에러 메시지를 보여주는 흐름 전체가 이 장에서 익힌 `connectionState`, `hasData`, `hasError` 확인 패턴 그대로 적용됩니다. 지금 이 패턴을 충분히 익혀두면 15장에서는 "무엇을 요청할 것인가"에만 집중하면 됩니다.

## 요약

- `Future<T>`는 미래에 값을 반환할 비동기 작업을 나타내며, `async`/`await` 문법으로 결과를 순차적인 코드처럼 다룰 수 있습니다. 이는 Dart 기초에서 다룬 내용이며, 이 장에서는 이를 UI에 적용하는 방법에 집중했습니다.
- 비동기 작업이 진행되는 동안 로딩 인디케이터를 보여주고 완료되면 결과를 반영하는 패턴을 `setState`만으로 구현하면 로딩·에러·데이터 상태를 각각 별도 필드로 관리해야 해 번거로워집니다.
- `FutureBuilder<T>`는 `future`와 `builder`를 받아, `Future`의 진행 상태에 따라 다른 위젯을 그릴 수 있게 해주는 위젯입니다. `builder`에 전달되는 `AsyncSnapshot<T>`의 `connectionState`로 로딩 여부를, `hasData`/`hasError`로 성공·실패 여부를 확인합니다.
- `FutureBuilder`의 `future`에는 `build` 메서드 안에서 새로 생성한 `Future`가 아니라, `initState` 등에서 미리 만들어 필드에 저장해둔 `Future`를 전달해야 불필요한 재요청을 막을 수 있습니다.
- `StreamBuilder`는 `FutureBuilder`와 사용법이 비슷하지만, 한 번의 결과가 아니라 시간에 걸쳐 여러 번 전달되는 `Stream`의 값을 다룰 때 사용합니다.
- 이 패턴은 15장의 네트워크 통신에서 서버 응답을 화면에 표시하는 데 바로 활용됩니다.

## 연습문제

1. `Future`가 완료되기를 `await`로 기다리는 동안, Flutter 앱의 이벤트 루프에는 어떤 일이 일어나는지 설명하세요.
2. `AsyncSnapshot`의 `connectionState`, `hasData`, `hasError`가 각각 어떤 정보를 제공하는지 서술하세요.
3. 파일에서 설정값을 읽어오는 다음 함수가 있다고 합시다.

   ```dart
   Future<String> loadSettings() async {
     await Future.delayed(const Duration(seconds: 1));
     return 'dark_mode=true';
   }
   ```

   이 함수의 결과를 `FutureBuilder`로 표시하는 위젯을 작성하세요. 로딩 중에는 `CircularProgressIndicator`를, 에러가 발생하면 에러 메시지를, 성공하면 결과 문자열을 화면에 보여줘야 합니다.
4. `FutureBuilder`의 `future` 매개변수에 `build` 메서드 안에서 직접 함수를 호출한 결과를 전달하면 어떤 문제가 생길 수 있는지 설명하고, 이를 피하는 방법을 코드로 보이세요.
5. `FutureBuilder`와 `StreamBuilder`를 각각 사용해야 하는 상황을 하나씩 예로 들어 설명하세요.


---

📖 [◀ 목차](00-목차.md) | [◀ 이전: 13장. 상태 관리 심화](ch13-상태관리심화.md) | [다음: 15장. 네트워크 통신 ▶](ch15-네트워크통신.md)
