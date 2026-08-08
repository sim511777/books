# 17장. Java와의 상호운용

[◀ 이전: 16장. 코루틴 기초](ch16-코루틴기초.md) | [📖 목차](00-목차.md) | [다음: 18장. 좋은 Kotlin 코드 작성 습관 ▶](ch18-좋은코드작성습관.md)

---

1장에서 Kotlin의 가장 실용적인 장점으로 Java와의 상호운용성을 언급했습니다. Kotlin은 JVM 위에서 동작하며 Java와 같은 형태의 바이트코드로 컴파일되기 때문에, 한 프로젝트 안에 Kotlin 파일과 Java 파일을 함께 두고 서로 자유롭게 호출할 수 있습니다. 이 장에서는 Kotlin에서 Java 코드를 호출하는 방법, 반대로 Java에서 Kotlin 코드를 호출할 때 알아둘 점, 그리고 이 상호운용성이 실무에서 왜 중요한지를 살펴봅니다.

## Kotlin에서 Java 클래스 그대로 쓰기

Kotlin에서 Java 클래스를 사용하는 방법은 놀랍도록 단순합니다. `import` 문으로 가져오기만 하면, 마치 원래부터 Kotlin으로 작성된 클래스였던 것처럼 사용할 수 있습니다. 예를 들어 Java 표준 라이브러리의 `ArrayList`나 `StringBuilder`는 모두 Java 클래스이지만, Kotlin 코드에서 다음과 같이 아무 거리낌 없이 사용해 왔습니다.

```kotlin
import java.util.ArrayList

fun main() {
    val names = ArrayList<String>()   // Java 클래스를 그대로 생성
    names.add("서연")
    names.add("민준")

    val builder = StringBuilder()     // 이 역시 Java 클래스
    for (name in names) {
        builder.append(name).append(" ")
    }
    println(builder.toString().trim())
}
```

직접 작성한 Java 클래스도 마찬가지입니다. 같은 프로젝트 안에 다음과 같은 Java 클래스가 있다고 가정해 봅시다.

```java
// Greeter.java
public class Greeter {
    private String name;

    public Greeter(String name) {
        this.name = name;
    }

    public String greet() {
        return "안녕하세요, " + name + "님!";
    }

    public static String defaultGreeting() {
        return "안녕하세요!";
    }
}
```

Kotlin 코드에서는 이 클래스를 다음처럼 그대로 가져와 씁니다.

```kotlin
fun main() {
    val greeter = Greeter("하윤")
    println(greeter.greet())              // 인스턴스 메서드 호출
    println(Greeter.defaultGreeting())    // static 메서드는 클래스 이름으로 바로 호출
}
```

여기서 흥미로운 점이 하나 있습니다. Java의 `getXxx()` / `setXxx()` 형태의 getter·setter는 Kotlin에서 자동으로 프로퍼티처럼 취급됩니다. 예를 들어 Java 클래스에 `getName()`이라는 getter가 있다면, Kotlin 코드에서는 `greeter.getName()`이라고 호출할 수도 있지만 `greeter.name`이라는 프로퍼티 문법으로도 접근할 수 있습니다. 9장에서 배운 Kotlin의 프로퍼티 문법이 Java의 getter·setter 관례와 자연스럽게 이어지도록 컴파일러가 다리를 놓아주는 셈입니다.

## Java에서 Kotlin 클래스 호출하기

반대 방향, 즉 Java 코드에서 Kotlin으로 작성한 클래스를 호출하는 경우도 대부분 매끄럽지만 몇 가지 알아둘 점이 있습니다.

### 파일 최상위 함수는 클래스 뒤에 붙는다

Kotlin에서는 클래스 밖에 함수를 바로 정의할 수 있습니다(7장에서 본 `main` 함수도 사실 최상위 함수입니다). 그런데 Java는 클래스 없이 함수만 존재하는 구조를 허용하지 않기 때문에, Kotlin 컴파일러는 파일 이름을 딴 클래스를 자동으로 만들어 그 안에 최상위 함수들을 담습니다. 예를 들어 `StringUtils.kt` 파일에 다음과 같은 최상위 함수가 있다면,

```kotlin
// StringUtils.kt
fun reverseWords(text: String): String =
    text.split(" ").reversed().joinToString(" ")
```

Java 코드에서는 파일 이름에 `Kt`가 붙은 `StringUtilsKt`라는 클래스의 static 메서드로 호출합니다.

```java
// Java 코드에서 Kotlin 최상위 함수 호출
String result = StringUtilsKt.reverseWords("나는 학교에 간다");
```

파일에 `@file:JvmName("StringHelper")`처럼 애노테이션을 붙이면 생성되는 클래스 이름을 원하는 대로 바꿀 수도 있습니다.

### companion object 멤버는 @JvmStatic으로 static처럼 노출하기

9장과 10장에서 다룬 클래스 안의 `companion object`는 Kotlin에서는 마치 static 멤버처럼 편하게 쓰이지만, 실제로는 companion object라는 별도의 객체 인스턴스에 속한 멤버입니다. 그래서 Java 쪽에서는 기본적으로 `Greeter.Companion.defaultGreeting()`처럼 `Companion`을 한 번 더 거쳐야 합니다.

