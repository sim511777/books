# 15장. sealed class와 when 심화

[◀ 이전: 14장. 확장 함수](ch14-확장함수.md) | [📖 목차](00-목차.md) | [다음: 16장. 코루틴 기초 ▶](ch16-코루틴기초.md)

---

## 들어가며

5장에서 `when`을 배울 때, 이 문법이 여러 조건을 깔끔하게 분기할 수 있는 강력한 도구라는 것을 확인했습니다. 하지만 5장의 `when`에는 한 가지 약점이 있었습니다. 조건을 하나 빠뜨려도 컴파일러가 알려주지 않는다는 점입니다. 예를 들어 `Int` 값을 분기하는 `when`에서 특정 숫자 하나를 처리하는 분기를 빠뜨려도, 컴파일러는 `else`가 없다는 것 정도만 지적할 뿐 "그 숫자를 놓쳤다"는 사실은 알려주지 않습니다. 애초에 `Int`가 가질 수 있는 값이 무한히 많기 때문에, 컴파일러가 "모든 경우"를 계산할 방법이 없는 것입니다.

이 장에서는 이 문제를 근본적으로 해결하는 `sealed class`를 배웁니다. sealed class는 "이 타입의 하위 타입은 정해진 몇 개뿐이다"라고 컴파일러에게 못박아 두는 선언입니다. 하위 타입의 개수가 정해져 있으면, 컴파일러는 `when`이 그 하위 타입을 하나도 빠뜨리지 않았는지 정확하게 검사할 수 있습니다. 10장에서 배운 상속 개념과 5장의 `when`이 이 장에서 하나로 합쳐집니다.

## sealed class란 무엇인가

`sealed`는 "봉인된"이라는 뜻입니다. sealed class로 선언된 클래스는 자신을 상속하는 하위 클래스의 목록을 그 자리에서 봉인해 버립니다. 즉, sealed class를 상속하는 클래스는 반드시 같은 파일(또는 같은 패키지 안의 정해진 범위) 안에서만 선언할 수 있고, 외부의 다른 파일에서 마음대로 새로운 하위 클래스를 추가할 수 없습니다.

10장에서 배운 일반 상속을 떠올려 봅시다. `open class Animal`을 선언하면 어느 파일에서든 `class Dog : Animal()`, `class Cat : Animal()`처럼 하위 클래스를 얼마든지 추가할 수 있었습니다. 이 유연함은 라이브러리를 설계할 때는 장점이지만, "이 타입은 딱 이 세 가지 경우로만 존재한다"는 것을 코드로 보장하고 싶을 때는 오히려 방해가 됩니다. sealed class는 바로 이런 상황을 위한 문법입니다.

```kotlin
// 네트워크 요청의 결과 상태를 표현하는 sealed class
sealed class NetworkResult

class Loading : NetworkResult()
class Success(val data: String) : NetworkResult()
class Error(val message: String) : NetworkResult()
```

이렇게 선언하면 `NetworkResult`의 하위 타입은 `Loading`, `Success`, `Error` 세 가지뿐이라는 사실이 컴파일러 차원에서 고정됩니다. 다른 파일에서 `class Timeout : NetworkResult()`를 추가하려고 하면 컴파일 오류가 발생합니다(정확히는 코틀린 1.5부터는 같은 컴파일 모듈 안이라면 다른 파일에서도 하위 클래스 선언이 가능하지만, 여전히 그 목록은 컴파일 시점에 컴파일러가 전부 알고 있는 상태로 닫혀 있습니다).

## sealed class와 when이 만나면: exhaustiveness 검사

sealed class의 진짜 힘은 `when`과 결합할 때 드러납니다. `when`의 대상이 sealed class라면, 컴파일러는 그 sealed class의 모든 하위 타입이 `when` 안에서 처리되었는지를 검사합니다. 이를 흔히 "빠짐없음 검사(exhaustiveness check)"라고 부릅니다.

```kotlin
fun describe(result: NetworkResult): String {
    return when (result) {
        is Loading -> "로딩 중입니다."
        is Success -> "성공: ${result.data}"
        is Error -> "실패: ${result.message}"
    }
    // else가 없어도 컴파일 오류가 나지 않습니다.
    // Loading, Success, Error 세 가지를 모두 처리했기 때문입니다.
}
```

