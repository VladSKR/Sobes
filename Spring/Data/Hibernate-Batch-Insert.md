# Hibernate Batch Insert

## 3. Hibernate Batch Insert

### Проблема N+1 INSERT

```kotlin
// saveAll НЕ гарантирует batch по умолчанию
userRepository.saveAll(listOf(user1, user2, user3))

// Hibernate без настроек выполняет:
// INSERT INTO users VALUES (...)  -- 1
// INSERT INTO users VALUES (...)  -- 2
// INSERT INTO users VALUES (...)  -- 3
// N отдельных round-trip к БД
```

---

### Включение batch insert

```yaml
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          batch_size: 50          # размер батча
        order_inserts: true       # группировать INSERT одного типа вместе
        order_updates: true       # то же для UPDATE
        batch_versioned_data: true # батч для сущностей с @Version
```

С этими настройками Hibernate группирует запросы:
```sql
-- Один round-trip вместо 50
INSERT INTO users (name, email) VALUES (?, ?), (?, ?), ... (50 раз)
```

---

### Проблема с IDENTITY генерацией ID

```java
@Entity
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY) // ПРОБЛЕМА
    private Long id;
}
```

`IDENTITY` стратегия требует выполнить INSERT чтобы получить ID от БД. Hibernate **отключает батчинг** для таких сущностей — нельзя батчить если нужен ID после каждой вставки.

**Решение — SEQUENCE стратегия:**
```java
@Id
@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "user_seq")
@SequenceGenerator(name = "user_seq", sequenceName = "user_sequence", allocationSize = 50)
private Long id;
```

`allocationSize = 50` — Hibernate резервирует 50 ID за один запрос к sequence, потом выдаёт их локально. Один запрос на sequence вместо 50.

---

### Как это выглядело в твоём проекте с Kafka

```java
// Без батчинга — 4 млн записей = 8 млн запросов (select seq + insert)
repository.saveAll(entities); // медленно

// С батчингом:
// - allocationSize = 50 → 4млн / 50 = 80к запросов к sequence
// - batch_size = 50 → 4млн / 50 = 80к INSERT батчей
// Итого ~160к запросов вместо 8 млн
```

---

### Проверка что батчинг работает

```yaml
# Включить логирование SQL статистики
spring:
  jpa:
    properties:
      hibernate:
        generate_statistics: true

logging:
  level:
    org.hibernate.stat: DEBUG
```

В логах увидишь:
```
Hibernate: 
    batch size: 50
    jdbc.batch.size: 50
```

---

### JDBC батчинг напрямую (максимальная производительность)

Если Hibernate всё равно медленный — JDBC батч напрямую:

```kotlin
jdbcTemplate.batchUpdate(
    "INSERT INTO users (name, email) VALUES (?, ?)",
    object : BatchPreparedStatementSetter {
        override fun setValues(ps: PreparedStatement, i: Int) {
            ps.setString(1, users[i].name)
            ps.setString(2, users[i].email)
        }
        override fun getBatchSize() = users.size
    }
)
```

---
