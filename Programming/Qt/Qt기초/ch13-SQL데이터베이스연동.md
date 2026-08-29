# 13장. SQL 데이터베이스 연동(Qt SQL)

📖 [◀ 목차](00-목차.md) | [◀ 이전: 12장. XML과 JSON 데이터 처리](ch12-XML과JSON데이터처리.md) | [다음: 14장. 멀티스레딩(QThread)과 동시성 ▶](ch14-멀티스레딩과동시성.md)

---

## 13.1 QtSql 모듈 개요

많은 데스크톱 애플리케이션은 사용자가 입력한 데이터를 프로그램이 종료된 뒤에도 보존해야 합니다. 앞선 장에서 다룬 `QFile` 기반의 텍스트/바이너리 저장도 하나의 방법이지만, 데이터의 양이 많아지거나 조건을 걸어 검색·정렬해야 하는 경우, 또는 여러 테이블 사이의 관계를 다뤄야 하는 경우에는 SQL 데이터베이스가 훨씬 적합합니다. Qt는 QtSql 모듈을 통해 다양한 데이터베이스 엔진에 통일된 방식으로 접근할 수 있는 API를 제공합니다.

QtSql을 사용하려면 프로젝트 설정에 모듈을 추가해야 합니다. qmake 프로젝트라면 `.pro` 파일에 다음 줄을 추가합니다.

```
QT += sql
```

CMake 기반 프로젝트라면 다음과 같이 `Sql` 컴포넌트를 링크합니다.

```cmake
find_package(Qt6 REQUIRED COMPONENTS Widgets Sql)
target_link_libraries(myapp PRIVATE Qt6::Widgets Qt6::Sql)
```

QtSql의 구조는 크게 세 계층으로 이해할 수 있습니다.

1. **드라이버 계층**: 각 데이터베이스 제품과 통신하는 저수준 드라이버입니다. Qt는 SQLite용 `QSQLITE`, MySQL/MariaDB용 `QMYSQL`, PostgreSQL용 `QPSQL`, ODBC용 `QODBC` 등의 드라이버 플러그인을 제공합니다. `QSQLITE`는 별도의 서버 프로세스 없이 파일 하나로 동작하는 임베디드 데이터베이스이므로, Qt를 설치하면 기본적으로 함께 제공되어 학습과 소규모 애플리케이션에 매우 편리합니다. 반면 `QMYSQL`, `QPSQL` 드라이버는 각 데이터베이스의 클라이언트 라이브러리가 시스템에 설치되어 있어야 하며, 경우에 따라 Qt를 직접 빌드해야 드라이버가 활성화되기도 합니다.
2. **SQL 실행 계층**: `QSqlDatabase`로 연결을 관리하고, `QSqlQuery`로 SQL 문을 직접 실행하는 계층입니다.
3. **모델 계층**: `QSqlQueryModel`, `QSqlTableModel`처럼 SQL 결과를 Qt의 모델/뷰 프레임워크와 연결해 주는 계층입니다. `QTableView` 같은 뷰에 모델을 연결하기만 하면 별도의 코드 없이 데이터를 표 형태로 보여줄 수 있습니다.

이 장에서는 SQLite를 기본 예제로 사용하며, MySQL이나 PostgreSQL을 사용할 때 달라지는 부분은 필요할 때마다 짚고 넘어갑니다.

## 13.2 데이터베이스 연결: QSqlDatabase

`QSqlDatabase`는 하나의 데이터베이스 연결을 표현하는 클래스입니다. 연결은 `addDatabase()` 정적 함수로 생성합니다.

```cpp
#include <QSqlDatabase>
#include <QSqlError>
#include <QDebug>
#include <QStandardPaths>
#include <QDir>

bool openDatabase()
{
    QSqlDatabase db = QSqlDatabase::addDatabase("QSQLITE");

    // 애플리케이션 데이터 폴더에 데이터베이스 파일을 둔다.
    QString dataDir = QStandardPaths::writableLocation(QStandardPaths::AppDataLocation);
    QDir().mkpath(dataDir);
    db.setDatabaseName(dataDir + "/contacts.db");

    if (!db.open()) {
        qWarning() << "데이터베이스 열기 실패:" << db.lastError().text();
        return false;
    }
    return true;
}
```