이 코드에는 `else` 분기가 없습니다. 그런데도 컴파일 오류가 나지 않는 이유는, 컴파일러가 `NetworkResult`의 하위 타입이 정확히 세 개뿐이라는 것을 알고 있고, 그 세 개가 모두 `when` 안에 등장했다는 것을 확인했기 때문입니다. 이 `when`을 함수의 반환값으로 사용하려면(즉 표현식으로 사용하려면) 이 빠짐없음 검사를 반드시 통과해야 합니다.

이제 여기에 새로운 상태 하나를 추가해 봅시다.

```kotlin
class Timeout : NetworkResult()
```

이렇게 `NetworkResult`의 하위 클래스를 하나 더 추가하면, 위에서 작성한 `describe` 함수는 더 이상 컴파일되지 않습니다. `Timeout`을 처리하는 분기가 없다는 컴파일 오류가 즉시 발생합니다. 이것이 바로 sealed class가 주는 가장 큰 실무적 가치입니다. 새로운 상태를 추가했을 때, "이 상태를 처리해야 하는 모든 곳"을 컴파일러가 자동으로 찾아 주는 것입니다. 만약 `NetworkResult`가 sealed class가 아니라 일반 `open class`였다면, `Timeout`을 깜빡하고 처리하지 않아도 컴파일은 그냥 통과했을 것이고, 실행 중에야 문제를 발견했을 것입니다.

## sealed class는 각 하위 타입마다 다른 데이터를 가질 수 있다

6장까지 다룬 반복문이나 5장의 `when`을 Java의 열거형(enum)과 비교해 본 독자라면, "이건 enum class로도 되지 않나?"라는 생각이 들 수 있습니다. 실제로 Kotlin에는 `enum class`가 있고, `when`과 함께 쓰면 sealed class와 매우 비슷하게 빠짐없음 검사를 받을 수 있습니다.

```kotlin
enum class Direction { NORTH, SOUTH, EAST, WEST }

fun move(direction: Direction) = when (direction) {
    Direction.NORTH -> "위로 이동"
    Direction.SOUTH -> "아래로 이동"
    Direction.EAST -> "오른쪽으로 이동"
    Direction.WEST -> "왼쪽으로 이동"
}
```

하지만 enum class와 sealed class 사이에는 결정적인 차이가 있습니다. **enum class의 각 상수는 하나의 고정된 인스턴스이며, 모두 같은 형태의 데이터만 가질 수 있습니다.** 반면 **sealed class의 각 하위 타입은 완전히 다른 클래스이므로, 서로 다른 종류의 데이터와 다른 개수의 프로퍼티를 가질 수 있습니다.**

앞서 본 `NetworkResult` 예제를 다시 보면, `Success`는 `data: String`이라는 프로퍼티를 가지고 `Error`는 `message: String`이라는 전혀 다른 프로퍼티를 가지며, `Loading`은 아무 프로퍼티도 없습니다. 이런 구조는 enum class로는 표현할 수 없습니다. enum class로 억지로 표현하려고 하면 다음과 같이 모든 상수가 사용하지 않는 프로퍼티까지 떠안게 됩니다.

```kotlin
// enum class로 억지로 표현하면 이렇게 됩니다 (권장하지 않는 방식)
enum class NetworkResultEnum(val data: String?, val message: String?) {
    LOADING(null, null),
    SUCCESS("어떤 데이터", null),
    ERROR(null, "어떤 오류")
}
```

이 방식은 `LOADING` 상태인데도 `data`와 `message` 프로퍼티에 접근할 수 있는 것처럼 보이고, 실제로는 항상 `null`이 나옵니다. 8장에서 다룬 null 안전성의 관점에서 보면 이런 설계는 불필요한 널러블 타입을 강제로 만들어내는 셈이라 바람직하지 않습니다. sealed class를 쓰면 `Success`일 때만 `data`에 접근할 수 있고, 그 값은 결코 null일 수 없다는 것을 타입 시스템으로 보장할 수 있습니다.

