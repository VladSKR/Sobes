# 📚 Индекс — Backend Developer Notes

#index #moc

---

## Java Core

| Файл | Темы |
|---|---|
| [[01_Collections]] | ArrayList, LinkedList, HashMap, HashSet, HashCode, PECS, Concurrent Collections, Iterator |
| [[02_Generics]] | Type Erasure, Wildcards, PECS, Bounded Types, Reification |
| [[03_Exceptions]] | Checked/Unchecked, Multicatch, Nested, Propagation, Suppressed, UncaughtExceptionHandler |
| [[04_OOP]] | Принципы ООП, Статическое/динамическое связывание, Композиция/Агрегация, SOLID, Рефлексия |
| [[05_Multithreading]] | Runnable/Callable, Volatile, Future, CompletableFuture, Синхронизация |
| [[06_JVM_GC]] | Структура памяти, JIT, GC алгоритмы, G1, Аннотации RetentionPolicy, Ingress |

---

## Spring

| Файл | Темы |
|---|---|
| [[01_Spring_Core]] | IoC, DI, Способы инъекций, Скоупы, Жизненный цикл бина, Циклические зависимости, Проксирование |
| [[02_JPA_Hibernate_JDBC]] | JDBC, Statement, Batch, JPA аннотации, EntityManager, Hibernate N+1, Spring Data JPA |

---

## Базы данных

| Файл | Темы |
|---|---|
| [[01_Databases]] | SQL vs NoSQL, ACID, Уровни изоляции, Блокировки, Deadlock, Индексы, EXPLAIN, WAL, Репликация, Партиционирование |

---

## Архитектура

| Файл | Темы |
|---|---|
| [[01_Architecture]] | CAP, SAGA, Circuit Breaker, OSI, WebSocket, SSE, Webhook, Backpressure, Горизонтальное масштабирование, Перцентили, Шифрование |
| [[02_Kafka]] | Партиционирование, Семантика доставки, Consumer Groups, Event Streaming |
| [[03_REST_API]] | REST, HTTP методы, Идемпотентность, PATCH vs PUT, HATEOAS, SOAP, Пирамида тестирования, Maven |

---

## Быстрые ссылки по темам

### Коллекции
- Как работает HashMap → [[01_Collections#Как работает HashMap]]
- Удаление при итерации → [[01_Collections#Удаление элементов при итерации]]
- Concurrent коллекции → [[01_Collections#Concurrent коллекции]]

### Многопоточность
- CompletableFuture → [[05_Multithreading#CompletableFuture]]
- Исключения в потоках → [[05_Multithreading#Отслеживание исключений в потоках]]

### Spring
- Жизненный цикл бина → [[01_Spring_Core#Жизненный цикл бина]]
- Циклические зависимости → [[01_Spring_Core#Циклическая зависимость]]

### БД
- Уровни изоляции → [[01_Databases#Уровни изоляции транзакций]]
- Индексы PostgreSQL → [[01_Databases#Типы индексов PostgreSQL]]