`addDatabase()`의 첫 번째 인자는 드라이버 이름입니다. MySQL을 사용한다면 다음과 같이 접속 정보를 추가로 지정합니다.

```cpp
QSqlDatabase db = QSqlDatabase::addDatabase("QMYSQL");
db.setHostName("localhost");
db.setPort(3306);
db.setDatabaseName("mydb");
db.setUserName("appuser");
db.setPassword("secret");
if (!db.open()) {
    qWarning() << db.lastError().text();
}
```

PostgreSQL도 드라이버 이름만 `"QPSQL"`로 바꾸면 나머지 API는 거의 동일합니다. 이처럼 QtSql은 데이터베이스 종류가 달라져도 애플리케이션 코드의 대부분을 그대로 유지할 수 있도록 설계되어 있습니다. 다만 SQL 문법 자체(데이터 타입, 함수 이름 등)는 데이터베이스 제품마다 조금씩 다를 수 있으므로 완전히 이식 가능한 것은 아니라는 점은 기억해 두어야 합니다.

`addDatabase()`에 두 번째 인자로 연결 이름을 지정하면 여러 개의 연결을 동시에 유지할 수도 있습니다. 인자를 생략하면 `"qt_sql_default_connection"`이라는 이름의 기본 연결이 사용됩니다.

```cpp
QSqlDatabase db1 = QSqlDatabase::addDatabase("QSQLITE", "conn1");
QSqlDatabase db2 = QSqlDatabase::addDatabase("QSQLITE", "conn2");
```

연결을 더 이상 사용하지 않을 때는 `db.close()`로 닫고, `QSqlDatabase::removeDatabase(connectionName)`으로 연결 객체 자체를 제거합니다. 이때 `QSqlDatabase` 객체가 스코프를 벗어나기 전에 관련된 `QSqlQuery` 객체들이 먼저 소멸되어야 "connection still in use" 경고를 피할 수 있습니다.

## 13.3 QSqlQuery로 쿼리 실행하기

### 13.3.1 테이블 생성과 단순 실행

`QSqlQuery`는 SQL 문을 실행하고 결과를 순회하는 클래스입니다. DDL(테이블 생성 등)처럼 결과 집합이 없는 문장은 `exec()`로 바로 실행할 수 있습니다.

```cpp
#include <QSqlQuery>
#include <QSqlError>

bool createContactsTable()
{
    QSqlQuery query;
    bool ok = query.exec(
        "CREATE TABLE IF NOT EXISTS contacts ("
        "  id INTEGER PRIMARY KEY AUTOINCREMENT,"
        "  name TEXT NOT NULL,"
        "  phone TEXT,"
        "  email TEXT"
        ")");

    if (!ok) {
        qWarning() << "테이블 생성 실패:" << query.lastError().text();
    }
    return ok;
}
```

`QSqlQuery`의 기본 생성자는 앞서 열어 둔 기본 연결을 사용합니다. 특정 연결을 지정하려면 생성자에 `QSqlDatabase` 객체를 전달합니다.

```cpp
QSqlQuery query(db2);
```

### 13.3.2 파라미터 바인딩과 SQL 인젝션 방지

사용자 입력을 SQL 문자열에 직접 이어 붙이면 SQL 인젝션(injection) 공격에 취약해질 뿐 아니라, 작은따옴표가 포함된 값 하나만으로도 문법 오류가 발생할 수 있습니다.

```cpp
// 절대 이렇게 작성하지 않는다.
QString bad = QString("SELECT * FROM contacts WHERE name = '%1'").arg(userInput);
query.exec(bad);
```

대신 `prepare()`와 `bindValue()`(또는 위치 기반 자리표시자 `?`, 이름 기반 자리표시자 `:name`)를 사용해 값을 안전하게 바인딩해야 합니다.

```cpp
bool insertContact(const QString &name, const QString &phone, const QString &email)
{
    QSqlQuery query;
    query.prepare("INSERT INTO contacts (name, phone, email) VALUES (:name, :phone, :email)");
    query.bindValue(":name", name);
    query.bindValue(":phone", phone);
    query.bindValue(":email", email);

    if (!query.exec()) {
        qWarning() << "삽입 실패:" << query.lastError().text();
        return false;
    }
    return true;
}
```

