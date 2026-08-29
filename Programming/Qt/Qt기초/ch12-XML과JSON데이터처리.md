# 12장. XML과 JSON 데이터 처리

📖 [◀ 목차](00-목차.md) | [◀ 이전: 11장. 파일 입출력과 설정 저장(QSettings)](ch11-파일입출력과설정저장.md) | [다음: 13장. SQL 데이터베이스 연동(Qt SQL) ▶](ch13-SQL데이터베이스연동.md)

---

11장에서는 `QFile`, `QTextStream`, `QDataStream`으로 파일을 읽고 쓰는 방법과, `QSettings`로 간단한 키-값 설정을 저장하는 방법을 배웠습니다. 하지만 저장해야 할 데이터가 단순한 키-값 쌍을 넘어서 목록이나 중첩된 구조(예: "사용자 한 명이 여러 개의 주소를 가진다")를 가진다면, `QSettings`만으로는 표현하기가 번거로워집니다. 이런 구조화된 데이터를 저장하고 다른 프로그램과 교환할 때는 흔히 JSON이나 XML 같은 텍스트 기반 데이터 형식을 사용합니다.

이 장에서는 Qt가 기본으로 제공하는 JSON 처리 클래스(`QJsonDocument`, `QJsonObject`, `QJsonArray`, `QJsonValue`)로 데이터를 파싱하고 생성하는 방법을 먼저 익히고, 이어서 XML을 다루는 두 가지 방식인 스트림 기반의 `QXmlStreamReader`/`QXmlStreamWriter`와 트리 기반의 `QDomDocument`를 살펴봅니다. 마지막에는 간단한 할 일 목록을 JSON 파일로 저장하고 다시 불러오는 실전 예제를 만들어 봅니다.

## 12.1 JSON과 QJsonDocument 개요

JSON(JavaScript Object Notation)은 사람이 읽기 쉬우면서도 파싱하기 단순한 텍스트 기반 데이터 형식입니다. 웹 API 응답, 설정 파일, 애플리케이션 간 데이터 교환 등 오늘날 가장 널리 쓰이는 데이터 형식 중 하나입니다. JSON은 다음 여섯 가지 값 종류만으로 어떤 구조든 표현합니다.

| JSON 타입 | 예시 | Qt에서 대응하는 값 |
|---|---|---|
| 객체(object) | `{"name": "Alice"}` | `QJsonObject` |
| 배열(array) | `[1, 2, 3]` | `QJsonArray` |
| 문자열(string) | `"Alice"` | `QString` |
| 숫자(number) | `42`, `3.14` | `double` |
| 불리언(boolean) | `true`, `false` | `bool` |
| null | `null` | (값 없음) |

Qt는 이 여섯 가지 값을 `QJsonValue`라는 하나의 클래스로 통합해서 표현합니다. `QJsonValue`는 내부적으로 어떤 타입의 값을 담고 있는지 `QJsonValue::Type` 열거형(`Null`, `Bool`, `Double`, `String`, `Array`, `Object`, `Undefined`)으로 구분하며, `isString()`, `isDouble()`, `isBool()`, `isObject()`, `isArray()`, `isNull()`, `isUndefined()` 같은 판별 함수와 `toString()`, `toDouble()`, `toBool()`, `toObject()`, `toArray()` 같은 변환 함수를 제공합니다.

JSON 문서 전체를 나타내는 최상위 클래스는 `QJsonDocument`입니다. JSON 문서의 최상위는 객체이거나 배열이어야 하므로, `QJsonDocument`는 `QJsonObject` 하나 또는 `QJsonArray` 하나를 감싸는 래퍼(wrapper) 역할을 합니다.

이 장에서 다루는 JSON 관련 클래스들(`QJsonDocument`, `QJsonObject`, `QJsonArray`, `QJsonValue`, `QJsonParseError`)은 모두 QtCore 모듈에 포함되어 있습니다. 즉 `QT += sql`이나 `find_package(... COMPONENTS Sql)`처럼 별도의 모듈을 추가할 필요 없이, 헤더만 포함하면 바로 사용할 수 있습니다.

```cpp
#include <QJsonDocument>
#include <QJsonObject>
#include <QJsonArray>
#include <QJsonValue>
```

## 12.2 QJsonObject와 QJsonArray로 JSON 만들기

