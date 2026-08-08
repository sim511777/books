# 12장. Maybe와 Either를 이용한 에러 처리

[◀ 이전: 11장. 타입클래스 기초](ch11-타입클래스기초.md) | [📖 목차](00-목차.md) | [다음: 13장. 지연 평가 ▶](ch13-지연평가.md)

---

리스트에서 원소를 찾거나, 나눗셈을 하거나, 사용자 입력을 검증하는 함수는 언제나 "실패할 가능성"을 안고 있다. 찾는 값이 없을 수도 있고, 0으로 나누려 할 수도 있고, 입력이 유효하지 않을 수도 있다. 많은 언어는 이런 상황을 `null`을 반환하거나 예외(exception)를 던지는 방식으로 처리하지만, 둘 다 함정이 있다. `null`은 타입 시그니처만 봐서는 함수가 실패할 수 있다는 사실을 알려주지 않고, 예외는 프로그램 흐름을 눈에 보이지 않게 끊어버린다. Haskell은 이 문제를 전혀 다른 방식으로 푼다. "실패할 수 있다"는 사실 자체를 **타입**으로 표현하는 것이다. 이 장에서는 그 역할을 하는 두 타입, `Maybe`와 `Either`를 다룬다.

## 실패할 수 있는 계산: 문제 제기

리스트의 첫 원소를 꺼내는 `head` 함수를 보자.

```haskell
ghci> head [1, 2, 3]
1
ghci> head ([] :: [Int])

<interactive>: Prelude.head: empty list
```

빈 리스트에 `head`를 쓰면 프로그램이 그대로 멈춘다. 이런 함수를 **완전하지 않은 함수(partial function)**라고 부른다. 입력값의 일부(빈 리스트)에 대해서는 정의되어 있지 않다는 뜻이다. 문제는 `head`의 타입 시그니처 `head :: [a] -> a`만 봐서는 이런 위험을 전혀 알 수 없다는 점이다. 호출하는 쪽에서는 실행해보기 전까지 실패 가능성을 알아챌 방법이 없다.

리스트에서 특정 조건을 만족하는 원소를 찾는 함수를 직접 만든다고 해보자.

```haskell
findEven :: [Int] -> Int
findEven [] = error "짝수를 찾지 못했다"
findEven (x:xs)
  | even x = x
  | otherwise = findEven xs
```

`error`를 호출하면 `head []`와 똑같이 프로그램이 강제로 종료된다. 실패를 표현하고 싶은데, 표현할 방법이 "터뜨리는 것"뿐인 상황이다. 이제 이 문제를 타입으로 풀어보자.

## Maybe 타입: 값이 있을 수도, 없을 수도 있다

Haskell 표준 라이브러리(Prelude)에는 다음과 같이 정의된 타입이 있다.

```haskell
data Maybe a = Nothing | Just a
```

이 선언은 10장에서 직접 만들어본 대수적 데이터 타입과 완전히 같은 모양이다. `Maybe`는 특별한 문법이 아니라, `data`로 정의된 평범한 타입일 뿐이다. 생성자가 두 개인데, `Nothing`은 "값이 없다"를, `Just a`는 "`a` 타입의 값이 있고, 그 값은 이것이다"를 나타낸다. `a`는 타입 매개변수이므로 `Maybe Int`, `Maybe String`, `Maybe Shape`처럼 어떤 타입과도 짝지을 수 있다.

`findEven`을 `Maybe`로 다시 써보자.

```haskell
findEven :: [Int] -> Maybe Int
findEven [] = Nothing
findEven (x:xs)
  | even x = Just x
  | otherwise = findEven xs
```

```haskell
ghci> findEven [1, 3, 4, 5]
Just 4
ghci> findEven [1, 3, 5]
Nothing
```

이제 타입 시그니처 `[Int] -> Maybe Int` 자체가 "이 함수는 실패할 수 있다"는 사실을 정직하게 드러낸다. 호출하는 쪽은 `Maybe Int`를 받았다는 사실만으로 "값이 있을 수도, 없을 수도 있으니 반드시 확인해야 한다"는 것을 컴파일러의 강제를 통해 알게 된다.

리스트에서 특정 키에 대응하는 값을 찾는 `lookup` 함수도 같은 아이디어를 쓴다.

```haskell
ghci> lookup "b" [("a", 1), ("b", 2), ("c", 3)]
Just 2
ghci> lookup "z" [("a", 1), ("b", 2), ("c", 3)]
Nothing
```

나눗셈처럼 특정 입력에서만 실패하는 계산도 `Maybe`로 안전하게 표현할 수 있다.

```haskell
safeDivide :: Double -> Double -> Maybe Double
safeDivide _ 0 = Nothing
safeDivide x y = Just (x / y)
```

