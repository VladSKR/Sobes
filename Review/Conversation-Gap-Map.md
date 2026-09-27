# Что добавлено по нашим разборам собеседований

Карта построена по доступной истории учебных диалогов и заметкам репозитория. **Оценка здесь про глубину конкретного ответа**, а не про общий уровень разработчика.

| Уверенность в ответе | Вопрос, который нужно повторить | Где ответ |
|---|---|---|
| Требовал подробного разбора | Почему `ready=true, data=0` возможно? Какая цепочка HB это запрещает? | [[Java Core/Concurrency/JMM-Happens-Before]] |
| Требовал подробного разбора | Зачем `finally` для Lock? Что будет при interrupt до захвата? | [[Java Core/Concurrency/Lock-Condition-Interrupt]] |
| Требовал подробного разбора | CAS failure; `count++`; почему `containsKey + put` не атомарны? | [[Java Core/Concurrency/CAS-Atomic-ConcurrentHashMap]] |
| Требовал подробного разбора | `thenCompose` vs `thenCombine`; что реально останавливает `orTimeout`? | [[Java Core/Concurrency/Executors-CompletableFuture]] |
| Требовал подробного разбора | GC roots, G1 marking и эвакуация; Stack vs Heap? | [[Java Core/JVM/Memory-and-GC-Roots]] |
| Требовал подробного разбора | LinkedList вставки, defensive copy, immutable view? | [[Java Core/Collections/Choosing-and-Immutability]] |
| Требовал подробного разбора | Путь proxy → TransactionInterceptor → manager → JDBC/JPA → commit? | [[Spring/Core/Transactions-and-JPA]] |
| Требовал подробного разбора | BeanDefinition, BPP, prototype, `@Bean` vs `@Component`? | [[Spring/Core/Bean-Lifecycle-Interview]] |
| Требовал подробного разбора | MVCC, EXPLAIN, составной индекс, keyset, Hikari exhaustion? | [[Databases/PostgreSQL-Interview-Gaps]] |
| Требовал подробного разбора | Kafka KRaft, ISR, offset commit, ordering, rebalance? | [[Messaging/Kafka-Cluster-and-Consumers]] |
| Требовал подробного разбора | `request(n)` и перегрузка; event loop и блокирующий JDBC? | [[Reactive/Reactive-Streams-and-WebFlux]] |
| Требовал подробного разбора | Continuation, `async/await`, cold/hot Flow, отмена? | [[Kotlin/Coroutines-and-Flow]] |
| Требовал базового разбора | MongoDB `find`/индексы/`$lookup`? | [[Databases/MongoDB-Basics]] |
| Просил обзор без практического опыта | Чем gRPC и GraphQL отличаются от REST? | [[API/gRPC-and-GraphQL]] |
| Основа сильная, нужны точные границы | Идемпотентность, Outbox, два commit, retry и компенсации? | [[Architecture/Transactional-Outbox-and-Idempotency]] |
| Основа сильная, нужна точная модель | Balance/reserve и состояния checkout/оплаты? | [[Architecture/Money-Transfer-and-Order-Design]] |

## Как повторять за 3–5 минут

Открой строку таблицы → прочитай первый раздел заметки → закрой заметку и ответь на вопрос вслух → открой снова и сравни. Если не можешь объяснить механизм на одном примере, допиши свой пример прямо в тематическую заметку.