`QJsonObject`는 문자열 키와 `QJsonValue` 값의 쌍을 저장하는 컨테이너입니다. `insert()`나 `operator[]`로 값을 채워 넣습니다.

```cpp
#include <QJsonDocument>
#include <QJsonObject>
#include <QJsonArray>
#include <QDebug>

void buildJsonExample()
{
    QJsonObject person;
    person["name"] = QString("김철수");
    person["age"] = 28;
    person["isStudent"] = false;

    QJsonArray hobbies;
    hobbies.append("독서");
    hobbies.append("등산");
    person["hobbies"] = hobbies;

    QJsonDocument doc(person);
    const QByteArray bytes = doc.toJson(QJsonDocument::Indented);

    qDebug().noquote() << bytes;
}
```

`QJsonObject::operator[]`는 `QJsonValueRef`를 반환하는데, 이 참조에 `QString`, `int`, `bool`, `double`, `QJsonArray`, `QJsonObject` 등을 대입하면 자동으로 `QJsonValue`로 변환되어 저장됩니다. `insert(key, value)`를 명시적으로 호출해도 동일하게 동작하며, 어느 쪽을 쓰든 결과는 같습니다.

`QJsonDocument::toJson()`은 `QJsonDocument::JsonFormat` 열거형 값을 인자로 받습니다.

- `QJsonDocument::Indented`(기본값): 사람이 읽기 좋게 들여쓰기와 줄바꿈을 넣은 형식.
- `QJsonDocument::Compact`: 불필요한 공백을 모두 제거한 한 줄짜리 형식. 네트워크로 전송할 때 데이터 크기를 줄이는 데 유리합니다.

위 예제를 실행하면 다음과 비슷한 결과가 출력됩니다.

```json
{
    "age": 28,
    "hobbies": [
        "독서",
        "등산"
    ],
    "isStudent": false,
    "name": "김철수"
}
```

`QJsonObject`는 내부적으로 키를 정렬된 상태로 유지하므로, 삽입한 순서와 상관없이 키 이름 순서대로 출력된다는 점에 유의해야 합니다. 순서가 중요한 데이터라면 배열(`QJsonArray`)을 사용해야 합니다.

중첩 배열과 객체도 자유롭게 조합할 수 있습니다. 예를 들어 사람 여러 명을 담은 배열은 다음과 같이 만듭니다.

```cpp
QJsonArray people;

QJsonObject p1;
p1["name"] = "김철수";
p1["age"] = 28;
people.append(p1);

QJsonObject p2;
p2["name"] = "이영희";
p2["age"] = 25;
people.append(p2);

QJsonDocument doc(people);   // 최상위가 객체가 아니라 배열인 문서
```

## 12.3 QJsonDocument::fromJson()으로 JSON 파싱하기

JSON 텍스트를 Qt 객체로 되돌리려면 `QJsonDocument::fromJson()` 정적 함수를 사용합니다. 이 함수는 `QByteArray`를 입력으로 받고 `QJsonDocument`를 반환합니다.

```cpp
#include <QJsonDocument>
#include <QJsonObject>
#include <QJsonArray>
#include <QJsonValue>
#include <QDebug>

void parseJsonExample()
{
    const QByteArray json = R"({
        "name": "김철수",
        "age": 28,
        "hobbies": ["독서", "등산"]
    })";

    QJsonDocument doc = QJsonDocument::fromJson(json);

    if (doc.isObject()) {
        QJsonObject obj = doc.object();

        const QString name = obj.value("name").toString();
        const int age = obj["age"].toInt();

        qDebug() << "이름:" << name << ", 나이:" << age;

        const QJsonArray hobbies = obj["hobbies"].toArray();
        for (const QJsonValue &hobby : hobbies)
            qDebug() << "취미:" << hobby.toString();
    }
}
```

파싱 결과를 다룰 때 몇 가지 기억해 둘 점이 있습니다.