```kotlin
class Config {
    companion object {
        fun defaultTimeout(): Int = 30
    }
}
```

```java
// @JvmStatic 없이 Java에서 호출하면 Companion을 거쳐야 한다
int timeout = Config.Companion.defaultTimeout();
```

이 번거로움을 없애고 싶다면 companion object의 멤버에 `@JvmStatic`을 붙입니다. 그러면 Java 쪽에서 진짜 static 메서드처럼 클래스 이름으로 바로 호출할 수 있습니다.

```kotlin
class Config {
    companion object {
        @JvmStatic
        fun defaultTimeout(): Int = 30
    }
}
```

```java
// @JvmStatic을 붙이면 Config.Companion 없이 바로 호출 가능
int timeout = Config.defaultTimeout();
```

이런 애노테이션들은 대부분 "Kotlin에서 Java 쪽 사용자를 배려해 API를 더 자연스럽게 다듬는 도구"라고 이해하면 됩니다. 라이브러리를 만들어 Java 사용자에게 배포할 계획이 없다면 당장 외울 필요는 없지만, Java 프로젝트에 Kotlin을 섞어 쓰는 팀에서는 자주 마주치는 개념이므로 이름 정도는 기억해 둘 만합니다.

## 플랫폼 타입: Java의 null 미표시 타입 다루기

8장에서 Kotlin의 null 안전성을 배우면서 `String`과 `String?`이 엄격히 구분된다는 점을 강조했습니다. 그런데 Java에는 이런 구분이 없습니다. Java의 타입 시스템은 어떤 값이 `null`일 수 있는지 컴파일러 수준에서 표시하지 않기 때문입니다. 그렇다면 Kotlin 코드에서 Java 메서드를 호출했을 때 돌아오는 값은 널러블일까요, 아닐까요?

이 질문에 답하기 위해 Kotlin은 "플랫폼 타입(platform type)"이라는 특별한 취급을 도입했습니다. Java 코드에서 넘어온 타입은 Kotlin 컴파일러가 널러블 여부를 알 수 없으므로, 컴파일러는 일단 그 타입을 널러블일 수도 있고 아닐 수도 있는 상태로 남겨둡니다. 화면에는 `String!`처럼 느낌표가 붙은 형태로 표시되는데, 이 느낌표는 실제 문법이 아니라 IDE나 문서에서 "이건 플랫폼 타입이라 확신할 수 없다"는 것을 알리는 표시일 뿐입니다.

```java
// Legacy.java
public class Legacy {
    public static String getNickname() {
        return null;   // Java는 이 반환값에 null 가능성을 타입으로 표시할 방법이 없다
    }
}
```

```kotlin
fun main() {
    val nickname = Legacy.getNickname()   // 타입은 String! (플랫폼 타입)

    // 컴파일러는 막지 않지만, 실제로 null이 들어오면 여기서 NPE가 발생한다
    println(nickname.length)
}
```

위 코드는 컴파일 시점에는 아무 경고도 없이 통과합니다. Kotlin 컴파일러가 `nickname`을 널이 아닌 `String`으로도, 널러블한 `String?`으로도 취급할 수 있게 열어두었기 때문입니다. 하지만 실행 시점에 `Legacy.getNickname()`이 실제로 `null`을 반환하면, `nickname.length`에서 다름 아닌 `NullPointerException`이 그대로 발생합니다. 즉 플랫폼 타입은 Kotlin이 자랑하는 null 안전성의 "사각지대"라고 할 수 있습니다. Java 쪽 코드가 잘못된 것이 아니라, Java의 타입 시스템 자체가 애초에 null 여부를 표현하지 못하기 때문에 생기는 경계 지점의 문제입니다.

이 사각지대를 안전하게 다루는 방법은 두 가지입니다.

첫째, Java 쪽 API에 `@Nullable` / `@NonNull` 같은 애노테이션(JetBrains, javax, Android 등에서 제공하는 것들)이 붙어 있다면, Kotlin 컴파일러가 이를 인식해서 진짜 `String?` 또는 `String`으로 정확히 처리해 줍니다. 가능하다면 이런 애노테이션이 잘 붙어 있는 Java 라이브러리를 쓰는 것이 가장 안전합니다.

둘째, 애노테이션이 없는 애매한 Java API를 다룰 때는, 넘어오자마자 직접 널러블 타입으로 받아 8장에서 배운 안전 호출(`?.`)이나 엘비스 연산자(`?:`)로 다루는 습관을 들이는 것이 좋습니다.

```kotlin
fun main() {
    val nickname: String? = Legacy.getNickname()   // 스스로 널러블로 명시
    val safeName = nickname ?: "익명"                // 엘비스 연산자로 기본값 지정
    println("닉네임: $safeName")
}
```