정리하면, enum class는 "고정된 개수의 동일한 형태를 가진 값들"을 표현할 때 적합하고, sealed class는 "고정된 개수의 서로 다른 형태를 가진 타입들"을 표현할 때 적합합니다.

## data class와 함께 사용하기

sealed class의 각 하위 타입은 보통 9장, 10장에서 배운 `data class`로 선언합니다. `data class`가 제공하는 `equals()`, `toString()`, 그리고 `when`의 `is` 검사와 결합되는 스마트 캐스트 덕분에, 하위 타입의 프로퍼티에 안전하고 편리하게 접근할 수 있습니다.

```kotlin
// 하위 타입을 data class로 선언
sealed class UiState

data class Loading(val progress: Int) : UiState()
data class Success(val items: List<String>) : UiState()
data class Error(val code: Int, val message: String) : UiState()

fun render(state: UiState) {
    when (state) {
        is Loading -> println("진행률: ${state.progress}%")
        is Success -> println("항목 ${state.items.size}개 로드 완료")
        is Error -> println("오류 코드 ${state.code}: ${state.message}")
    }
}

fun main() {
    render(Loading(42))
    render(Success(listOf("사과", "바나나")))
    render(Error(404, "찾을 수 없음"))
}
```

이 코드에서 `is Loading`으로 분기한 블록 안에서는 `state`가 자동으로 `Loading` 타입으로 취급되어 `state.progress`에 바로 접근할 수 있습니다. 이것이 5장에서 잠깐 언급했던 스마트 캐스트(smart cast)이며, sealed class와 결합할 때 특히 강력하게 작동합니다. `is` 검사 하나로 타입을 좁히는 동시에, 그 타입에만 있는 프로퍼티까지 안전하게 꺼내 쓸 수 있는 것입니다.

## object와 함께 사용하기: 데이터가 없는 상태 표현하기

`Loading`처럼 아무 데이터도 필요 없는 상태라면, 매번 `Loading()`처럼 인스턴스를 새로 만들 필요 없이 `object` 키워드로 선언하는 것이 더 자연스럽습니다. `object`는 클래스를 선언하면서 동시에 그 클래스의 인스턴스를 딱 하나만 만들어 주는 문법입니다.

```kotlin
sealed class DownloadState

object Idle : DownloadState()
data class InProgress(val percent: Int) : DownloadState()
data class Completed(val fileName: String) : DownloadState()
object Failed : DownloadState()

fun report(state: DownloadState) = when (state) {
    is Idle -> "대기 중"
    is InProgress -> "다운로드 중: ${state.percent}%"
    is Completed -> "완료: ${state.fileName}"
    is Failed -> "다운로드 실패"
}
```

데이터가 필요한 상태는 `data class`로, 데이터가 필요 없는 단순한 상태는 `object`로 선언하는 조합은 실무에서 매우 흔하게 쓰이는 패턴입니다. 특히 안드로이드나 서버 애플리케이션에서 화면 상태, API 응답 상태, 처리 결과 등을 표현할 때 sealed class와 이 조합은 거의 표준처럼 사용됩니다.

## when을 표현식으로 사용할 때의 강제성

한 가지 주의할 점은, 빠짐없음 검사는 `when`을 **표현식**으로 사용할 때만 강제된다는 것입니다. 5장에서 살펴본 것처럼 `when`은 문장(statement)으로도, 표현식(expression)으로도 쓸 수 있습니다.

```kotlin
// 문장으로 사용: else가 없어도 컴파일은 되지만, 빠뜨린 경우는 조용히 무시됩니다.
fun logStatement(result: NetworkResult) {
    when (result) {
        is Success -> println("성공")
        is Error -> println("실패")
        // Loading을 빠뜨렸지만 문장이라 컴파일 오류가 나지 않습니다.
    }
}

// 표현식으로 사용: 반환값이 필요하므로 모든 경우를 처리해야 합니다.
fun logExpression(result: NetworkResult): String {
    return when (result) {
        is Success -> "성공"
        is Error -> "실패"
        // Loading을 빠뜨리면 여기서는 컴파일 오류가 발생합니다.
    }
}
```