- `doc.isObject()` / `doc.isArray()`로 최상위 값이 객체인지 배열인지 먼저 확인하는 것이 안전합니다. 형식이 예상과 다르면 `object()`나 `array()`는 빈 값을 반환할 뿐 예외를 던지지 않습니다.
- `QJsonObject::value(key)`와 `operator[]`는 동작이 같습니다. 키가 존재하지 않으면 `QJsonValue::Undefined` 타입의 값을 반환하므로, 그 값에 `toString()`, `toInt()` 등을 호출해도 프로그램이 죽지 않고 빈 문자열이나 0 같은 기본값이 나옵니다.
- `toInt()`, `toDouble()`, `toBool()`, `toString()` 계열 함수는 모두 "값이 없거나 타입이 다를 때 반환할 기본값"을 두 번째 인자로 받을 수 있습니다. 예를 들어 `obj.value("age").toInt(-1)`은 `age` 키가 없으면 `-1`을 반환합니다.
- `QJsonArray`는 `QList`처럼 범위 기반 `for`문으로 순회할 수 있습니다.

파일에서 읽은 JSON을 파싱하는 흐름은 11장에서 배운 `QFile`과 결합하면 됩니다.

```cpp
#include <QFile>
#include <QJsonDocument>

QJsonDocument loadJsonFile(const QString &path)
{
    QFile file(path);
    if (!file.open(QIODevice::ReadOnly))
        return QJsonDocument();

    const QByteArray data = file.readAll();
    file.close();

    return QJsonDocument::fromJson(data);
}
```

## 12.4 QJsonParseError로 파싱 오류 다루기

`fromJson()`은 잘못된 형식의 JSON을 넘겨받으면 예외를 던지지 않고 그냥 `isNull()`이 `true`인 빈 `QJsonDocument`를 반환합니다. 이것만으로는 "데이터가 원래 비어 있었던 것"인지 "형식이 잘못되어 파싱에 실패한 것"인지 구분할 수 없습니다. 오류의 원인과 위치를 알고 싶다면 `fromJson()`의 두 번째 인자로 `QJsonParseError` 포인터를 넘겨야 합니다.

```cpp
#include <QJsonDocument>
#include <QJsonParseError>
#include <QDebug>

void parseWithErrorHandling()
{
    // "hobbies" 뒤에 콤마가 빠져 있는 잘못된 JSON
    const QByteArray brokenJson = R"({
        "name": "김철수"
        "age": 28
    })";

    QJsonParseError error;
    QJsonDocument doc = QJsonDocument::fromJson(brokenJson, &error);

    if (error.error != QJsonParseError::NoError) {
        qWarning() << "JSON 파싱 실패:" << error.errorString();
        qWarning() << "오류 위치(바이트 오프셋):" << error.offset;
        return;
    }

    // 파싱이 성공했을 때만 doc을 사용한다.
    qDebug() << doc.object();
}
```

`QJsonParseError`는 두 개의 멤버를 갖는 구조체입니다.

- `error`: `QJsonParseError::ParseError` 열거형 값입니다. 오류가 없으면 `QJsonParseError::NoError`이고, 그 밖에 `UnterminatedObject`(객체가 닫히지 않음), `MissingValueSeparator`(콤마 누락), `IllegalValue`(잘못된 값), `UnterminatedString`(문자열이 닫히지 않음) 등 오류 종류에 따른 값이 있습니다.
- `offset`: 오류가 발생한 지점까지의 바이트 오프셋입니다. 긴 JSON 문서에서 어디가 문제인지 찾을 때 유용합니다.

`errorString()`은 오류 종류를 사람이 읽을 수 있는 문장으로 바꿔 줍니다. 사용자가 직접 편집할 수 있는 설정 파일(예: 수작업으로 고친 JSON 설정)을 읽어 들이는 프로그램이라면, 이렇게 오류 위치까지 안내해 주는 것이 사용자 경험에 큰 차이를 만듭니다.

> **주의**: `error.error != QJsonParseError::NoError`로 비교하는 것과, `doc.isNull()`로 확인하는 것은 대부분 같은 결과를 주지만 완전히 동일하지는 않습니다. 빈 입력(`QByteArray()`)을 넘기면 `isNull()`이 `true`가 되면서도 `error.error`는 `NoError`가 아닌 경우가 있으므로, 오류를 명확히 검사하고 싶다면 `QJsonParseError`를 직접 확인하는 쪽이 안전합니다.

## 12.5 실전 예제: 할 일 목록을 JSON 파일로 저장·불러오기

