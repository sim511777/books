# 14장. 컬렉션과 LINQ

[◀ 이전: 13장. 제네릭](ch13-제네릭.md) | [📖 목차](00-목차.md) | [다음: 15장. 예외 처리 ▶](ch15-예외처리.md)

---

13장에서 제네릭의 원리를 배웠으니, 이제 그 원리 위에 세워진 .NET의 주요 컬렉션 타입들을 제대로 살펴볼 차례입니다. 여기에 더해 컬렉션에 담긴 데이터를 정렬하고 걸러내고 가공하는 강력한 도구인 LINQ(Language Integrated Query)까지 다룹니다. LINQ를 익히면 반복문을 여러 줄 써서 처리하던 작업을 짧고 읽기 쉬운 코드 한두 줄로 표현할 수 있게 됩니다.

## 주요 컬렉션 타입 다시 보기

7장에서 배열과 `List<T>`의 기초를 다뤘습니다. 이번 절에서는 `List<T>`를 포함해 실무에서 자주 쓰이는 컬렉션 세 가지를 비교하며 정리합니다. 어떤 컬렉션을 선택할지는 "데이터를 어떻게 찾고, 어떻게 저장할 것인가"에 따라 달라집니다.

### List&lt;T&gt; — 순서가 있는 목록

`List<T>`는 순서를 유지하면서 값을 저장하고, 인덱스로 특정 위치의 값에 접근할 수 있습니다. 배열과 비슷하지만 크기가 자동으로 늘어난다는 점이 다릅니다.

```csharp
List<string> fruits = new List<string> { "사과", "바나나", "포도" };
fruits.Add("오렌지");
fruits.Remove("바나나");

foreach (string fruit in fruits)
{
    Console.WriteLine(fruit); // 사과, 포도, 오렌지 순서로 출력
}
```

"목록의 순서가 중요하고, 값이 중복돼도 괜찮다면" `List<T>`가 기본 선택지입니다.

### Dictionary&lt;TKey, TValue&gt; — 키로 값을 찾는 저장소

`Dictionary<TKey, TValue>`는 키(key)와 값(value)을 짝지어 저장하고, 키를 이용해 빠르게 값을 찾아내는 컬렉션입니다. 전화번호부에서 이름으로 전화번호를 찾는 것과 비슷한 구조입니다.

```csharp
Dictionary<string, int> scores = new Dictionary<string, int>
{
    ["철수"] = 90,
    ["영희"] = 85
};

scores["민수"] = 78; // 새 키를 추가
scores["철수"] = 95; // 기존 키의 값을 갱신

if (scores.TryGetValue("영희", out int score))
{
    Console.WriteLine($"영희의 점수: {score}");
}

foreach (KeyValuePair<string, int> pair in scores)
{
    Console.WriteLine($"{pair.Key} : {pair.Value}");
}
```

`scores["존재하지않는키"]`처럼 없는 키에 직접 접근하면 예외가 발생하므로, 키가 있는지 확실치 않을 때는 `TryGetValue`를 사용하는 것이 안전합니다. "이름으로 값을 빠르게 찾고 싶다"면 `Dictionary`가 알맞은 선택입니다.

### HashSet&lt;T&gt; — 중복 없는 집합

`HashSet<T>`는 같은 값을 두 번 저장하지 않는 컬렉션입니다. 순서는 보장하지 않지만, "이 값이 이미 있는가"를 매우 빠르게 확인할 수 있다는 장점이 있습니다.

```csharp
HashSet<string> visitedCities = new HashSet<string>();
visitedCities.Add("서울");
visitedCities.Add("부산");
bool added = visitedCities.Add("서울"); // 이미 있으므로 false가 반환되고 추가되지 않습니다.

Console.WriteLine(visitedCities.Count);       // 2
Console.WriteLine(visitedCities.Contains("부산")); // true
```

"중복을 자동으로 걸러내고 싶다" 또는 "값이 존재하는지만 빠르게 확인하고 싶다"면 `HashSet<T>`가 적합합니다.

## IEnumerable&lt;T&gt; — 모든 컬렉션을 관통하는 개념

`List<T>`, `Dictionary<TKey, TValue>`, `HashSet<T>`, 그리고 배열까지도 공통으로 구현하는 인터페이스가 있습니다. 바로 `IEnumerable<T>`입니다. 이 인터페이스는 "나는 `foreach`로 하나씩 순회할 수 있다"는 능력만을 나타내는 최소한의 계약입니다.

```csharp
void PrintAll(IEnumerable<string> items)
{
    // items가 List든, HashSet이든, 배열이든 상관없이 동작합니다.
    foreach (string item in items)
    {
        Console.WriteLine(item);
    }
}

PrintAll(new List<string> { "a", "b" });
PrintAll(new HashSet<string> { "x", "y" });
PrintAll(new string[] { "1", "2" });
```

`PrintAll` 메서드는 구체적인 컬렉션 타입을 신경 쓰지 않고 "순회 가능한 것"이라면 무엇이든 받을 수 있습니다. 이렇게 특정 구현이 아니라 인터페이스를 기준으로 코드를 작성하면 재사용성이 크게 높아집니다. 그리고 바로 이 `IEnumerable<T>`가 이제부터 다룰 LINQ가 동작하는 기반입니다. LINQ의 대부분의 기능은 사실 `IEnumerable<T>`에 대해 정의된 확장 메서드일 뿐입니다.

