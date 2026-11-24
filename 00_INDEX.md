# Оглавление Sobes

При первом знакомстве с темой начни с основ и разбери пример. Затем закрой текст и ответь на вопрос в конце. При повторении открой **Two-Minute Drill** и проверь, можешь ли пояснить каждый пункт своими словами. В главах о платформе примеры ориентируются на JDK 21, а различия между версиями отмечены по месту.

## 1. Java Core

### Язык

1. [ООП, Object и рефлексия](Java%20Core/Language/OOP.md)
2. [Дженерики и PECS](Java%20Core/Language/Generics.md)
3. [Исключения и try-with-resources](Java%20Core/Language/Exceptions.md)
4. [Аннотации](Java%20Core/Language/Annotations.md)
5. [String и неизменяемость](Java%20Core/Language/String-and-Immutable.md)

### Коллекции

1. [Интерфейсы коллекций и реализации](Java%20Core/Collections/Overview.md)
2. [Как устроен HashMap](Java%20Core/Collections/HashMap.md)
3. [Выбор коллекции и неизменяемость](Java%20Core/Collections/Choosing-and-Immutability.md)
4. [Java Stream, Spliterator и parallelStream](Java%20Core/Collections/Streams-and-Parallelism.md)

### JVM

1. [Запуск JVM и JIT](Java%20Core/JVM/Overview.md)
2. [Куча, стеки и GC roots](Java%20Core/JVM/Memory-and-GC-Roots.md)
3. [Сборщики мусора, фазы и паузы](Java%20Core/JVM/Garbage-Collectors.md)

### Concurrency

1. [Обзор многопоточности и порядок изучения](Java%20Core/Concurrency/00_Overview.md)
2. [Кэши L1–L3, когерентность и Java Memory Model](Java%20Core/Concurrency/JMM-Happens-Before.md)
3. [Гонки и составные операции](Java%20Core/Concurrency/Race-Conditions.md)
4. [Мониторы, volatile и ReentrantLock](Java%20Core/Concurrency/Synchronization-Interview.md)
5. [Ожидание и прерывание: Lock, Condition, interrupt](Java%20Core/Concurrency/Lock-Condition-Interrupt.md)
6. [CAS, атомики, ABA и ConcurrentHashMap](Java%20Core/Concurrency/CAS-Atomic-ConcurrentHashMap.md)
7. [Executor, Future и CompletableFuture](Java%20Core/Concurrency/Executors-CompletableFuture.md)

## 2. Kotlin и модели исполнения

1. [Исключения в Kotlin](Kotlin/Exceptions.md)
2. [Корутины и Flow](Kotlin/Coroutines-and-Flow.md)
3. [Диспетчеры корутин](Kotlin/Dispatchers.md)
4. [Kotlin Collections и Sequence: eager vs lazy](Kotlin/Collections-and-Sequence.md)
5. [Потоки Tomcat, event loop и virtual threads](Architecture/Threading-Models.md)
6. [Reactive Streams и WebFlux](Reactive/Reactive-Streams-and-WebFlux.md)

## 3. Базы данных и доступ из Java

1. [Модели хранения, ACID, изоляция, индексы и репликация](Databases/Overview.md)
2. [SQL, нормализация и JOIN](Databases/PostgreSQL-SQL-Fundamentals.md)
3. [Блокировки строк](Databases/Row-Locks.md), [проверка версии](Databases/Optimistic-Locking.md) и [deadlock](Databases/Deadlocks.md)
4. [Практика PostgreSQL: планы, пагинация и пул](Databases/PostgreSQL-Interview-Gaps.md)
5. [MongoDB](Databases/MongoDB-Basics.md) и [Redis](Databases/Redis-Basics.md)
6. [JDBC: драйвер, Statement, ResultSet, транзакция и batch](Spring/Data/JDBC.md)
7. [JPA, Hibernate и Spring Data](Spring/Data/Overview.md) и [пакетная вставка Hibernate](Spring/Data/Hibernate-Batch-Insert.md)

## 4. Spring

1. [IoC, DI, скоупы, жизненный цикл бина и прокси](Spring/Core/Overview.md)
2. [Рефлексия в Spring](Spring/Core/Reflection.md)
3. [Spring Boot и свой стартер](Spring/Boot/Overview.md)
4. [Регистрация BeanDefinition и refresh](Spring/Boot/Internals.md)
5. [Транзакция от прокси до commit](Spring/Core/Transactions-and-JPA.md)
6. [REST-контроллер и ошибки](Spring/Web/RestController-and-Errors.md)
7. [Выполнение через @Async](Spring/Core/Async-Methods.md)
8. [Spring Cloud Gateway](Spring/Cloud/Gateway.md)

## 5. API и обмен сообщениями

1. [HTTP, REST, SOAP и JSON-RPC](API/REST-and-Protocols.md)
2. [gRPC и GraphQL](API/gRPC-and-GraphQL.md)
3. [OSI, WebSocket, SSE, webhook и RSocket](API/Network-and-Streaming.md)
4. [Kafka: порядок, доставка и поток событий](Messaging/Kafka.md)
5. [Кластер Kafka, KRaft и consumer group](Messaging/Kafka-Cluster-and-Consumers.md)

## 6. Проектирование, эксплуатация и тесты

1. [SOLID и YAGNI](Architecture/Design-Principles.md)
2. [CAP, Saga, Circuit Breaker и масштабирование](Architecture/Distributed-Systems.md)
3. [Outbox и идемпотентность](Architecture/Transactional-Outbox-and-Idempotency.md)
4. [Проектирование переводов и заказов](Architecture/Money-Transfer-and-Order-Design.md)
5. [Kubernetes](Infrastructure/Kubernetes-Basics.md) и [Ingress](Infrastructure/Kubernetes-Ingress.md)
6. [Метрики и перцентили](Operations/Observability-and-Percentiles.md) и [Quartz](Operations/Quartz-Scheduling.md)
7. [Уровни тестирования](Testing/Testing-Pyramid.md) и [ревью кода платежей](Testing/Payment-Code-Review-Checklist.md)
8. [Maven, Gradle и управление зависимостями](Tooling/Dependency-Management.md)

## Повторение

[Быстрые вопросы](Review/Quick-Questions.md) · [Недавние вопросы с интервью](Review/Interview-Checklist.md) · [Темы, которые стоит добрать](Review/Senior-Checklist.md) · [Карта технических вопросов для повторения](Review/Conversation-Gap-Map.md) · [Куда перенесён материал Questions.docx](Review/Questions-Source-Map.md)

## Как добавлять учебный материал

[Манифест заметок](CONTRIBUTING.md) описывает структуру главы, поиск существующего материала и проверки перед PR.