물음표 자리표시자를 사용할 경우 순서대로 `addBindValue()`를 호출합니다.

```cpp
query.prepare("INSERT INTO contacts (name, phone, email) VALUES (?, ?, ?)");
query.addBindValue(name);
query.addBindValue(phone);
query.addBindValue(email);
query.exec();
```

파라미터 바인딩을 사용하면 드라이버가 값의 이스케이프 처리를 책임지므로, 값 안에 작은따옴표나 세미콜론이 들어 있어도 안전하게 처리됩니다. **사용자 입력이 조금이라도 섞이는 쿼리는 반드시 바인딩 방식을 사용해야 합니다.**

### 13.3.3 SELECT 결과 순회

`SELECT` 문을 실행한 뒤에는 `next()`로 커서를 한 행씩 이동시키고, `value()`로 열 값을 꺼냅니다.

```cpp
void printAllContacts()
{
    QSqlQuery query("SELECT id, name, phone, email FROM contacts ORDER BY name");
    while (query.next()) {
        int id = query.value(0).toInt();
        QString name = query.value("name").toString();
        QString phone = query.value(2).toString();
        QString email = query.value(3).toString();

        qDebug() << id << name << phone << email;
    }
}
```

`value()`는 열 인덱스(0부터 시작) 또는 열 이름 문자열을 인자로 받을 수 있으며, 반환 타입은 `QVariant`이므로 `toInt()`, `toString()`, `toDouble()` 등으로 원하는 타입으로 변환합니다.

### 13.3.4 UPDATE와 DELETE

```cpp
bool updatePhone(int id, const QString &newPhone)
{
    QSqlQuery query;
    query.prepare("UPDATE contacts SET phone = :phone WHERE id = :id");
    query.bindValue(":phone", newPhone);
    query.bindValue(":id", id);
    return query.exec();
}

bool deleteContact(int id)
{
    QSqlQuery query;
    query.prepare("DELETE FROM contacts WHERE id = :id");
    query.bindValue(":id", id);
    return query.exec();
}
```

`INSERT` 실행 직후에는 `lastInsertId()`로 방금 생성된 기본 키 값을 얻을 수 있습니다(드라이버가 이 기능을 지원하는 경우).

```cpp
query.exec();
QVariant newId = query.lastInsertId();
```

## 13.4 트랜잭션

여러 개의 SQL 문을 하나의 논리적 작업 단위로 묶어야 할 때는 트랜잭션을 사용합니다. 예를 들어 "계좌 A에서 출금하고 계좌 B에 입금"하는 두 단계는 반드시 둘 다 성공하거나 둘 다 실패해야 합니다. `QSqlDatabase`는 `transaction()`, `commit()`, `rollback()` 세 함수로 트랜잭션을 제어합니다.

```cpp
bool transferBalance(const QString &fromId, const QString &toId, double amount)
{
    QSqlDatabase db = QSqlDatabase::database(); // 기본 연결

    if (!db.transaction()) {
        qWarning() << "트랜잭션 시작 실패:" << db.lastError().text();
        return false;
    }

    QSqlQuery query(db);

    query.prepare("UPDATE accounts SET balance = balance - :amt WHERE id = :id");
    query.bindValue(":amt", amount);
    query.bindValue(":id", fromId);
    if (!query.exec()) {
        db.rollback();
        return false;
    }

    query.prepare("UPDATE accounts SET balance = balance + :amt WHERE id = :id");
    query.bindValue(":amt", amount);
    query.bindValue(":id", toId);
    if (!query.exec()) {
        db.rollback();
        return false;
    }

    if (!db.commit()) {
        qWarning() << "커밋 실패:" << db.lastError().text();
        db.rollback();
        return false;
    }
    return true;
}
```

모든 데이터베이스 드라이버가 트랜잭션을 지원하는 것은 아니므로, `db.driver()->hasFeature(QSqlDriver::Transactions)`로 지원 여부를 미리 확인할 수 있습니다. SQLite와 대부분의 서버형 데이터베이스는 트랜잭션을 지원합니다.

