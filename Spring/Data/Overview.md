# JDBC, JPA, Hibernate и Spring Data JPA

Перед ORM полезно выполнить запрос через драйвер и явно завершить транзакцию. Это разобрано в [главе JDBC](JDBC.md).

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

## Состояние сущности и лишние запросы

У новой `User` ещё нет связи с persistence context. После `persist` контекст управляет объектом. После его закрытия объект обычно detached, поэтому изменение поля само по себе не синхронизируется с базой. Если передать detached-объект в `merge`, используй возвращённый managed-экземпляр. Он может отличаться по идентичности от входного.

Для списка из двадцати пользователей обращение в цикле к лениво загружаемому `department` может вызвать отдельные SELECT для ещё не загруженных департаментов. Это N+1. Для карточки помогает `JOIN FETCH` нужной связи. Для страницы с коллекциями может подойти сначала запрос IDs страницы, затем загрузка нужных данных по IDs. При выборе проверь форму SQL, количество запросов, дубликаты строк и порядок выдачи.

## Проблема N+1: стратегия загрузки связей Hibernate

Пусть у нас 100 заказов, у каждого по 10 строк `order_items`, коллекция `items` — `LAZY`.

```kotlin
val orders = orderRepository.findAll()
orders.forEach { println(it.items.size) }
```

В простом случае Hibernate делает **1 SQL** на заказы и **до 100 SQL** на ленивые коллекции = **101 запрос** (если эти коллекции не были загружены в persistence context/кеше и не используется batch/subselect fetching). Это не повод отказаться от ORM-связей: проблема в плане выборки под конкретный сценарий.

**1. `JOIN FETCH`:**

```kotlin
@Query("select distinct o from OrderEntity o left join fetch o.items")
fun findAllWithItems(): List<OrderEntity>
```

Один SQL join может вернуть **примерно 1000 строк** (100 заказов × 10 items), хотя результирующих корневых сущностей будет 100. `distinct` в JPQL описывает устранение дублей корней на уровне результата; реальный SQL и форма dedup зависят от версии Hibernate. При нескольких коллекциях JOIN может создать декартово произведение.

**2. `@EntityGraph(attributePaths = ["items"])`:** декларативно запросить связь; фактический SQL-план зависит от mapping и запроса, не гарантируй везде «ровно один SQL».

**3. `@BatchSize(size = 25)` или `hibernate.default_batch_fetch_size=25`:** несколько ленивых коллекций могут загружаться в одном `WHERE order_id IN (...)`. Для 100 заказов оценка **1 + 4 SQL** вместо 101, если все коллекции требуют и допускают групповую загрузку. Выбирай размер с учётом DB limits и нагрузки.

**4. DTO projection:** когда API не нужны полноценные managed entities, выбирай необходимые поля и явную агрегацию данных.

### Почему JOIN FETCH + Pageable опасен

```kotlin
@Query("select o from OrderEntity o left join fetch o.items")
fun findPage(pageable: Pageable): Page<OrderEntity>
```

Для `@OneToMany` физические SQL-строки — позиции заказа, а `LIMIT 20` должен означать **20 корневых заказов**. Hibernate может применить пагинацию **в памяти** после чтения множества строк, что опасно для больших объёмов. Для раннего обнаружения можно включить `hibernate.query.fail_on_pagination_over_collection_fetch=true`.

Надёжная схема для REST `GET /orders?page=0&size=20`:
1. Получить **20 ID заказов** с фильтром, стабильным `ORDER BY` и пагинацией на стороне БД.
2. Вторым SQL загрузить `OrderEntity` с `items` по списку ID через `JOIN FETCH` или batch.
3. Восстановить порядок по списку ID (запрос `IN` не гарантирует порядок) и собрать DTO. Для консистентности между запросами учитывай transaction isolation и изменения данных.

**EAGER не предотвращает N+1 автоматически:** JPQL-запрос может загрузить eagerly требуемую связь отдельными SELECT, если она не включена в основной fetch plan.

Источник: [Hibernate fetching](https://docs.jboss.org/hibernate/orm/6.6/userguide/html_single/Hibernate_User_Guide.html#fetching).

## Two-Minute Drill

- Проблема N+1 возникает, когда после одного SELECT на список выполняются дополнительные SELECT для связанных данных каждого элемента; выбирать нужно fetch-plan, а не отказываться от связей ORM.
- JOIN FETCH коллекции может увеличить число SQL-строк и конфликтовать с пагинацией; DTO или двухэтапная выборка IDs+связи часто безопаснее.

- **JDBC** — Java API работы с соединением, SQL и результатом.
- **JPA** (Jakarta Persistence в современных версиях) — стандарт отображения сущностей и работы через `EntityManager`; **Hibernate** — одна из реализаций.
- **Spring Data JPA** — надстройка с интерфейсами репозиториев поверх JPA. Это не линейная цепочка наследования всех пяти технологий.

**Проверь себя:** если изменил поле entity, полученной через `find` в транзакции, обязателен ли `save`? Нет; объясни dirty checking, flush и commit.