## LINQ로 데이터 다루기

LINQ는 컬렉션에 담긴 데이터를 SQL 쿼리와 비슷한 감각으로 걸러내고, 변형하고, 정렬하고, 집계할 수 있게 해주는 기능입니다. 반복문과 조건문을 직접 조합해서 만들던 로직을 훨씬 선언적인 방식으로 표현할 수 있습니다.

예제로 사용할 간단한 데이터를 준비합니다.

```csharp
class Product
{
    public string Name { get; set; }
    public string Category { get; set; }
    public decimal Price { get; set; }
}

List<Product> products = new List<Product>
{
    new Product { Name = "노트북", Category = "전자기기", Price = 1_500_000 },
    new Product { Name = "마우스", Category = "전자기기", Price = 30_000 },
    new Product { Name = "책상",   Category = "가구",   Price = 200_000 },
    new Product { Name = "의자",   Category = "가구",   Price = 150_000 },
    new Product { Name = "키보드", Category = "전자기기", Price = 80_000 },
};
```

### Where — 조건에 맞는 항목만 걸러내기

```csharp
// 10만원이 넘는 상품만 골라냅니다.
IEnumerable<Product> expensiveProducts =
    products.Where(p => p.Price > 100_000);

foreach (Product p in expensiveProducts)
{
    Console.WriteLine(p.Name); // 노트북, 책상, 의자
}
```

`Where`는 각 항목을 검사해 `true`를 반환하는 항목만 남깁니다. `p => p.Price > 100_000` 부분은 람다 식으로, "매개변수 `p`를 받아서 `p.Price > 100_000`이라는 결과를 돌려주는 함수"를 즉석에서 정의한 것입니다. 람다 식은 16장 델리게이트와 이벤트에서 델리게이트와 함께 더 자세히 다룹니다.

### Select — 각 항목을 원하는 형태로 변형하기

```csharp
// 상품 이름만 뽑아냅니다.
IEnumerable<string> names = products.Select(p => p.Name);

foreach (string name in names)
{
    Console.WriteLine(name);
}
```

`Select`는 각 항목을 다른 값으로 변환합니다. 상품 전체 대신 이름만 필요하다면 `Select`로 그 부분만 뽑아낼 수 있습니다.

### OrderBy — 정렬하기

```csharp
// 가격이 낮은 순으로 정렬합니다.
IEnumerable<Product> byPrice = products.OrderBy(p => p.Price);

// 내림차순으로 정렬하려면 OrderByDescending을 씁니다.
IEnumerable<Product> byPriceDesc = products.OrderByDescending(p => p.Price);
```

### 집계 메서드 — Sum, Count, Average, Max, Min

```csharp
decimal totalPrice = products.Sum(p => p.Price);
int count = products.Count(p => p.Category == "전자기기");
decimal averagePrice = products.Average(p => p.Price);
decimal maxPrice = products.Max(p => p.Price);

Console.WriteLine($"전체 합계: {totalPrice}");
Console.WriteLine($"전자기기 개수: {count}");
Console.WriteLine($"평균 가격: {averagePrice}");
```

### 메서드를 이어붙여 복합적인 질의 만들기

LINQ 메서드들은 대부분 `IEnumerable<T>`를 반환하므로, 마침표(`.`)로 계속 이어붙일 수 있습니다.

```csharp
// "전자기기 카테고리 중, 가격이 낮은 순으로 정렬해서 이름만 가져온다"
List<string> cheapElectronicsNames = products
    .Where(p => p.Category == "전자기기")
    .OrderBy(p => p.Price)
    .Select(p => p.Name)
    .ToList();

foreach (string name in cheapElectronicsNames)
{
    Console.WriteLine(name); // 마우스, 키보드, 노트북
}
```

`Where`로 걸러내고, `OrderBy`로 정렬하고, `Select`로 필요한 값만 뽑아낸 다음, 마지막에 `ToList()`로 실제 `List<string>`으로 변환했습니다. 이렇게 메서드를 연쇄적으로 호출하는 방식을 메서드 문법(method syntax)이라고 부릅니다.

## 쿼리 문법 — SQL과 비슷한 또 다른 표기법

C#은 같은 작업을 SQL과 더 비슷한 모양으로 쓸 수 있는 쿼리 문법(query syntax)도 지원합니다. 바로 위 예제를 쿼리 문법으로 다시 써보면 다음과 같습니다.

```csharp
List<string> cheapElectronicsNames2 =
    (from p in products
     where p.Category == "전자기기"
     orderby p.Price
     select p.Name).ToList();
```

