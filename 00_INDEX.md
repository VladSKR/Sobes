# Карта заметок для собеседования

Начни с [[Review/Quick-Questions|быстрых вопросов]] и [[Review/Senior-Checklist|чеклиста Senior]]. Разбор конкретной темы открывай по ссылкам ниже.

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

## Повторение

[[Review/Quick-Questions|Быстрые вопросы]] · [[Review/Interview-Checklist|Вопросы с интервью]] · [[Review/Senior-Checklist|Пробелы Senior]]

## Куда добавлять новое

1. Одна тема — одна заметка в подходящей папке. Перед созданием проверь ссылки выше и поиск Obsidian, чтобы дополнить существующую заметку.
2. Java SE (язык, коллекции, JVM, concurrency) → `Java Core/`. Kotlin, Spring, БД, брокеры и инфраструктура → одноимённые разделы.
3. Сначала краткий ответ для повторения, затем механизм, пример и блок «Где я ошибался / что проверить».
4. Новую заметку добавь в этот индекс. В `Review/` храни вопросы и чеклисты со ссылками на канонические заметки.

> Материалы перенесены из исходных заметок с сохранением разделов. При повторении сверяй спорные численные оценки и примеры с документацией выбранной версии Java, Spring и PostgreSQL.
