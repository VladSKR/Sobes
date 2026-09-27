# Spring Data / JPA / Hibernate / JDBC

#spring #jpa #hibernate #jdbc #database

---

## Иерархия абстракций

```
Spring Data (абстракция над всем)
    └── JPA (стандарт/спецификация)
            └── Hibernate (реализация JPA)
                    └── JDBC (низкоуровневое API для СУБД)
                            └── JDBC Driver (конкретный драйвер БД)
```

---

## JDBC

Низкоуровневое Java API для работы с реляционными БД.

### Основы
```java
// 1. Получить соединение
Connection conn = DriverManager.getConnection(url, user, password);

// 2. Создать statement
Statement stmt = conn.createStatement();
ResultSet rs = stmt.executeQuery("SELECT * FROM users");

// 3. Обработать результат
while (rs.next()) {
    String name = rs.getString("name");
}

// 4. Закрыть ресурсы
rs.close(); stmt.close(); conn.close();
```

### Statement vs PreparedStatement vs CallableStatement

| | Statement | PreparedStatement | CallableStatement |
|---|---|---|---|
| Параметры | Нет (конкатенация) | Через `?` | Через `?` |
| SQL Injection | ❌ Уязвим | ✅ Защищён | ✅ Защищён |
| Производительность | Хуже | Лучше (precompiled) | - |
| Использование | DDL | DML запросы | Хранимые процедуры |

```java
// PreparedStatement
PreparedStatement ps = conn.prepareStatement(
    "SELECT * FROM users WHERE id = ? AND name = ?"
);
ps.setInt(1, 42);
ps.setString(2, "Alice");
ResultSet rs = ps.executeQuery();
```

### Batch операции
```java
PreparedStatement ps = conn.prepareStatement("INSERT INTO users VALUES (?, ?)");
conn.setAutoCommit(false);

for (User u : users) {
    ps.setInt(1, u.getId());
    ps.setString(2, u.getName());
    ps.addBatch(); // не выполняет, только добавляет
}

ps.executeBatch(); // один раз отправляет все
conn.commit();
```

---

## JPA — Java Persistence API

Стандарт (спецификация) ORM для Java. Описывает аннотации и интерфейсы.

### Основные аннотации
```java
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "user_name", nullable = false, length = 100)
    private String name;
    
    @ManyToOne
    @JoinColumn(name = "department_id")
    private Department department;
    
    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    private List<Order> orders;
}
```

### EntityManager
```java
@PersistenceContext
EntityManager em;

// CRUD
em.persist(user);           // INSERT
em.find(User.class, 1L);    // SELECT by PK
em.merge(user);             // UPDATE
em.remove(user);            // DELETE

// JPQL
List<User> users = em.createQuery(
    "SELECT u FROM User u WHERE u.name = :name", User.class)
    .setParameter("name", "Alice")
    .getResultList();
```

---

## Hibernate

Самая популярная реализация JPA.

### N+1 проблема
```java
// Плохо: 1 запрос для списка + N запросов для каждого элемента
List<Department> depts = em.createQuery("FROM Department", Department.class).getResultList();
for (Department d : depts) {
    d.getEmployees().size(); // N дополнительных запросов!
}

// Решение 1: JOIN FETCH
em.createQuery("FROM Department d JOIN FETCH d.employees", Department.class)

// Решение 2: @EntityGraph
// Решение 3: @BatchSize
@BatchSize(size = 20)
private List<Employee> employees;
```

### FetchType
```java
@OneToMany(fetch = FetchType.LAZY)  // по умолчанию для коллекций — загружать при обращении
@ManyToOne(fetch = FetchType.EAGER) // по умолчанию для @ManyToOne — загружать сразу
```

---

## Spring Data JPA

Абстракция над JPA — убирает шаблонный код.

```java
public interface UserRepository extends JpaRepository<User, Long> {
    // Авто-генерация запроса по имени метода
    List<User> findByName(String name);
    Optional<User> findByEmailAndActive(String email, boolean active);
    
    // JPQL
    @Query("SELECT u FROM User u WHERE u.name = :name")
    List<User> findByNameJpql(@Param("name") String name);
    
    // Native SQL
    @Query(value = "SELECT * FROM users WHERE name = :name", nativeQuery = true)
    List<User> findByNameNative(@Param("name") String name);
}
```

### JpaRepository методы из коробки
```java
save(entity)        // INSERT или UPDATE
findById(id)        // SELECT по PK → Optional
findAll()           // SELECT *
deleteById(id)      // DELETE
count()             // COUNT(*)
existsById(id)      // EXISTS
findAll(Pageable)   // Пагинация + сортировка
```