또한 대량의 데이터를 삽입할 때 각 `INSERT`마다 자동 커밋이 일어나면 성능이 크게 저하될 수 있습니다. 이런 경우 트랜잭션으로 여러 삽입을 묶은 뒤 한 번에 커밋하면 속도가 눈에 띄게 향상됩니다.

```cpp
db.transaction();
QSqlQuery query(db);
query.prepare("INSERT INTO logs (message) VALUES (:msg)");
for (const QString &line : logLines) {
    query.bindValue(":msg", line);
    query.exec();
}
db.commit();
```

## 13.5 모델 기반 접근: QSqlQueryModel과 QSqlTableModel

`QSqlQuery`를 직접 다루는 방식은 유연하지만, 결과를 화면에 표로 보여주려면 매번 `QTableWidget`을 채우는 코드를 작성해야 합니다. Qt는 모델/뷰 프레임워크와 QtSql을 연결해 주는 두 가지 모델 클래스를 제공하여 이 과정을 크게 단순화합니다.

### 13.5.1 QSqlQueryModel: 읽기 전용 조회 결과

`QSqlQueryModel`은 임의의 SQL `SELECT` 문 결과를 모델로 감싸는 읽기 전용 모델입니다. 여러 테이블을 조인한 복잡한 조회 결과를 보여줄 때 유용합니다.

```cpp
#include <QSqlQueryModel>
#include <QTableView>

QSqlQueryModel *model = new QSqlQueryModel(this);
model->setQuery("SELECT id, name, phone FROM contacts ORDER BY name");
model->setHeaderData(0, Qt::Horizontal, tr("ID"));
model->setHeaderData(1, Qt::Horizontal, tr("이름"));
model->setHeaderData(2, Qt::Horizontal, tr("전화번호"));

QTableView *view = new QTableView(this);
view->setModel(model);
```

`QSqlQueryModel`은 조회 전용이며, 사용자가 셀을 편집해도 데이터베이스에는 반영되지 않습니다.

### 13.5.2 QSqlTableModel: 편집 가능한 단일 테이블 뷰

특정 테이블 하나를 그대로 편집 가능한 형태로 보여주고 싶다면 `QSqlTableModel`이 더 적합합니다.

```cpp
#include <QSqlTableModel>
#include <QTableView>
#include <QHeaderView>

QSqlTableModel *model = new QSqlTableModel(this);
model->setTable("contacts");
model->setEditStrategy(QSqlTableModel::OnFieldChange);
model->select();

model->setHeaderData(1, Qt::Horizontal, tr("이름"));
model->setHeaderData(2, Qt::Horizontal, tr("전화번호"));
model->setHeaderData(3, Qt::Horizontal, tr("이메일"));

QTableView *view = new QTableView(this);
view->setModel(model);
view->hideColumn(0); // id 열은 사용자에게 숨긴다.
view->horizontalHeader()->setStretchLastSection(true);
```

`setEditStrategy()`는 사용자가 셀을 수정했을 때 언제 데이터베이스에 반영할지를 결정합니다.

- `QSqlTableModel::OnFieldChange`: 셀 편집이 끝나는 즉시(포커스가 다른 셀로 이동하는 순간) 바로 `UPDATE` 문을 실행합니다.
- `QSqlTableModel::OnRowChange`: 편집 중인 행에서 다른 행으로 이동할 때 그 행 전체를 한 번에 반영합니다.
- `QSqlTableModel::OnManualSubmit`: 자동으로 반영하지 않고, 개발자가 명시적으로 `submitAll()`을 호출해야 반영됩니다. 여러 변경 사항을 모았다가 사용자가 "저장" 버튼을 눌렀을 때만 커밋하고 싶은 경우에 적합합니다.

`OnManualSubmit` 전략을 사용할 때는 저장/취소 버튼과 함께 사용하는 것이 일반적입니다.

```cpp
model->setEditStrategy(QSqlTableModel::OnManualSubmit);

// "저장" 버튼 슬롯
void MainWindow::onSaveClicked()
{
    if (!model->submitAll()) {
        qWarning() << "저장 실패:" << model->lastError().text();
        model->database().rollback();
    }
}

// "취소" 버튼 슬롯
void MainWindow::onRevertClicked()
{
    model->revertAll();
}
```

새 행을 추가하거나 행을 삭제할 때도 모델의 함수를 사용합니다.

