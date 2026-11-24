# Карта технических вопросов и повторения

Этот раздел связывает темы учебника с типовыми вопросами на собеседованиях Java/Kotlin Backend. Каждая ссылка ведёт к **самостоятельному объяснению** механизма. Личные оценки кандидатов, даты занятий и отчёты о подготовке здесь не ведутся.

## Коллекции и конкуренция

| Вопрос | Где изучать |
|---|---|
| Почему `filter().map().first()` для `List` и `Sequence` выполняются по-разному? | [Kotlin Collections и Sequence](../Kotlin/Collections-and-Sequence.md) |
| Какие задачи подходят `parallelStream` и как работают `Spliterator`, `ForkJoinPool` и work stealing? | [Java Streams](../Java%20Core/Collections/Streams-and-Parallelism.md) |
| Чем асинхронность отличается от параллельности? Как один event loop ждёт I/O? | [Модели исполнения](../Architecture/Threading-Models.md) |
| Как `suspend`, `Continuation` и `Dispatchers.IO` ведут себя с JDBC и R2DBC? | [Корутины](../Kotlin/Coroutines-and-Flow.md), [Dispatchers](../Kotlin/Dispatchers.md) |
| В каком порядке `ThreadPoolExecutor` использует core threads, очередь, дополнительные потоки и политику отказа? | [Executors и CompletableFuture](../Java%20Core/Concurrency/Executors-CompletableFuture.md) |
| Когда `synchronized` гарантирует видимость? Почему `volatile counter++` не атомарен? | [Java Memory Model](../Java%20Core/Concurrency/JMM-Happens-Before.md), [синхронизация](../Java%20Core/Concurrency/Synchronization-Interview.md) |
| Как избежать двух загрузок одного ключа с `ConcurrentHashMap`? | [CAS и ConcurrentHashMap](../Java%20Core/Concurrency/CAS-Atomic-ConcurrentHashMap.md) |

## JVM и диагностика

| Вопрос | Где изучать |
|---|---|
| Какие фазы G1 останавливают приложение и как ZGC уменьшает паузы? | [Garbage Collectors](../Java%20Core/JVM/Garbage-Collectors.md) |
| Как отличить давление аллокаций от удержания объектов? | [GC и диагностика](../Java%20Core/JVM/Garbage-Collectors.md) |
| Как открыть `.hprof`, читать Histogram, Dominator Tree, Retained Heap и GC Roots? | [Память JVM и Heap Dump](../Java%20Core/JVM/Memory-and-GC-Roots.md) |
| Почему memory limit контейнера не равен `-Xmx`? | [Память JVM](../Java%20Core/JVM/Memory-and-GC-Roots.md), [Kubernetes](../Infrastructure/Kubernetes-Basics.md) |

## Spring, Hibernate, PostgreSQL

| Вопрос | Где изучать |
|---|---|
| Чем отличаются `@SpringBootConfiguration`, `@ComponentScan` и `@EnableAutoConfiguration`? Как написать starter? | [Spring Boot](../Spring/Boot/Overview.md), [BeanDefinition/refresh](../Spring/Boot/Internals.md) |
| Чем отличаются `@Component`, `@Configuration`, `@Bean` и `@Import`? | [Spring Core](../Spring/Core/Overview.md), [Spring Boot](../Spring/Boot/Overview.md) |
| Почему возникает N+1? Как выбрать `JOIN FETCH`, `@EntityGraph`, batch fetching и DTO? | [JPA/Hibernate](../Spring/Data/Overview.md) |
| Почему `JOIN FETCH` коллекции конфликтует с `Pageable`? | [JPA/Hibernate](../Spring/Data/Overview.md) |
| Чем Keyset Pagination отличается от OFFSET при вставке и удалении строк? | [PostgreSQL](../Databases/PostgreSQL-Interview-Gaps.md) |
| Как Spring реализует транзакции через proxy и `TransactionInterceptor`? | [Транзакции](../Spring/Core/Transactions-and-JPA.md) |

## Распределённые системы и инфраструктура

| Вопрос | Где изучать |
|---|---|
| Что CAP утверждает о согласованности и доступности при network partition? | [Распределённые системы](../Architecture/Distributed-Systems.md) |
| Что делают CLOSED, OPEN и HALF_OPEN в Circuit Breaker? | [Распределённые системы](../Architecture/Distributed-Systems.md) |
| Чем `requests` отличаются от `limits`, а Evicted Pod — от OOMKilled? | [Kubernetes](../Infrastructure/Kubernetes-Basics.md) |
| Как обеспечить идемпотентность платежей и безопасную публикацию событий? | [Outbox и Idempotency](../Architecture/Transactional-Outbox-and-Idempotency.md), [HTTP](../API/REST-and-Protocols.md) |

## Как повторять материал

1. Выбрать вопрос и сначала объяснить механизм своими словами, зафиксировав входные условия.
2. Прочитать предметную главу, последовательно пройти её пример.
3. Открыть **Two-Minute Drill** и выполнить самопроверку в главе; сверить не только ответ, но и причину.
4. При ошибке повторить соответствующий раздел, а не запоминать отдельную фразу.

Дополнительные разделы по [JMM](../Java%20Core/Concurrency/JMM-Happens-Before.md), [PostgreSQL и MVCC](../Databases/Overview.md), [Kafka](../Messaging/Kafka.md), [структурной конкурентности](../Kotlin/Coroutines-and-Flow.md) и [REST](../API/REST-and-Protocols.md) доступны через [оглавление](../00_INDEX.md).
