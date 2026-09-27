# Hibernate batch insert

## За две минуты

- `saveAll` сохраняет набор сущностей, но само по себе не обещает отправку SQL в JDBC batch.
- `hibernate.jdbc.batch_size` разрешает Hibernate группировать однотипные SQL-команды; `IDENTITY` у вставляемой сущности обычно мешает JDBC insert batching.
- JDBC batch — несколько подготовленных операций за меньшее число сетевых взаимодействий; это **не обязательно один SQL INSERT с множеством VALUES**.

## Пример

```yaml
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          batch_size: 50
        order_inserts: true
```

Допустим, нужно вставить 1000 `User`. Без batching Hibernate отправит много отдельных insert. При подходящей стратегии ID и настройках он может собирать их по 50 команд в JDBC batch. Драйвер и БД могут отправить/переписать batch по-своему — по одному красивому SQL в логах нельзя надёжно оценивать число round trips.

`IDENTITY` получает ID при самой вставке; для этой сущности Hibernate обычно отключает JDBC insert batching. С `SEQUENCE` и правильно настроенным генератором ID может быть получен заранее, что делает batching возможным. `allocationSize` уменьшает обращения к sequence в подходящей конфигурации, но не подставляй всегда арифметику вида «ровно 8 млн → 160 тысяч запросов»: влияют ID generator, драйвер, flush, ограничения и реальная нагрузка.

При миллионах сущностей не накапливай всё в одном persistence context: периодически `flush()` и `clear()` в управляемой транзакции или применяй специализированную пакетную загрузку. `flush` отправляет SQL, но `commit` остаётся отдельной границей. Для проверки включи Hibernate statistics/JDBC metrics, смотри число подготовленных statements и выполненных batch, измеряй время и память. Наличие `batch_size` в YAML само по себе не доказывает эффект.

```java
// Пример стратегии, которую можно использовать вместе с batching:
@Id
@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "user_seq")
@SequenceGenerator(name = "user_seq", sequenceName = "user_sequence", allocationSize = 50)
private Long id;
```

## Другие настройки и прямой JDBC

`order_updates=true` может группировать UPDATE одного типа; `batch_versioned_data` требует учитывать версию Hibernate и драйвер. Для предсказуемого batch без ORM используй `PreparedStatement.addBatch()` и `executeBatch()` в транзакции; Spring `JdbcTemplate.batchUpdate` упрощает закрытие ресурсов. Это всё ещё пакет отдельных подготовленных команд, а не обязательный SQL с множеством VALUES.

**Проверь себя:** почему `saveAll` и `batch_size` ещё не доказывают, что 1000 вставок превратились в 20 батчей? Нужно проверить генератор ID, SQL, настройки и реальные показатели JDBC.

Источник: [Hibernate User Guide — batching](https://docs.jboss.org/hibernate/orm/6.5/userguide/html_single/Hibernate_User_Guide.html#batch).