```cpp
// 새 행 추가
int row = model->rowCount();
model->insertRow(row);
model->setData(model->index(row, 1), "홍길동");
model->setData(model->index(row, 2), "010-1234-5678");

// 행 삭제 (뷰에서 선택된 행 기준)
QModelIndexList selected = view->selectionModel()->selectedRows();
for (const QModelIndex &index : selected) {
    model->removeRow(index.row());
}
```

`OnManualSubmit` 전략에서는 `insertRow()`, `setData()`, `removeRow()` 이후 반드시 `submitAll()`을 호출해야 실제 데이터베이스에 반영됩니다.

`setFilter()`와 `setSort()`를 사용하면 SQL의 `WHERE`, `ORDER BY` 절에 해당하는 조건을 지정할 수 있습니다.

```cpp
model->setFilter("name LIKE '%김%'");
model->setSort(1, Qt::AscendingOrder);
model->select(); // 필터와 정렬을 다시 적용하려면 select()를 다시 호출한다.
```

`setFilter()`에 전달하는 문자열도 결국 SQL의 일부로 조합되므로, 사용자 입력을 그대로 넣는 것은 바람직하지 않습니다. 사용자 입력이 섞인 검색 조건은 값 이스케이프에 주의하거나, `QSqlQueryModel` + 파라미터 바인딩 조합으로 대체하는 것이 더 안전합니다.

## 13.6 예제: 연락처 관리 프로그램

지금까지 배운 내용을 종합하여, SQLite 데이터베이스를 사용하는 간단한 연락처 관리 창을 만들어 보겠습니다. 프로그램은 시작 시 데이터베이스를 열고 테이블이 없으면 생성하며, `QSqlTableModel`을 `QTableView`에 연결하여 표를 통해 직접 추가·수정·삭제를 할 수 있도록 합니다.

```cpp
// contactwindow.h
#pragma once

#include <QMainWindow>

class QSqlTableModel;
class QTableView;
class QPushButton;
class QLineEdit;

class ContactWindow : public QMainWindow
{
    Q_OBJECT

public:
    explicit ContactWindow(QWidget *parent = nullptr);

private slots:
    void addContact();
    void removeSelected();
    void saveChanges();

private:
    bool initDatabase();

    QSqlTableModel *m_model = nullptr;
    QTableView *m_view = nullptr;
    QLineEdit *m_nameEdit = nullptr;
    QLineEdit *m_phoneEdit = nullptr;
    QLineEdit *m_emailEdit = nullptr;
};
```