지금까지 배운 내용을 종합해서, 간단한 할 일 목록(Todo List)을 JSON 파일로 저장하고 프로그램을 다시 실행했을 때 불러오는 기능을 만들어 보겠습니다. 저장 위치는 11장에서 배운 `QStandardPaths::AppDataLocation`을 사용합니다.

```cpp
// todoitem.h
#pragma once

#include <QString>
#include <QList>

struct TodoItem
{
    QString title;
    bool done = false;
};

using TodoList = QList<TodoItem>;
```

```cpp
// todostorage.h
#pragma once

#include "todoitem.h"
#include <QString>

class TodoStorage
{
public:
    // 저장 파일의 전체 경로를 반환한다. 폴더가 없으면 생성한다.
    static QString filePath();

    static bool save(const TodoList &todos);
    static TodoList load();
};
```

```cpp
// todostorage.cpp
#include "todostorage.h"

#include <QFile>
#include <QDir>
#include <QStandardPaths>
#include <QJsonDocument>
#include <QJsonObject>
#include <QJsonArray>
#include <QJsonParseError>
#include <QDebug>

QString TodoStorage::filePath()
{
    const QString dir =
        QStandardPaths::writableLocation(QStandardPaths::AppDataLocation);
    QDir().mkpath(dir);
    return dir + "/todos.json";
}

bool TodoStorage::save(const TodoList &todos)
{
    QJsonArray array;
    for (const TodoItem &item : todos) {
        QJsonObject obj;
        obj["title"] = item.title;
        obj["done"] = item.done;
        array.append(obj);
    }

    QJsonObject root;
    root["version"] = 1;
    root["todos"] = array;

    QFile file(filePath());
    if (!file.open(QIODevice::WriteOnly | QIODevice::Truncate)) {
        qWarning() << "설정 파일을 쓸 수 없습니다:" << file.errorString();
        return false;
    }

    QJsonDocument doc(root);
    file.write(doc.toJson(QJsonDocument::Indented));
    file.close();
    return true;
}

TodoList TodoStorage::load()
{
    TodoList result;

    QFile file(filePath());
    if (!file.open(QIODevice::ReadOnly))
        return result;   // 처음 실행이라 파일이 없는 경우: 빈 목록 반환

    const QByteArray data = file.readAll();
    file.close();

    QJsonParseError error;
    const QJsonDocument doc = QJsonDocument::fromJson(data, &error);

    if (error.error != QJsonParseError::NoError) {
        qWarning() << "todos.json 파싱 실패:" << error.errorString()
                    << "(오프셋" << error.offset << ")";
        return result;
    }

    if (!doc.isObject())
        return result;

    const QJsonArray array = doc.object().value("todos").toArray();
    for (const QJsonValue &value : array) {
        const QJsonObject obj = value.toObject();

        TodoItem item;
        item.title = obj.value("title").toString();
        item.done = obj.value("done").toBool(false);
        result.append(item);
    }

    return result;
}
```

사용하는 쪽 코드는 다음과 같이 간단해집니다.

```cpp
#include "todostorage.h"

void demo()
{
    TodoList todos = TodoStorage::load();

    todos.append(TodoItem{"Qt 12장 읽기", false});
    todos.append(TodoItem{"연습문제 풀기", false});

    TodoStorage::save(todos);
}
```

이 구조의 장점은, 데이터 모델(`TodoItem`)과 저장 형식(JSON)이 `TodoStorage` 클래스 하나에 캡슐화되어 있다는 점입니다. 나중에 저장 형식을 JSON에서 XML이나 SQLite(13장)로 바꾸더라도, `TodoStorage`의 구현만 교체하면 되고 `TodoItem`을 사용하는 UI 코드는 전혀 손댈 필요가 없습니다. 이는 11장에서 `QSettings`를 다룰 때와 마찬가지로, 저장 로직을 데이터를 사용하는 코드로부터 분리해 두는 것이 유지보수에 유리하다는 원칙을 다시 보여 줍니다.

## 12.6 XML 개요와 두 가지 처리 방식