```haskell
ghci> safeDivide 10 2
Just 5.0
ghci> safeDivide 10 0
Nothing
```

## Either 타입: 실패 이유까지 담기

`Maybe`의 한계는 실패했을 때 "왜 실패했는지"를 말해주지 못한다는 점이다. `Nothing`은 그저 "없다"고만 할 뿐이다. 실패 이유를 함께 담고 싶을 때는 `Either`를 쓴다.

```haskell
data Either a b = Left a | Right b
```

역시 10장 스타일의 대수적 데이터 타입이며, 타입 매개변수가 두 개(`a`, `b`)라는 점만 `Maybe`와 다르다. 관례상 `Left`에는 실패와 그 이유를, `Right`에는 성공한 결과를 담는다. "옳다(right)"는 뜻과 "성공"을 뜻하는 `right`가 겹치는 말장난으로 기억하면 외우기 쉽다.

`safeDivide`를 `Either`로 바꿔서 실패 이유를 문자열로 담아보자.

```haskell
safeDivide :: Double -> Double -> Either String Double
safeDivide _ 0 = Left "0으로 나눌 수 없다"
safeDivide x y = Right (x / y)
```

```haskell
ghci> safeDivide 10 2
Right 5.0
ghci> safeDivide 10 0
Left "0으로 나눌 수 없다"
```

여러 가지 이유로 실패할 수 있는 검증 함수에서 `Either`의 장점이 더 잘 드러난다.

```haskell
validateAge :: Int -> Either String Int
validateAge age
  | age < 0 = Left "나이는 음수일 수 없다"
  | age > 150 = Left "나이가 너무 크다"
  | otherwise = Right age
```

```haskell
ghci> validateAge 25
Right 25
ghci> validateAge (-3)
Left "나이는 음수일 수 없다"
ghci> validateAge 200
Left "나이가 너무 크다"
```

`Maybe Int`였다면 세 가지 실패 상황을 전부 `Nothing` 하나로 뭉개버렸을 것이다. `Either String Int`는 실패한 이유를 그대로 값으로 들고 다닐 수 있다.

## 패턴 매칭으로 처리하기

`Maybe`와 `Either` 모두 결국 생성자가 여러 개인 데이터 타입이므로, 4장에서 배운 패턴 매칭으로 값을 꺼내 처리한다. `case` 표현식을 쓰는 것이 가장 직접적인 방법이다.

```haskell
describeResult :: Maybe Int -> String
describeResult result =
  case result of
    Nothing -> "결과가 없다"
    Just n  -> "결과는 " ++ show n ++ "이다"
```

```haskell
ghci> describeResult (findEven [1, 3, 4])
"결과는 4이다"
ghci> describeResult (findEven [1, 3, 5])
"결과가 없다"
```

`Either`도 마찬가지로 `Left`와 `Right`를 패턴으로 나눠 처리한다.

```haskell
report :: Either String Int -> String
report (Left err)  = "오류: " ++ err
report (Right age) = "유효한 나이: " ++ show age
```

```haskell
ghci> report (validateAge 25)
"유효한 나이: 25"
ghci> report (validateAge (-3))
"오류: 나이는 음수일 수 없다"
```

함수 정의에서 등식을 나누는 방식(4장에서 본 패턴 매칭)과 `case`는 완전히 같은 원리다. `Nothing`/`Just`, `Left`/`Right`도 결국 `Shape`의 `Circle`/`Rectangle`과 다를 바 없는, 그냥 생성자일 뿐이라는 점을 기억하면 된다.

## Data.Maybe와 Data.Either의 도우미 함수

매번 `case`를 쓰는 대신, 자주 쓰이는 처리 패턴은 표준 라이브러리 함수로도 준비되어 있다. `Data.Maybe` 모듈의 `fromMaybe`는 "값이 있으면 꺼내고, 없으면 기본값을 쓴다"는 패턴을 한 줄로 줄여준다.

```haskell
import Data.Maybe (fromMaybe)
```

```haskell
ghci> fromMaybe 0 (findEven [1, 3, 4])
4
ghci> fromMaybe 0 (findEven [1, 3, 5])
0
```

`maybe` 함수는 조금 더 일반적이다. "`Nothing`일 때 쓸 기본값"과 "`Just`일 때 값을 변형할 함수"를 함께 받는다.

```haskell
ghci> maybe "없음" show (findEven [1, 3, 4])
"4"
ghci> maybe "없음" show (findEven [1, 3, 5])
"없음"
```

`Either`에도 비슷한 짝인 `either` 함수가 있다. `Left`일 때 쓸 함수와 `Right`일 때 쓸 함수를 각각 넘긴다.