```cpp
// contactwindow.cpp
#include "contactwindow.h"

#include <QSqlDatabase>
#include <QSqlQuery>
#include <QSqlTableModel>
#include <QSqlError>
#include <QTableView>
#include <QHeaderView>
#include <QLineEdit>
#include <QPushButton>
#include <QLabel>
#include <QMessageBox>
#include <QVBoxLayout>
#include <QHBoxLayout>
#include <QWidget>
#include <QStandardPaths>
#include <QDir>

ContactWindow::ContactWindow(QWidget *parent)
    : QMainWindow(parent)
{
    if (!initDatabase()) {
        QMessageBox::critical(this, tr("오류"), tr("데이터베이스를 열 수 없습니다."));
    }

    m_model = new QSqlTableModel(this);
    m_model->setTable("contacts");
    m_model->setEditStrategy(QSqlTableModel::OnManualSubmit);
    m_model->select();
    m_model->setHeaderData(1, Qt::Horizontal, tr("이름"));
    m_model->setHeaderData(2, Qt::Horizontal, tr("전화번호"));
    m_model->setHeaderData(3, Qt::Horizontal, tr("이메일"));

    m_view = new QTableView(this);
    m_view->setModel(m_model);
    m_view->hideColumn(0);
    m_view->horizontalHeader()->setStretchLastSection(true);
    m_view->setSelectionBehavior(QAbstractItemView::SelectRows);

    m_nameEdit = new QLineEdit(this);
    m_nameEdit->setPlaceholderText(tr("이름"));
    m_phoneEdit = new QLineEdit(this);
    m_phoneEdit->setPlaceholderText(tr("전화번호"));
    m_emailEdit = new QLineEdit(this);
    m_emailEdit->setPlaceholderText(tr("이메일"));

    auto *addButton = new QPushButton(tr("추가"), this);
    auto *removeButton = new QPushButton(tr("삭제"), this);
    auto *saveButton = new QPushButton(tr("저장"), this);

    connect(addButton, &QPushButton::clicked, this, &ContactWindow::addContact);
    connect(removeButton, &QPushButton::clicked, this, &ContactWindow::removeSelected);
    connect(saveButton, &QPushButton::clicked, this, &ContactWindow::saveChanges);

    auto *formLayout = new QHBoxLayout;
    formLayout->addWidget(m_nameEdit);
    formLayout->addWidget(m_phoneEdit);
    formLayout->addWidget(m_emailEdit);
    formLayout->addWidget(addButton);

    auto *buttonLayout = new QHBoxLayout;
    buttonLayout->addStretch();
    buttonLayout->addWidget(removeButton);
    buttonLayout->addWidget(saveButton);

    auto *mainLayout = new QVBoxLayout;
    mainLayout->addLayout(formLayout);
    mainLayout->addWidget(m_view);
    mainLayout->addLayout(buttonLayout);

    auto *central = new QWidget(this);
    central->setLayout(mainLayout);
    setCentralWidget(central);
    resize(600, 400);
    setWindowTitle(tr("연락처 관리"));
}

bool ContactWindow::initDatabase()
{
    QSqlDatabase db = QSqlDatabase::addDatabase("QSQLITE");

    QString dataDir = QStandardPaths::writableLocation(QStandardPaths::AppDataLocation);
    QDir().mkpath(dataDir);
    db.setDatabaseName(dataDir + "/contacts.db");

    if (!db.open()) {
        qWarning() << db.lastError().text();
        return false;
    }

    QSqlQuery query(db);
    return query.exec(
        "CREATE TABLE IF NOT EXISTS contacts ("
        "  id INTEGER PRIMARY KEY AUTOINCREMENT,"
        "  name TEXT NOT NULL,"
        "  phone TEXT,"
        "  email TEXT"
        ")");
}

void ContactWindow::addContact()
{
    const QString name = m_nameEdit->text().trimmed();
    if (name.isEmpty()) {
        QMessageBox::warning(this, tr("입력 오류"), tr("이름을 입력하세요."));
        return;
    }

    int row = m_model->rowCount();
    m_model->insertRow(row);
    m_model->setData(m_model->index(row, 1), name);
    m_model->setData(m_model->index(row, 2), m_phoneEdit->text().trimmed());
    m_model->setData(m_model->index(row, 3), m_emailEdit->text().trimmed());

    m_nameEdit->clear();
    m_phoneEdit->clear();
    m_emailEdit->clear();
}

void ContactWindow::removeSelected()
{
    const QModelIndexList selected = m_view->selectionModel()->selectedRows();
    // 뒤에서부터 지워야 행 번호가 밀리지 않는다.
    QList<int> rows;
    for (const QModelIndex &index : selected) {
        rows.append(index.row());
    }
    std::sort(rows.begin(), rows.end(), std::greater<int>());
    for (int row : rows) {
        m_model->removeRow(row);
    }
}

void ContactWindow::saveChanges()
{
    if (!m_model->submitAll()) {
        QMessageBox::warning(this, tr("저장 실패"), m_model->lastError().text());
        m_model->database().rollback();
    }
}
```

```cpp
// main.cpp
#include <QApplication>
#include "contactwindow.h"

int main(int argc, char *argv[])
{
    QApplication app(argc, argv);

    ContactWindow window;
    window.show();

    return app.exec();
}
```

이 예제는 `OnManualSubmit` 전략을 사용하므로, 표에서 셀을 수정하거나 새 행을 추가/삭제해도 "저장" 버튼을 눌러야 실제로 데이터베이스 파일에 반영됩니다. 만약 즉시 반영을 원한다면 `OnFieldChange`로 전략을 바꾸면 됩니다.

## 13.7 참고: 여러 데이터베이스 드라이버 다루기