XML(eXtensible Markup Language)은 JSON보다 오래된 텍스트 기반 데이터 형식으로, 태그로 데이터를 감싸는 방식을 사용합니다. 오늘날 새로 설계하는 데이터 형식에서는 JSON이 더 많이 쓰이지만, RSS 피드, SVG, 각종 설정 파일, 레거시 기업 시스템의 데이터 교환 형식 등 XML을 반드시 다뤄야 하는 상황은 여전히 많습니다. 앞서 만든 할 일 목록을 XML로 표현하면 다음과 같은 모습이 됩니다.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<todos version="1">
    <todo done="false">
        <title>Qt 12장 읽기</title>
    </todo>
    <todo done="false">
        <title>연습문제 풀기</title>
    </todo>
</todos>
```

Qt는 XML을 다루는 방법을 크게 두 가지로 제공합니다.

- **스트림 기반(`QXmlStreamReader` / `QXmlStreamWriter`)**: 문서를 처음부터 끝까지 한 번만 순차적으로 훑으면서 토큰 단위(시작 태그, 끝 태그, 텍스트 등)로 처리합니다. 문서 전체를 메모리에 트리로 구성하지 않으므로 매우 큰 XML 파일도 적은 메모리로 처리할 수 있습니다. `QXmlStreamReader`와 `QXmlStreamWriter`는 QtCore 모듈에 포함되어 있어 별도의 모듈 추가 없이 사용할 수 있습니다.
- **DOM 기반(`QDomDocument`)**: XML 문서 전체를 메모리 안에 트리 구조로 구성한 뒤, 노드를 자유롭게 탐색하거나 수정할 수 있게 해 줍니다. 문서 전체를 한 번에 메모리에 올려야 하므로 대용량 파일에는 불리하지만, 문서의 특정 부분을 오가며 여러 번 읽거나 구조를 편집해야 하는 경우에는 코드가 더 직관적일 수 있습니다. `QDomDocument`는 QtXml이라는 별도 모듈에 속해 있습니다.

이 장에서는 대부분의 상황에 더 적합하고 Qt 공식 문서에서도 권장하는 스트림 방식을 중심으로 다루고, DOM 방식은 개념과 간단한 사용법만 소개합니다.

`QDomDocument`를 사용하려면 프로젝트 설정에 Xml 모듈을 추가해야 합니다. qmake 프로젝트라면 다음 줄을 추가합니다.

```
QT += xml
```

CMake 기반 프로젝트라면 다음과 같이 링크합니다.

```cmake
find_package(Qt6 REQUIRED COMPONENTS Widgets Xml)
target_link_libraries(myapp PRIVATE Qt6::Widgets Qt6::Xml)
```

## 12.7 QXmlStreamWriter로 XML 생성하기

`QXmlStreamWriter`는 `QIODevice`(주로 `QFile`)에 태그를 순서대로 써 내려가는 방식으로 XML을 생성합니다. `setAutoFormatting(true)`를 설정해 두면 들여쓰기와 줄바꿈을 자동으로 넣어 주므로, 사람이 읽기 좋은 결과물을 얻을 수 있습니다.

```cpp
#include <QFile>
#include <QXmlStreamWriter>
#include "todoitem.h"

bool saveTodosAsXml(const QString &path, const TodoList &todos)
{
    QFile file(path);
    if (!file.open(QIODevice::WriteOnly | QIODevice::Text))
        return false;

    QXmlStreamWriter writer(&file);
    writer.setAutoFormatting(true);

    writer.writeStartDocument();
    writer.writeStartElement("todos");
    writer.writeAttribute("version", "1");

    for (const TodoItem &item : todos) {
        writer.writeStartElement("todo");
        writer.writeAttribute("done", item.done ? "true" : "false");
        writer.writeTextElement("title", item.title);
        writer.writeEndElement();   // todo
    }

    writer.writeEndElement();   // todos
    writer.writeEndDocument();

    file.close();
    return true;
}
```

주요 함수의 역할은 다음과 같습니다.

- `writeStartDocument()` / `writeEndDocument()`: XML 선언(`<?xml version="1.0"?>`)을 쓰고, 문서 작성을 마무리합니다.
- `writeStartElement(name)` / `writeEndElement()`: 시작 태그와 끝 태그를 씁니다. 반드시 짝을 맞춰 호출해야 하며, `writeEndElement()`는 가장 최근에 연 태그를 닫습니다.
- `writeAttribute(name, value)`: 현재 열려 있는 태그에 속성을 추가합니다. `writeStartElement()` 직후, 자식 요소를 쓰기 전에 호출해야 합니다.
- `writeTextElement(name, text)`: `writeStartElement(name)` → `writeCharacters(text)` → `writeEndElement()`를 한 번에 처리하는 편의 함수입니다. `<title>Qt 12장 읽기</title>`처럼 텍스트만 담는 단순한 요소를 쓸 때 편리합니다.
- `writeCharacters(text)`: 현재 요소 안에 텍스트 내용을 씁니다. `<`, `&`, `>` 같은 특수 문자는 자동으로 이스케이프(escape) 처리되므로, 사용자가 입력한 문자열을 그대로 넘겨도 안전합니다.

## 12.8 QXmlStreamReader로 XML 파싱하기

`QXmlStreamReader`는 XML을 처음부터 끝까지 순서대로 읽으면서, 태그를 만날 때마다 "이건 시작 태그다", "이건 텍스트다" 같은 토큰을 하나씩 돌려주는 방식으로 동작합니다. 가장 기본적인 형태는 `readNext()`로 다음 토큰을 읽고, 토큰의 종류를 확인하는 반복문입니다.

```cpp
#include <QFile>
#include <QXmlStreamReader>
#include "todoitem.h"

