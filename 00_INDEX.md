# Карта заметок для собеседования

Начни с [[Review/Quick-Questions|быстрых вопросов]] и [[Review/Senior-Checklist|чеклиста Senior]]. Разбор конкретной темы открывай по ссылкам ниже.

**Как пользоваться:** в каждой тематической заметке раздел «За две минуты» даёт ответ для повторения. Если формулировка непонятна, читай следующие разделы: там механизм, пример и ограничения. После чтения попробуй объяснить тему своими словами и проверить себя на примере. Числа производительности и настройки всегда сверяй с версией технологии и реальной нагрузкой.

## Java Core — только язык и платформа Java

- **Язык:** [[Java Core/Language/OOP|ООП и Object]], [[Java Core/Language/Generics|дженерики]], [[Java Core/Language/Exceptions|исключения]], [[Java Core/Language/Annotations|аннотации]].
- **Коллекции:** [[Java Core/Collections/Overview|обзор]], [[Java Core/Collections/HashMap|HashMap подробно]].
- **Concurrency:** [[Java Core/Concurrency/Overview|основы, JMM, Future]], [[Java Core/Concurrency/Synchronization-Interview|volatile и блокировки]], [[Java Core/Concurrency/Race-Conditions|гонки]], [[Java Core/Concurrency/CompletableFuture-Interview|практика CompletableFuture]].
- **JVM:** [[Java Core/JVM/Overview|память, JIT, GC и запуск]], [[Java Core/JVM/G1-vs-ZGC|G1 и ZGC]].

## Kotlin

[[Kotlin/Exceptions|Исключения]] · [[Kotlin/Dispatchers|Диспетчеры корутин]]

## Spring

- **Core:** [[Spring/Core/Overview|IoC, DI, бины и прокси]], [[Spring/Core/Reflection|рефлексия в Spring]].
- **Boot:** [[Spring/Boot/Overview|стартеры и автоконфигурация]], [[Spring/Boot/Internals|BeanDefinition и refresh()]].
- **Data:** [[Spring/Data/Overview|JDBC, JPA, Hibernate]], [[Spring/Data/Hibernate-Batch-Insert|batch insert]].
- **Cloud:** [[Spring/Cloud/Gateway|Gateway]].

## Данные и интеграция

- **БД:** [[Databases/Overview|основы БД]], [[Databases/Row-Locks|строчные блокировки]], [[Databases/Optimistic-Locking|оптимистичная блокировка]], [[Databases/Deadlocks|взаимные блокировки]].
- **Сообщения:** [[Messaging/Kafka|Kafka]].
- **API:** [[API/REST-and-Protocols|REST, HTTP, SOAP и RPC]].

## Архитектура и эксплуатация

[[Architecture/Distributed-Systems|Распределённые системы]] · [[Architecture/Design-Principles|SOLID и YAGNI]] · [[Architecture/Threading-Models|Потоки, event loop и virtual threads]] · [[Operations/Quartz-Scheduling|Quartz]] · [[Infrastructure/Kubernetes-Ingress|Kubernetes Ingress]] · [[Testing/Testing-Pyramid|Тестирование]] · [[Tooling/Dependency-Management|Сборка и зависимости]]

## Разборы по нашим собеседованиям

- **Java:** [[Java Core/Concurrency/JMM-Happens-Before|JMM и HB]], [[Java Core/Concurrency/Lock-Condition-Interrupt|Lock/Condition/interrupt]], [[Java Core/Concurrency/CAS-Atomic-ConcurrentHashMap|CAS и ConcurrentHashMap]], [[Java Core/Concurrency/Executors-CompletableFuture|Executor и CompletableFuture]], [[Java Core/JVM/Memory-and-GC-Roots|GC roots]], [[Java Core/Collections/Choosing-and-Immutability|выбор и неизменяемость коллекций]], [[Java Core/Language/String-and-Immutable|String и immutable]].
- **Spring и БД:** [[Spring/Core/Transactions-and-JPA|транзакции]], [[Spring/Core/Bean-Lifecycle-Interview|жизненный цикл бина]], [[Spring/Web/RestController-and-Errors|REST-контроллер]], [[Spring/Core/Async-Methods|@Async]], [[Databases/PostgreSQL-Interview-Gaps|PostgreSQL]], [[Databases/MongoDB-Basics|MongoDB]], [[Databases/PostgreSQL-SQL-Fundamentals|SQL и нормализация]], [[Databases/Redis-Basics|Redis]].
- **Интеграции:** [[Messaging/Kafka-Cluster-and-Consumers|кластер Kafka и consumer]], [[Reactive/Reactive-Streams-and-WebFlux|WebFlux/Reactive Streams]], [[Kotlin/Coroutines-and-Flow|корутины и Flow]], [[API/gRPC-and-GraphQL|gRPC и GraphQL]].
- **Архитектура и практика:** [[Architecture/Transactional-Outbox-and-Idempotency|Outbox/идемпотентность]], [[Architecture/Money-Transfer-and-Order-Design|переводы и заказы]], [[Testing/Payment-Code-Review-Checklist|code review]], [[Infrastructure/Kubernetes-Basics|Kubernetes]], [[Operations/Observability-and-Percentiles|наблюдаемость]].

## Повторение

[[Review/Conversation-Gap-Map|Карта пробелов из диалогов]] · [[Review/Quick-Questions|Быстрые вопросы]] · [[Review/Interview-Checklist|Вопросы с интервью]] · [[Review/Senior-Checklist|Пробелы Senior]]

## Куда добавлять новое

1. Одна тема — одна заметка в подходящей папке. Перед созданием проверь ссылки выше и поиск Obsidian, чтобы дополнить существующую заметку.
2. Java SE (язык, коллекции, JVM, concurrency) → `Java Core/`. Kotlin, Spring, БД, брокеры и инфраструктура → одноимённые разделы.
3. Сначала краткий ответ для повторения, затем механизм, пример и блок «Где я ошибался / что проверить».
4. Новую заметку добавь в этот индекс. В `Review/` храни вопросы и чеклисты со ссылками на канонические заметки.

> Материалы перенесены из исходных заметок с сохранением разделов. При повторении сверяй спорные численные оценки и примеры с документацией выбранной версии Java, Spring и PostgreSQL.