실무에서는 개발 중에는 SQLite를 사용하다가 배포 환경에서는 MySQL이나 PostgreSQL 서버를 사용하는 경우가 흔합니다. 이런 경우 데이터베이스 연결 설정 부분만 별도 함수로 분리해 두면, 나머지 SQL 실행 코드(`QSqlQuery`, `QSqlTableModel` 사용부)는 거의 그대로 재사용할 수 있습니다.

```cpp
bool connectToDatabase(const QString &driver, const QString &host,
                        const QString &dbName, const QString &user,
                        const QString &password)
{
    QSqlDatabase db = QSqlDatabase::addDatabase(driver);
    if (driver != "QSQLITE") {
        db.setHostName(host);
        db.setUserName(user);
        db.setPassword(password);
    }
    db.setDatabaseName(dbName);
    return db.open();
}
```

사용 가능한 드라이버 목록은 `QSqlDatabase::drivers()`로 런타임에 확인할 수 있으며, 특정 드라이버가 설치되어 있는지는 `QSqlDatabase::isDriverAvailable("QMYSQL")` 같은 방식으로 점검할 수 있습니다.

## 요약

- QtSql은 `QSqlDatabase`, `QSqlQuery`, `QSqlTableModel`/`QSqlQueryModel` 등을 통해 다양한 SQL 데이터베이스에 통일된 방식으로 접근할 수 있게 해 주는 모듈입니다.
- `QSqlDatabase::addDatabase()`로 드라이버(`QSQLITE`, `QMYSQL`, `QPSQL` 등)를 선택하고 연결 정보를 설정한 뒤 `open()`으로 연결합니다.
- `QSqlQuery`는 `exec()`로 즉시 실행하거나, `prepare()` + `bindValue()`/`addBindValue()`로 값을 안전하게 바인딩한 뒤 `exec()`를 호출합니다. 사용자 입력이 섞이는 쿼리는 반드시 바인딩 방식을 사용해 SQL 인젝션을 방지해야 합니다.
- `QSqlQueryModel`은 임의의 SELECT 결과를 보여주는 읽기 전용 모델이고, `QSqlTableModel`은 특정 테이블 하나를 편집 가능한 형태로 `QTableView`에 연결할 때 사용합니다. 편집 반영 시점은 `setEditStrategy()`(`OnFieldChange`, `OnRowChange`, `OnManualSubmit`)로 제어합니다.
- `QSqlDatabase::transaction()`, `commit()`, `rollback()`으로 여러 SQL 문을 하나의 원자적 작업으로 묶을 수 있으며, 대량 삽입 시 성능 향상에도 도움이 됩니다.

## 연습문제

1. `QSqlQuery`로 사용자 입력을 담은 `SELECT` 쿼리를 작성할 때, 문자열을 직접 이어 붙이는 방식과 `prepare()`/`bindValue()`를 사용하는 방식의 차이를 SQL 인젝션 관점에서 설명하시오.
2. `QSqlTableModel`의 세 가지 편집 전략(`OnFieldChange`, `OnRowChange`, `OnManualSubmit`)의 차이를 설명하고, 각각 어떤 상황에 적합한지 예를 들어 서술하시오.
3. `QSqlQueryModel`과 `QSqlTableModel`의 근본적인 차이는 무엇이며, 여러 테이블을 조인한 조회 결과를 표로 보여주고 싶을 때 어느 쪽을 사용해야 하는지 그 이유와 함께 설명하시오.
4. 은행 계좌 이체처럼 두 개 이상의 UPDATE 문이 반드시 함께 성공하거나 함께 실패해야 하는 상황에서 트랜잭션을 사용하지 않으면 어떤 문제가 발생할 수 있는지 설명하시오.
5. 13.6절의 연락처 관리 예제를 확장하여, 이름으로 검색할 수 있는 검색창을 추가하고 `QSqlTableModel::setFilter()`를 이용해 검색 결과만 표시하도록 구현하시오. 단, 검색어에 작은따옴표가 포함되어도 프로그램이 오류 없이 동작하도록 처리 방법을 함께 설명하시오.


---

📖 [◀ 목차](00-목차.md) | [◀ 이전: 12장. XML과 JSON 데이터 처리](ch12-XML과JSON데이터처리.md) | [다음: 14장. 멀티스레딩(QThread)과 동시성 ▶](ch14-멀티스레딩과동시성.md)