TodoList loadTodosFromXml(const QString &path)
{
    TodoList result;

    QFile file(path);
    if (!file.open(QIODevice::ReadOnly | QIODevice::Text))
        return result;

    QXmlStreamReader reader(&file);

    while (!reader.atEnd()) {
        reader.readNext();

        if (reader.isStartElement() && reader.name() == QLatin1String("todo")) {
            TodoItem item;

            const QString doneAttr = reader.attributes().value("done").toString();
            item.done = (doneAttr == QLatin1String("true"));

            // <todo> 다음에 나오는 자식 요소들을 계속 읽어 나간다.
            while (!(reader.isEndElement() && reader.name() == QLatin1String("todo"))) {
                reader.readNext();

                if (reader.isStartElement() && reader.name() == QLatin1String("title"))
                    item.title = reader.readElementText();
            }

            result.append(item);
        }
    }

    if (reader.hasError()) {
        qWarning() << "XML 파싱 오류:" << reader.errorString();
        result.clear();
    }

    file.close();
    return result;
}
```

이 코드에서 눈여겨볼 함수들은 다음과 같습니다.

- `readNext()`: 다음 토큰으로 이동하고, `QXmlStreamReader::TokenType` 값(예: `StartElement`, `EndElement`, `Characters`, `StartDocument`, `EndDocument`)을 반환합니다.
- `isStartElement()` / `isEndElement()`: 방금 읽은 토큰이 시작 태그인지 끝 태그인지 확인하는 편의 함수입니다. `tokenType() == QXmlStreamReader::StartElement`와 같은 의미입니다.
- `name()`: 현재 위치한 요소의 태그 이름을 반환합니다. Qt 6에서는 `QStringView`를, Qt 5에서는 `QStringRef`를 반환하는데, 둘 다 `QString`이나 `QLatin1String`과 `==` 비교가 가능하고 `toString()`으로 실제 `QString`을 얻을 수 있습니다.
- `attributes()`: 현재 시작 태그의 속성 목록(`QXmlStreamAttributes`)을 반환합니다. `value("속성이름")`으로 특정 속성 값을 조회합니다.
- `readElementText()`: 현재 요소의 텍스트 내용을 통째로 읽고, 그 요소의 끝 태그까지 자동으로 이동합니다. `<title>Qt 12장 읽기</title>`처럼 텍스트만 담은 단순한 요소를 읽을 때 유용합니다.
- `hasError()` / `errorString()`: 파싱 중 형식이 잘못된 XML을 만나면 `hasError()`가 `true`가 되고, `errorString()`으로 원인을 확인할 수 있습니다.

더 단순한 구조의 XML을 읽을 때는 `readNextStartElement()`라는 편의 함수를 사용해 반복문을 간결하게 만들 수도 있습니다. 이 함수는 다음 시작 태그를 만날 때까지 건너뛰고, 시작 태그를 찾으면 `true`를, 현재 요소가 끝나면 `false`를 반환합니다.

```cpp
QXmlStreamReader reader(&file);

reader.readNextStartElement();   // <todos> 요소로 진입