따라서 sealed class의 빠짐없음 검사를 제대로 활용하려면, 가능한 한 `when`을 값을 반환하는 표현식 형태로 사용하는 습관을 들이는 것이 좋습니다. 이 습관은 18장에서 다루는 "좋은 Kotlin 코드 작성 습관"과도 자연스럽게 연결됩니다.

## sealed interface

Kotlin에는 `sealed class`뿐 아니라 `sealed interface`도 있습니다. 11장에서 배운 인터페이스는 여러 개를 동시에 구현(implement)할 수 있다는 특징이 있는데, sealed interface는 이 특징을 유지하면서도 구현 목록을 봉인합니다.

```kotlin
sealed interface Shape

data class Circle(val radius: Double) : Shape
data class Rectangle(val width: Double, val height: Double) : Shape

fun area(shape: Shape): Double = when (shape) {
    is Circle -> Math.PI * shape.radius * shape.radius
    is Rectangle -> shape.width * shape.height
}
```

`sealed class`를 상속하는 하위 타입은 오직 하나의 부모만 가질 수 있는 반면(10장에서 배웠듯 Kotlin은 단일 상속만 허용), `sealed interface`를 구현하는 타입은 다른 클래스를 상속하면서 동시에 이 인터페이스도 구현할 수 있습니다. 표현하고자 하는 타입이 이미 다른 클래스를 상속하고 있다면 `sealed interface`가 더 유연한 선택이 됩니다.

## 요약

- `when`이 `Int`나 `String` 같은 무한한 값의 타입을 다룰 때는 컴파일러가 모든 경우를 다 처리했는지 검사할 수 없지만, `sealed class`로 하위 타입의 개수를 고정하면 `when`의 빠짐없음 검사(exhaustiveness check)가 가능해집니다.
- `sealed class`의 하위 클래스는 정해진 범위 밖에서 추가로 선언할 수 없으므로, 컴파일러는 항상 "가능한 모든 하위 타입"을 알고 있습니다.
- 새로운 하위 타입을 추가했는데 어떤 `when` 표현식이 그 타입을 처리하지 않으면, 컴파일 오류로 즉시 알 수 있습니다. 이는 실무에서 상태 추가에 따른 누락 버그를 예방하는 강력한 도구입니다.
- `enum class`는 모든 상수가 같은 형태의 데이터를 갖는 반면, `sealed class`의 각 하위 타입은 서로 다른 프로퍼티와 데이터를 가질 수 있습니다. 대표적인 예로 로딩/성공/오류 상태처럼 상태마다 필요한 데이터가 다른 경우가 있습니다.
- 하위 타입은 데이터가 있으면 `data class`, 데이터가 없으면 `object`로 선언하는 조합이 자주 쓰입니다.
- 빠짐없음 검사는 `when`을 값을 반환하는 표현식으로 사용할 때만 강제되므로, 가능하면 `when`을 표현식으로 사용하는 것이 좋습니다.
- 인터페이스 기반으로 유연하게 설계하고 싶다면 `sealed interface`를 사용할 수 있습니다.

## 연습문제

1. `sealed class`가 일반 `open class`와 비교해 `when`과 함께 사용할 때 어떤 이점을 주는지 설명해 보세요.
2. 결제 상태를 표현하는 `sealed class Payment`를 만들어 보세요. `Pending`, `Approved(val transactionId: String)`, `Declined(val reason: String)` 세 가지 하위 타입을 갖도록 설계하고, 이를 처리하는 `when` 표현식을 작성해 보세요.
3. `enum class`로는 표현하기 어렵지만 `sealed class`로는 자연스럽게 표현할 수 있는 상황을 한 가지 더 예로 들어 설명해 보세요.
4. 연습문제 2번의 `Payment`에 `Refunded(val amount: Int)`라는 새로운 하위 타입을 추가했을 때, 기존에 작성한 `when` 표현식에 어떤 일이 일어나는지 직접 컴파일해서 확인해 보세요.
5. `sealed class`와 `sealed interface`의 차이점을 설명하고, 어떤 상황에서 `sealed interface`를 선택하는 것이 더 적합한지 예를 들어 보세요.

---

[◀ 이전: 14장. 확장 함수](ch14-확장함수.md) | [📖 목차](00-목차.md) | [다음: 16장. 코루틴 기초 ▶](ch16-코루틴기초.md)