```haskell
ghci> either ("오류: " ++) (\age -> "나이: " ++ show age) (validateAge (-3))
"오류: 나이는 음수일 수 없다"
```

`describeResult`, `report`처럼 직접 `case`로 짠 함수와 결과가 같지만, 자주 나오는 패턴을 라이브러리에 맡기면 코드가 더 짧아진다. 모듈을 불러오는 `import` 구문의 자세한 규칙은 17장에서 다룬다.

## Maybe와 Either, 언제 무엇을 쓸까

`Maybe`는 실패 이유가 하나뿐이거나 굳이 설명할 필요가 없을 때 알맞다. "리스트가 비어 있어서 없다", "찾는 키가 없어서 없다"처럼 이유가 자명한 경우다. `Either`는 실패 원인이 여러 가지이고, 호출한 쪽이 그 이유를 구분해서 다르게 대응해야 할 때 알맞다. 사용자 입력 검증처럼 "어떤 규칙을 어겼는지"가 중요한 상황이 대표적이다.

두 타입 모두 `error`나 예외처럼 프로그램 흐름을 갑자기 끊지 않는다는 공통점이 있다. 실패도 성공과 마찬가지로 평범한 값으로 취급되고, 함수 시그니처만 보면 그 함수가 실패할 수 있는지 없는지를 곧바로 알 수 있다. 지금까지는 `case`와 몇몇 도우미 함수로 값을 직접 꺼내 처리했지만, 이렇게 "값을 감싸고 있는 상자"를 다루는 더 일반적이고 강력한 방법이 14장 펑터와 애플리커티브, 15장 모나드와 IO에서 이어진다. `Maybe`와 `Either`가 바로 그 개념들의 가장 친숙한 첫걸음이다.

## 요약

- `null`이나 예외 대신, Haskell은 실패할 수 있는 계산의 결과를 `Maybe`와 `Either`라는 타입으로 표현해서 함수 시그니처만으로 실패 가능성을 드러낸다.
- `Maybe a`는 `Nothing`(값 없음) 또는 `Just a`(값 있음) 두 생성자를 가지는, 10장에서 배운 것과 같은 형태의 대수적 데이터 타입이다.
- `Either a b`는 `Left a`(대개 실패와 그 이유) 또는 `Right b`(대개 성공한 결과)로, 실패 이유까지 함께 담을 수 있다.
- `Nothing`/`Just`, `Left`/`Right`는 다른 생성자와 마찬가지로 `case` 표현식이나 함수 정의의 패턴 매칭으로 값을 꺼내 처리한다.
- `Data.Maybe`의 `fromMaybe`, `maybe`, `Either`의 `either` 같은 도우미 함수는 자주 나오는 처리 패턴을 짧게 줄여준다.
- 실패 이유가 단순하면 `Maybe`, 실패 이유를 구분해서 다뤄야 하면 `Either`를 선택한다. 이 개념은 14장과 15장의 펑터, 모나드로 이어지는 첫 단계다.

## 연습문제

1. 리스트와 인덱스를 받아 그 위치의 원소를 반환하되, 인덱스가 범위를 벗어나면 `Nothing`을 반환하는 `safeIndex :: [a] -> Int -> Maybe a` 함수를 작성해 보라.

2. `Either String Double` 타입을 반환하며, 음수의 제곱근을 요청하면 `Left "음수는 제곱근을 구할 수 없다"`를, 그렇지 않으면 `Right`에 결과를 담아 반환하는 `safeSqrt` 함수를 작성해 보라. (`sqrt`는 `Prelude`에 이미 있다.)

3. 문제 1에서 만든 `safeIndex`의 결과를 `case`로 처리해, 값이 있으면 `"값: X"`를, 없으면 `"인덱스 범위 밖"`을 반환하는 함수를 작성해 보라. 그다음 같은 동작을 `maybe` 함수로 다시 작성해 두 버전을 비교해 보라.

4. 사용자로부터 이메일 문자열을 받아, `@`가 포함되어 있지 않으면 `Left "잘못된 이메일 형식"`을, 포함되어 있으면 `Right`에 그 문자열을 담아 반환하는 `validateEmail :: String -> Either String String` 함수를 작성해 보라. (``'@' `elem` str``처럼 `elem`을 중위 함수로 써서 문자열에 특정 문자가 포함되어 있는지 확인할 수 있다.)

5. `Maybe`로 표현했을 때와 `Either`로 표현했을 때 각각 어떤 상황에 더 적합한지, 본문에서 다룬 `findEven`과 `validateAge` 두 예제를 근거로 자신의 말로 정리해 보라.

---

[◀ 이전: 11장. 타입클래스 기초](ch11-타입클래스기초.md) | [📖 목차](00-목차.md) | [다음: 13장. 지연 평가 ▶](ch13-지연평가.md)