while (reader.readNextStartElement()) {
    if (reader.name() == QLatin1String("todo")) {
        // <todo> 요소 처리
        reader.skipCurrentElement();   // 세부 파싱이 필요 없다면 통째로 건너뛴다.
    }
}
```

`skipCurrentElement()`는 현재 요소와 그 자식 요소를 모두 건너뛰고 짝이 되는 끝 태그로 이동합니다. 관심 없는 요소를 무시하고 지나갈 때 유용합니다.

## 12.9 QDomDocument로 DOM 방식 다루기

`QDomDocument`는 XML 문서 전체를 메모리 안에 트리로 구성해 주므로, 문서의 여러 부분을 오가며 탐색하거나 특정 노드를 찾아 수정해야 하는 경우에 편리합니다.

```cpp
#include <QFile>
#include <QDomDocument>
#include <QDebug>

void readTodosWithDom(const QString &path)
{
    QFile file(path);
    if (!file.open(QIODevice::ReadOnly))
        return;

    QDomDocument doc;
    QString errorMsg;
    int errorLine = 0, errorColumn = 0;

    if (!doc.setContent(&file, &errorMsg, &errorLine, &errorColumn)) {
        qWarning() << "XML 파싱 실패:" << errorMsg
                    << QString("(줄 %1, 열 %2)").arg(errorLine).arg(errorColumn);
        file.close();
        return;
    }
    file.close();

    const QDomElement root = doc.documentElement();   // <todos>
    qDebug() << "버전:" << root.attribute("version");

    QDomElement todoElem = root.firstChildElement("todo");
    while (!todoElem.isNull()) {
        const bool done = (todoElem.attribute("done") == "true");
        const QDomElement titleElem = todoElem.firstChildElement("title");
        const QString title = titleElem.text();

        qDebug() << title << (done ? "(완료)" : "(미완료)");

        todoElem = todoElem.nextSiblingElement("todo");
    }
}
```

주요 함수는 다음과 같습니다.

- `QDomDocument::setContent(device, &errorMsg, &errorLine, &errorColumn)`: XML을 파싱해 트리를 구성합니다. 성공하면 `true`, 형식이 잘못되었으면 `false`를 반환하며 오류 메시지와 위치를 함께 얻을 수 있습니다.
- `documentElement()`: 문서의 최상위 요소(루트 노드)를 반환합니다.
- `firstChildElement(tagName)` / `nextSiblingElement(tagName)`: 이름이 일치하는 첫 번째 자식/다음 형제 요소를 찾습니다. 인자를 생략하면 이름과 관계없이 첫 번째 자식/다음 형제를 반환합니다.
- `attribute(name)`: 요소의 속성 값을 문자열로 반환합니다.
- `text()`: 요소가 담고 있는 텍스트 내용을 반환합니다.

DOM 트리를 만들어서 XML 문자열로 출력하는 것도 가능합니다.

```cpp
QDomDocument doc;
QDomElement root = doc.createElement("todos");
doc.appendChild(root);

QDomElement todoElem = doc.createElement("todo");
todoElem.setAttribute("done", "false");

QDomElement titleElem = doc.createElement("title");
titleElem.appendChild(doc.createTextNode("Qt 12장 읽기"));
todoElem.appendChild(titleElem);

root.appendChild(todoElem);

