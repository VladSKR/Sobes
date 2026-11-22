# JDBC от подключения до транзакции

JDBC (Java Database Connectivity) — стандартный API Java для выполнения SQL. Приложение вызывает интерфейсы из `java.sql`. Драйвер конкретной СУБД реализует эти интерфейсы и общается с сервером. Пройдём путь одного запроса, потом добавим обновления, транзакцию и пакет.

## Драйвер и соединение

URL `jdbc:postgresql://localhost:5432/zoo` состоит из префикса JDBC, имени подпротокола `postgresql` и параметров драйвера: хоста, порта и базы. Для встроенной Derby URL выглядит иначе, например `jdbc:derby:zoo`. Классы драйвера должны быть доступны при запуске. При использовании JDBC 4 драйвер часто обнаруживается как сервис `java.sql.Driver`: в JAR есть `META-INF/services/java.sql.Driver` с полным именем реализации или соответствующее объявление модуля. `DriverManager` ищет подходящий драйвер по URL. Ошибка `No suitable driver` повод проверить и classpath, и URL.

Пример для отдельной Java-программы с JAR драйвера в текущем каталоге. В Windows в classpath вместо `:` используется `;`.

```bash
javac ZooQuery.java
java -cp ".:postgresql.jar" ZooQuery
```

В приложении обычно получают соединение из `DataSource`. За ним может стоять пул, и `close()` вернёт логическое соединение в пул. Основные объекты запроса выглядят так.

| Объект | Назначение |
|---|---|
| `Driver` | Понимает URL и протокол СУБД |
| `Connection` | Представляет соединение и управляет транзакцией |
| `Statement`, `PreparedStatement` | Выполняет SQL. У подготовленного оператора параметры задаются отдельно |
| `ResultSet` | Курсор над возвращёнными строками |

## Выполнение SQL и чтение результата

```java
try (Connection conn = dataSource.getConnection();
     PreparedStatement statement = conn.prepareStatement(
         "SELECT id, name FROM species WHERE habitat_id = ?")) {
    statement.setLong(1, habitatId);
    try (ResultSet rows = statement.executeQuery()) {
        while (rows.next()) {
            System.out.println(rows.getInt("id") + ": " + rows.getString("name"));
        }
    }
}
```

Курсор начинает перед первой строкой. `next()` переводит его к следующей и возвращает `false`, когда строк больше нет. Колонки можно читать по имени или по номеру, начиная с 1. Для SQL NULL примитивный getter вроде `getInt` требует проверки `wasNull()`; для nullable идентификатора подходит `getObject("id", Integer.class)`, если преобразование поддерживает драйвер. Try-with-resources закрывает сначала `ResultSet`, потом statement и затем соединение. Для бинарной колонки можно взять `getBinaryStream()` и передать данные в файл через `transferTo`, пока строка и результат ещё открыты.

| Метод | Для какого результата | Что возвращает |
|---|---|---|
| `executeQuery()` | Один набор строк, например обычный `SELECT` | `ResultSet` |
| `executeUpdate()` | `INSERT`, `UPDATE`, `DELETE` без набора строк или DDL | Число затронутых строк или 0 для команды без счётчика |
| `execute()` | Тип результата заранее неизвестен или результатов несколько | `true`, если первый результат — набор строк. Иначе `false` |

У обычного `Statement` текст SQL передаётся в эти методы. У подготовленного — в `prepareStatement(sql)`. При `execute()` читай результат через `getResultSet()` или `getUpdateCount()`. Значение `-1` у счётчика помогает обнаружить отсутствие очередного результата. Для нескольких результатов используется `getMoreResults()`. Тип возвращаемого результата зависит от SQL: например, PostgreSQL `INSERT … RETURNING` выдаёт строки.

## Режимы и навигация ResultSet

Тип курсора и разрешение на обновление — независимые настройки. Например, `conn.createStatement(ResultSet.TYPE_FORWARD_ONLY, ResultSet.CONCUR_READ_ONLY)` выбирает прямой обход и чтение.

| Тип курсора | Назад или к произвольной строке | Реакция на изменения исходных данных |
|---|---|---|
| `TYPE_FORWARD_ONLY` | Только `next()` | Не полагайся на повторное чтение или обновление снимка |
| `TYPE_SCROLL_INSENSITIVE` | Да | Обычно изменения других операций не отражаются |
| `TYPE_SCROLL_SENSITIVE` | Да | Обычно чувствителен к изменениям, возможности зависят от драйвера |

| Режим | Чтение | Изменение через ResultSet |
|---|---|---|
| `CONCUR_READ_ONLY` | Да | Нет, это стандартный режим по умолчанию |
| `CONCUR_UPDATABLE` | Да | Да для поддерживаемых запросов и драйверов, например через `updateRow()` |

Проверяй сочетание типа и режима через `DatabaseMetaData.supportsResultSetConcurrency()`, фактические параметры — через `getType()` и `getConcurrency()` результата.

