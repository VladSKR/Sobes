# JDBC, JPA, Hibernate и Spring Data JPA

## За две минуты

- **JDBC** — Java API работы с соединением, SQL и результатом.
- **JPA** (Jakarta Persistence в современных версиях) — стандарт отображения сущностей и работы через `EntityManager`; **Hibernate** — одна из реализаций.
- **Spring Data JPA** — надстройка с интерфейсами репозиториев поверх JPA. Это не линейная цепочка наследования всех пяти технологий.

## От прямого SQL к сущности

```java
try (Connection c = dataSource.getConnection();
     PreparedStatement ps = c.prepareStatement(
         "SELECT name FROM users WHERE id = ?")) {
    ps.setLong(1, id);
    try (ResultSet rs = ps.executeQuery()) {
        if (rs.next()) return rs.getString("name");
    }
}
```

`PreparedStatement` отделяет значения параметров от SQL; не склеивай пользовательский ввод в запрос. `Connection` управляет границей JDBC-транзакции (например, `commit`/`rollback` при выключенном autocommit). В Spring соединением обычно управляет транзакционная инфраструктура, а не ручной `DriverManager` в сервисе.

В JPA `EntityManager` хранит persistence context — набор managed entities внутри текущего контекста. `find` может выполнить SELECT, `persist` делает новый объект managed, `remove` помечает на удаление, `merge` возвращает managed копию состояния detached entity. Не трактуй `merge` как немедленный `UPDATE`: SQL обычно отправляется при flush, а commit — отдельное действие. Изменение поля managed entity отслеживается через dirty checking и может попасть в SQL без `repository.save`.

```java
@Transactional
public void rename(long id, String name) {
    User u = entityManager.find(User.class, id);
    u.setName(name); // dirty checking -> flush -> commit
}
```

`JpaRepository<User, Long>` даёт `findById`, `save`, `delete`, paging и производные запросы. Репозиторий не отменяет принципов JPA: следи за LAZY, N+1, состоянием entity и границей транзакции. `JOIN FETCH`/`EntityGraph` могут помочь с N+1, но `JOIN FETCH` коллекций вместе с обычной пагинацией требует особой осторожности. [[Spring/Core/Transactions-and-JPA]] · [[Spring/Data/Hibernate-Batch-Insert]].

## Если нужно глубже: mapping и репозитории

```java
@Entity
class User {
    @Id @GeneratedValue(strategy = GenerationType.SEQUENCE)
    Long id;
    @Column(nullable = false, length = 100)
    String name;
    @ManyToOne(fetch = FetchType.LAZY)
    Department department;
}

interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByName(String name);
}
```

`@Entity` сопоставляет объект таблице, `@Id` задаёт ключ, `@ManyToOne` описывает связь; конкретные SQL и момент загрузки определяют mapping и запрос. Для явного запроса можно использовать `@Query` с JPQL или native SQL. Для пакетного SQL без ORM `PreparedStatement.addBatch()` накапливает команды и `executeBatch()` отправляет пакет; транзакцию и закрытие ресурсов по-прежнему нужно управлять правильно.

**Проверь себя:** если изменил поле entity, полученной через `find` в транзакции, обязателен ли `save`? Нет; объясни dirty checking, flush и commit.