qDebug().noquote() << doc.toString(4);   // 들여쓰기 4칸으로 문자열 출력
```

스트림 방식과 DOM 방식의 차이를 정리하면 다음과 같습니다.

| 구분 | QXmlStreamReader/Writer | QDomDocument |
|---|---|---|
| 처리 방식 | 순차적 토큰 스트림 | 메모리 내 트리 구조 |
| 메모리 사용량 | 문서 크기와 무관하게 일정 | 문서 전체 크기에 비례 |
| 임의 접근(랜덤 액세스) | 어렵다(한 번 지나간 노드는 되돌아갈 수 없음) | 자유롭다(부모/자식/형제 탐색) |
| 대용량 파일 | 적합 | 부적합할 수 있음 |
| 코드 스타일 | 상태 기계(토큰을 직접 판별) | 트리를 탐색하는 객체지향적 코드 |
| 소속 모듈 | QtCore | QtXml(별도 모듈) |

정리하면, 로그 파일이나 대용량 데이터 내보내기/가져오기처럼 문서를 한 번만 순서대로 처리하면 되는 경우에는 스트림 방식이 메모리 효율과 성능 면에서 유리합니다. 반대로 문서 구조를 자유롭게 오가며 검사하거나 편집해야 하는 도구(예: XML 편집기)를 만든다면 DOM 방식이 더 자연스러운 코드로 이어집니다.

## 요약

- JSON 관련 클래스(`QJsonDocument`, `QJsonObject`, `QJsonArray`, `QJsonValue`)는 QtCore에 포함되어 있으며, `QJsonDocument::fromJson()`으로 파싱하고 `toJson()`으로 직렬화한다.
- `QJsonValue`는 JSON의 여섯 가지 값 타입을 하나의 클래스로 표현하며, `toString()`, `toInt()`, `toBool()`, `toObject()`, `toArray()` 등으로 변환한다. 값이 없거나 타입이 다르면 기본값을 반환할 뿐 예외를 던지지 않는다.
- `QJsonParseError`는 JSON 파싱 실패의 원인(`error`)과 위치(`offset`)를 알려 주며, `errorString()`으로 사람이 읽을 수 있는 오류 메시지를 얻을 수 있다.
- `QStandardPaths`(11장)와 JSON 직렬화를 결합하면, 애플리케이션 데이터 디렉터리에 목록형 데이터를 저장하고 불러오는 저장소 클래스를 만들 수 있다.
- XML은 스트림 방식(`QXmlStreamReader`/`QXmlStreamWriter`, QtCore 소속)과 DOM 방식(`QDomDocument`, QtXml 모듈 소속) 두 가지로 다룰 수 있다. 대용량 문서나 순차 처리에는 스트림 방식이, 문서 구조를 자유롭게 탐색/수정해야 할 때는 DOM 방식이 적합하다.
- `QXmlStreamWriter`는 `writeStartElement()`/`writeEndElement()`/`writeAttribute()`/`writeTextElement()` 등으로 태그를 순서대로 기록하며, `QXmlStreamReader`는 `readNext()`로 토큰을 순회하면서 `isStartElement()`, `name()`, `attributes()`, `readElementText()`로 내용을 읽어 낸다.

## 연습문제

1. `QJsonObject`와 `QJsonArray`를 사용해 "이름(name)"과 "점수 목록(scores, 정수 배열)"을 담은 학생 세 명의 데이터를 `QJsonArray`로 구성하고, `QJsonDocument::toJson(QJsonDocument::Compact)`로 한 줄짜리 JSON 문자열을 출력하는 함수를 작성해 보라.
2. 1번에서 만든 JSON 문자열을 다시 `QJsonDocument::fromJson()`으로 파싱하여, 각 학생의 점수 평균을 계산해 출력하는 함수를 작성해 보라. 이때 `scores` 배열이 비어 있는 경우도 예외 없이 처리해야 한다.
3. 12.4절의 `QJsonParseError` 예제를 참고하여, 마지막 항목 뒤에 불필요한 콤마가 붙은 JSON(예: `{"a": 1, "b": 2,}`)을 파싱해 보고 어떤 `ParseError` 값과 오류 메시지가 나오는지 확인해 보라.
4. 12.5절의 `TodoStorage`를 확장하여, 각 `TodoItem`에 `QString dueDate`(마감일) 필드를 추가하고 JSON 저장/불러오기 코드가 이 필드를 함께 처리하도록 수정해 보라. 단, 예전 버전으로 저장된 `todos.json`(마감일 필드가 없는 파일)을 불러올 때도 프로그램이 오류 없이 동작해야 한다.
5. 12.7절과 12.8절의 예제를 참고하여, 12.5절의 `TodoStorage::save()`/`load()`를 JSON 대신 XML(`QXmlStreamWriter`/`QXmlStreamReader`)로 저장/불러오도록 다시 작성해 보라. 그리고 같은 데이터를 JSON과 XML 두 형식으로 각각 저장했을 때 파일 크기가 어떻게 다른지 비교해 보라.


---

📖 [◀ 목차](00-목차.md) | [◀ 이전: 11장. 파일 입출력과 설정 저장(QSettings)](ch11-파일입출력과설정저장.md) | [다음: 13장. SQL 데이터베이스 연동(Qt SQL) ▶](ch13-SQL데이터베이스연동.md)