이렇게 플랫폼 타입을 받는 즉시 스스로 널러블 여부를 판단해 타입을 명시하면, 이후 코드에서는 다시 Kotlin의 null 안전성 규칙을 온전히 누릴 수 있습니다. Java와 상호운용하는 코드를 작성할 때는 "이 값이 Java에서 왔다면 일단 null일 수도 있다고 의심하고, 경계에서 한 번 확인한다"는 원칙을 기억해 두면 좋습니다.

## 왜 상호운용성이 중요한가: 점진적 도입

Kotlin이 실무에서 널리 채택될 수 있었던 배경에는 이 상호운용성이 큰 역할을 했습니다. 수년간 쌓아온 대규모 Java 코드베이스를 하루아침에 Kotlin으로 전부 새로 작성하는 것은 현실적으로 거의 불가능합니다. 하지만 Kotlin은 그럴 필요가 없도록 설계되었습니다.

- 기존 Java 프로젝트에 Kotlin 컴파일러 설정만 추가하면, Java 파일과 Kotlin 파일이 같은 모듈 안에 공존할 수 있습니다.
- 새로 작성하는 클래스나 기능부터 Kotlin으로 작성하고, 기존 Java 코드는 그대로 두어도 서로 문제없이 호출할 수 있습니다.
- 버그를 수정하거나 리팩터링할 일이 생긴 기존 Java 클래스를, 그 시점에 Kotlin으로 옮겨 다시 작성할 수 있습니다. 이때 앞서 배운 데이터 클래스(10장), null 안전성(8장), 확장 함수(14장) 같은 기능을 적용하면 같은 로직도 훨씬 짧고 안전하게 다시 쓸 수 있는 경우가 많습니다.
- 팀 전체가 한 번에 새 언어를 익힐 필요 없이, 관심 있는 개발자부터 점진적으로 Kotlin에 익숙해질 수 있습니다.

이런 특징 때문에 Kotlin은 흔히 "Java를 대체하는 언어"라기보다는 "Java 생태계 위에서 점진적으로 채택할 수 있는 개선된 언어"로 설명됩니다. 실제로 많은 조직이 신규 코드는 Kotlin으로, 안정적으로 돌아가는 기존 Java 코드는 그대로 유지하는 방식으로 두 언어를 함께 운영하고 있습니다.

## 요약

- Kotlin에서는 Java 클래스를 `import`만으로 그대로 사용할 수 있으며, Java의 getter·setter는 Kotlin 프로퍼티 문법으로도 접근할 수 있습니다.
- Kotlin 파일의 최상위 함수는 Java에서 파일 이름 뒤에 `Kt`가 붙은 클래스의 static 메서드로 노출됩니다.
- companion object의 멤버는 기본적으로 `Companion`을 거쳐야 Java에서 호출할 수 있으며, `@JvmStatic`을 붙이면 진짜 static 메서드처럼 바로 호출할 수 있습니다.
- Java 코드에서 넘어온, null 여부가 타입에 표시되지 않은 타입을 Kotlin은 "플랫폼 타입"으로 취급합니다. 컴파일러가 널 여부를 확신할 수 없어 검사를 강제하지 않으므로, 경계에서 직접 널러블 여부를 판단하고 8장의 안전 호출·엘비스 연산자로 다루는 습관이 중요합니다.
- Kotlin과 Java의 상호운용성 덕분에 기존 Java 프로젝트를 한 번에 새로 작성하지 않고도 파일 단위, 팀원 단위로 점진적으로 Kotlin을 도입할 수 있습니다.

## 연습문제

1. Java로 작성된 `Calculator` 클래스에 `public int add(int a, int b)`와 `public static int version()`이라는 static 메서드가 있다고 가정하고, 이 클래스를 Kotlin `main` 함수에서 호출하는 코드를 작성해 보세요.
2. `MathUtils.kt`라는 파일에 최상위 함수 `fun square(x: Int) = x * x`를 정의했을 때, Java 코드에서 이 함수를 호출하려면 어떤 클래스 이름을 통해야 하는지 설명해 보세요.
3. companion object 안에 `fun create(): Config`라는 메서드가 있는 Kotlin 클래스가 있습니다. `@JvmStatic`을 붙이기 전과 후, Java 코드에서 이 메서드를 호출하는 방식이 각각 어떻게 달라지는지 코드로 비교해 보세요.
4. Java 메서드 `String findUserName(int id)`가 사용자가 없을 때 `null`을 반환할 수 있다고 가정합니다. 이 메서드를 Kotlin에서 호출한 결과를 담을 때, "플랫폼 타입"이 어떤 위험을 가질 수 있는지 설명하고, 엘비스 연산자를 이용해 안전하게 처리하는 코드를 작성해 보세요.
5. Kotlin과 Java의 상호운용성이 실무 프로젝트에서 어떤 실용적 장점을 주는지, 이 장에서 배운 내용을 바탕으로 자신의 말로 세 문장 이상 정리해 보세요.

---

[◀ 이전: 16장. 코루틴 기초](ch16-코루틴기초.md) | [📖 목차](00-목차.md) | [다음: 18장. 좋은 Kotlin 코드 작성 습관 ▶](ch18-좋은코드작성습관.md)