`from p in products`로 순회할 대상을 정하고, `where`로 조건을 걸고, `orderby`로 정렬한 뒤, `select`로 원하는 값을 뽑아냅니다. 결과는 메서드 문법으로 작성한 코드와 완전히 동일합니다. 실제로 C# 컴파일러는 쿼리 문법을 내부적으로 메서드 문법 호출로 변환해서 처리합니다. 즉 두 문법은 겉모습만 다를 뿐 같은 일을 하는 동등한 표현입니다. 이 책에서는 이후로도 메서드 문법을 기본으로 사용하지만, 여러 조건이 복잡하게 얽힌 질의를 짤 때는 쿼리 문법이 더 읽기 편하다고 느끼는 개발자도 많으니 두 가지 모두 알아두면 좋습니다.

## 지연 실행(Deferred Execution) 살짝 맛보기

LINQ에서 꼭 기억해야 할 특징 하나는, `Where`나 `Select` 같은 메서드가 호출되는 즉시 결과를 계산하지 않는다는 점입니다. 이런 메서드들은 "나중에 이 데이터를 순회할 때 어떻게 처리할지"에 대한 계획만 만들어두고, 실제 계산은 `foreach`로 순회하거나 `ToList()`, `Count()` 같은 메서드로 결과를 확정하는 순간에야 이루어집니다. 이를 지연 실행이라고 합니다.

```csharp
List<int> numbers = new List<int> { 1, 2, 3 };

// 이 시점에는 아무 계산도 일어나지 않습니다. "계획"만 만들어질 뿐입니다.
IEnumerable<int> doubled = numbers.Select(n =>
{
    Console.WriteLine($"{n}을 처리 중");
    return n * 2;
});

Console.WriteLine("Select 호출 직후, 아직 아무것도 출력되지 않았습니다.");

// foreach로 실제 순회를 시작하는 이 순간에 계산이 일어납니다.
foreach (int n in doubled)
{
    Console.WriteLine($"결과: {n}");
}
```

위 코드를 실행하면 "Select 호출 직후..." 문장이 먼저 출력되고, 그 다음에야 `1을 처리 중`, `결과: 2`처럼 실제 처리 과정이 출력됩니다. 만약 `doubled` 이후에 `numbers` 리스트의 내용을 바꾸고 나서 `foreach`를 실행하면, 바뀐 내용을 기준으로 다시 계산됩니다. 이 지연 실행 덕분에 LINQ는 불필요한 계산을 미루고, 정말 결과가 필요한 순간에만 데이터를 처리할 수 있습니다. 다만 당장은 "LINQ 메서드는 호출한다고 바로 실행되는 것이 아니라, 실제로 값을 꺼내 쓸 때 실행된다"는 정도만 기억해두면 충분합니다.

## 요약

- `List<T>`는 순서가 있는 목록, `Dictionary<TKey, TValue>`는 키로 값을 찾는 저장소, `HashSet<T>`는 중복이 없는 집합으로, 데이터의 특성에 맞는 컬렉션을 선택하는 것이 중요합니다.
- 모든 주요 컬렉션은 `IEnumerable<T>`라는 공통 인터페이스를 구현하며, `foreach`로 순회할 수 있다는 최소한의 계약을 공유합니다.
- LINQ는 `Where`(필터), `Select`(변형), `OrderBy`(정렬), `Sum`/`Count`/`Average`(집계) 같은 메서드를 조합해 데이터를 선언적으로 처리하게 해줍니다.
- 메서드 문법(`products.Where(...).Select(...)`)과 쿼리 문법(`from p in products where ... select ...`)은 표기만 다를 뿐 동등하며, 컴파일러가 쿼리 문법을 메서드 문법으로 변환합니다.
- LINQ의 많은 메서드는 지연 실행됩니다. 즉 실제 값이 필요해지는 시점(`foreach`, `ToList()` 등)에야 계산이 이루어집니다.

## 연습문제

1. 학생 이름과 점수를 담는 `Student` 클래스를 만들고, `List<Student>`에 5명의 데이터를 채운 뒤, LINQ의 `Where`와 `OrderByDescending`을 사용해 "60점 이상인 학생을 점수 높은 순으로" 출력해보세요.
2. 위 문제에서 만든 학생 목록을 `Dictionary<string, int>`(이름을 키, 점수를 값으로)로 변환해보세요. LINQ의 `ToDictionary` 메서드를 검색해서 활용해도 좋습니다.
3. 정수로 이루어진 배열에서 중복된 값을 제거하려고 합니다. `HashSet<int>`를 이용하는 방법과 LINQ의 `Distinct()` 메서드를 이용하는 방법 두 가지로 각각 구현하고 결과를 비교해보세요.
4. 메서드 문법으로 "가격이 5만원 이상인 상품의 이름과 가격을 가격 오름차순으로 출력"하는 코드를 작성한 뒤, 같은 로직을 쿼리 문법으로도 작성해 두 코드를 비교해보세요.
5. `Select` 안에 `Console.WriteLine`을 넣어 지연 실행을 확인하는 예제를 직접 실행해보고, `foreach`를 두 번 실행하면 처리 과정이 몇 번 출력되는지 관찰해보세요. 왜 그런 결과가 나오는지 설명해보세요.

---

[◀ 이전: 13장. 제네릭](ch13-제네릭.md) | [📖 목차](00-목차.md) | [다음: 15장. 예외 처리 ▶](ch15-예외처리.md)