| Метод курсора | Переход | Требуется scrollable |
|---|---|---|
| `next()` | Следующая строка | Нет |
| `previous()` | Предыдущая | Да |
| `first()` / `last()` | Первая / последняя | Да |
| `beforeFirst()` / `afterLast()` | Перед первой / после последней | Да |
| `absolute(n)` | К номеру строки, отрицательный номер отсчитывается с конца | Да |
| `relative(n)` | На `n` строк вперёд или назад | Да |

| Тип SQL | Традиционный JDBC-тип | JDBC 4.2 и `java.time` |
|---|---|---|
| `DATE` | `java.sql.Date` | `LocalDate` |
| `TIME` | `java.sql.Time` | `LocalTime` |
| `TIMESTAMP` | `java.sql.Timestamp` | `LocalDateTime` |
| `TIMESTAMP WITH TIME ZONE` | Зависит от драйвера | `OffsetDateTime` у совместимых драйверов |

Пример чтения — `rows.getObject("created_at", OffsetDateTime.class)` для соответствующего SQL-типа. `LocalDateTime` не содержит смещения, поэтому самостоятельно не задаёт момент времени на временной шкале.

## Ошибки, пакет и граница транзакции

`SQLException.getSQLState()` сообщает код состояния, `getErrorCode()` — код поставщика, `getMessage()` — диагностическое сообщение. Например, в PostgreSQL `23505` означает нарушение уникальности. Для нескольких ошибок JDBC есть цепочка `getNextException()`.

Соединение по умолчанию открыто с auto-commit. Две команды без общей транзакции могут зафиксироваться независимо. Если добавление сотрудника прошло, а вставка его обязательного адреса упала, сотрудник останется. Отключи auto-commit, выполни все связанные команды на одном соединении, вызови `commit()` после успеха или `rollback()` при ошибке. Откат делают **до закрытия соединения**.

```java
try (Connection conn = dataSource.getConnection()) {
    conn.setAutoCommit(false);
    try {
        try (PreparedStatement insert = conn.prepareStatement(
                "INSERT INTO employee (id, name) VALUES (?, ?)")) {
            for (Employee employee : employees) {
                insert.setLong(1, employee.id());
                insert.setString(2, employee.name());
                insert.addBatch();
            }
            insert.executeBatch();
        }
        conn.commit();
    } catch (SQLException | RuntimeException error) {
        try {
            conn.rollback();
        } catch (SQLException rollbackError) {
            error.addSuppressed(rollbackError);
        }
        throw error;
    }
}
```

`Statement.addBatch(sql)` принимает несколько текстов SQL. `PreparedStatement.addBatch()` добавляет набор параметров для одного выражения. `executeBatch()` возвращает счётчики в порядке добавления команд. Возможны специальные значения `SUCCESS_NO_INFO` и, при продолжении после ошибки, `EXECUTE_FAILED` в счётчиках `BatchUpdateException`. Драйвер может выполнить часть пакета, прежде чем сообщит о сбое. Целостность группы обеспечивается транзакцией. ORM-пакеты и транзакции Spring строятся поверх этих механизмов для JDBC-хранилищ. [JPA и Hibernate](Overview.md) · [Транзакции Spring](../Core/Transactions-and-JPA.md)

## Когда соединение остаётся занятым

Если метод читает большой `ResultSet`, соединение остаётся занятым до закрытия результата и connection. Даже если строку уже преобразовали в объект, ленивая итерация по курсору продолжает пользоваться JDBC-ресурсами. Поэтому при возврате результатов за пределы метода отдельно проектируй владение курсором и закрытие. Для обычной небольшой выборки собери результат внутри try-with-resources. Для большой проверь fetch size, как драйвер выдаёт порции и сколько времени чтение держит транзакцию.

## Two-Minute Drill

- Java предоставляет JDBC API, драйвер реализует его для конкретной СУБД. При ошибке подключения проверь JAR и префикс URL.
- `Connection` задаёт границу транзакции. `PreparedStatement` связывает значения с параметрами SQL. `ResultSet` начинает перед первой строкой.
- `executeQuery` возвращает строки, `executeUpdate` — счётчик, `execute` сообщает тип первого результата.
- `TYPE_*` задаёт навигацию, `CONCUR_*` — возможность менять данные через курсор. Полагайся на возможности драйвера.
- Batch группирует команды. Одна транзакция обеспечивает атомарность. При сбое сохрани исходное исключение и выполни rollback до закрытия соединения.

**Проверь себя.** Первая команда в batch выполнилась, вторая упала. Почему `executeBatch()` недостаточно для гарантии «всё или ничего»? Выполненные изменения нужно откатить в той же транзакции.

[JDBC API JDK 21](https://docs.oracle.com/en/java/javase/21/docs/api/java.sql/java/sql/package-summary.html) · [JDBC ResultSet](https://docs.oracle.com/en/java/javase/21/docs/api/java.sql/java/sql/ResultSet.html)
