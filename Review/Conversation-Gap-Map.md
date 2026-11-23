# Что добавлено по нашим разборам собеседований

Карта построена по доступной истории учебных диалогов и заметкам репозитория. **Оценка здесь про глубину конкретного ответа**, а не про общий уровень разработчика.

| Уверенность в ответе | Вопрос, который нужно повторить | Где ответ |
|---|---|---|
| Требовал подробного разбора | Почему `ready=true, data=0` возможно? Какая цепочка HB это запрещает? | [[Java Core/Concurrency/JMM-Happens-Before]] |
| Требовал подробного разбора | Зачем `finally` для Lock? Что будет при interrupt до захвата? | [[Java Core/Concurrency/Lock-Condition-Interrupt]] |
| Требовал подробного разбора | CAS failure; `count++`; почему `containsKey + put` не атомарны? | [[Java Core/Concurrency/CAS-Atomic-ConcurrentHashMap]] |
| Требовал подробного разбора | `thenCompose` vs `thenCombine`; что реально останавливает `orTimeout`? | [[Java Core/Concurrency/Executors-CompletableFuture]] |
| Требовал подробного разбора | GC roots, разница stack/heap, маркировка и эвакуация G1? | [Память JVM](../Java%20Core/JVM/Memory-and-GC-Roots.md), [сборщики](../Java%20Core/JVM/Garbage-Collectors.md) |
| Требовал подробного разбора | LinkedList вставки, defensive copy, immutable view? | [[Java Core/Collections/Choosing-and-Immutability]] |
| Требовал подробного разбора | Путь proxy → TransactionInterceptor → manager → JDBC/JPA → commit? | [[Spring/Core/Transactions-and-JPA]] |
| Требовал подробного разбора | BeanDefinition, BPP, prototype, `@Bean` vs `@Component`? | [[Spring/Core/Overview#Жизненный цикл бина]] |
| Требовал подробного разбора | MVCC, EXPLAIN, составной индекс, keyset, Hikari exhaustion? | [[Databases/PostgreSQL-Interview-Gaps]] |
| Требовал подробного разбора | Kafka KRaft, ISR, offset commit, ordering, rebalance? | [[Messaging/Kafka-Cluster-and-Consumers]] |
| Требовал подробного разбора | `request(n)` и перегрузка; event loop и блокирующий JDBC? | [[Reactive/Reactive-Streams-and-WebFlux]] |
| Требовал подробного разбора | Continuation, `async/await`, cold/hot Flow, отмена? | [[Kotlin/Coroutines-and-Flow]] |
| Требовал базового разбора | MongoDB `find`/индексы/`$lookup`? | [[Databases/MongoDB-Basics]] |
| Просил обзор без практического опыта | Чем gRPC и GraphQL отличаются от REST? | [[API/gRPC-and-GraphQL]] |
| Основа сильная, нужны точные границы | Идемпотентность, Outbox, два commit, retry и компенсации? | [[Architecture/Transactional-Outbox-and-Idempotency]] |
| Основа сильная, нужна точная модель | Balance/reserve и состояния checkout/оплаты? | [[Architecture/Money-Transfer-and-Order-Design]] |

## Собеседование 21–22 ноября 2026: конкретные оценки и разборы

Ниже **исходные оценки из тренировочного собеседования до повторения**. Они **не пересчитываются** при изучении темы. Статусы означают только, что механизм разобран здесь в чате и занесён в предметную главу; это не сертификат полного знания всех деталей.

| Тема | Исходная оценка | Результат разбора | Где объяснение |
|---|---:|---|---|
| Kotlin Sequence | **2/10** | Проверен eager/lazy, порядок `filter/map/first` | [Kotlin Collections и Sequence](../Kotlin/Collections-and-Sequence.md) |
| Java parallelStream | **2/10** | Проверен ForkJoin/commonPool, CPU vs I/O | [Stream и parallelStream](../Java%20Core/Collections/Streams-and-Parallelism.md) |
| Асинхронность и non-blocking I/O | **3/10** | Проверены ожидание I/O, suspend vs блокировка | [Модели исполнения](../Architecture/Threading-Models.md), [корутины](../Kotlin/Coroutines-and-Flow.md) |
| ThreadPoolExecutor | **4/10** | Проверен сценарий 2 core / 4 max / queue=2 и CallerRuns | [Executors](../Java%20Core/Concurrency/Executors-CompletableFuture.md) |
| Keyset Pagination | **4/10** | Разобраны курсор, направление DESC/ASC, вставка/удаление | [PostgreSQL](../Databases/PostgreSQL-Interview-Gaps.md) |
| Kotlin Collections | **5/10** | Проверены eager `map/filter` и промежуточные списки | [Kotlin Collections и Sequence](../Kotlin/Collections-and-Sequence.md) |
| G1 vs ZGC | **5/10** | Проверены STW, ZGC, allocation pressure и диагностика | [GC](../Java%20Core/JVM/Garbage-Collectors.md), [Heap Dump](../Java%20Core/JVM/Memory-and-GC-Roots.md) |
| JMM / synchronized | **6/10** | Проверена видимость с/без монитора, volatile и lost updates | [JMM](../Java%20Core/Concurrency/JMM-Happens-Before.md) |
| Circuit Breaker | **6/10** | Проверены OPEN/CLOSED/HALF_OPEN, аналогия с электрической цепью | [Распределённые системы](../Architecture/Distributed-Systems.md) |
| CAP | **6/10** | Базовая проверка кворума 2/3; глубокую проверку не проводили | [Распределённые системы](../Architecture/Distributed-Systems.md) |
| Kubernetes resources | **6/10** | Разобраны requests/limits, throttling, eviction и Pending | [Kubernetes](../Infrastructure/Kubernetes-Basics.md) |
| Hibernate N+1 | **6/10** | **Теория разобрана; контрольные вопросы ещё не отвечены** | [JPA и Hibernate](../Spring/Data/Overview.md) |

### Дополнительные вопросы, которые действительно затрагивали

- **ConcurrentHashMap:** `putIfAbsent` vs `computeIfAbsent`, один запрос в БД на ключ, ожидание второго потока, ограничения `suspend` — [CAS и ConcurrentHashMap](../Java%20Core/Concurrency/CAS-Atomic-ConcurrentHashMap.md).
- **Heap Dump через IntelliJ/MAT:** Histogram, Shallow/Retained Heap, Dominator Tree, Path to GC Roots, `DocumentCache → CustomerDocument → byte[]`; различие heap и памяти контейнера — [Память JVM](../Java%20Core/JVM/Memory-and-GC-Roots.md).
- **Spring Boot:** `@SpringBootConfiguration`, `@ComponentScan`, `@EnableAutoConfiguration`, `@Import`, `AutoConfiguration.imports`, минимальный starter и `@ConditionalOnClass/@ConditionalOnMissingBean` — [Spring Boot](../Spring/Boot/Overview.md), [внутреннее устройство](../Spring/Boot/Internals.md).
- **Spring Bean Lifecycle:** теория затронута, но самостоятельную проверку ещё не завершили — [Spring Core](../Spring/Core/Overview.md).

### Следующий этап после исходных оценённых пробелов

Из исходного чек-листа ещё стоит отдельно проверить и доучить [Spring Bean Lifecycle](../Spring/Core/Overview.md), [REST и Idempotency-Key](../API/REST-and-Protocols.md), [корутины: structured concurrency/cancellation](../Kotlin/Coroutines-and-Flow.md). После этого — короткая проверка [@Transactional](../Spring/Core/Transactions-and-JPA.md), [PostgreSQL Isolation/Indexes](../Databases/PostgreSQL-Interview-Gaps.md) и [Transactional Outbox](../Architecture/Transactional-Outbox-and-Idempotency.md). **Эти темы не помечены как завершённые в текущем разборе.**

## Как повторять за 3–5 минут

Открой строку таблицы → найди и прочитай Two-Minute Drill, затем при необходимости вернись к примеру в главе → закрой заметку и ответь на вопрос вслух → открой снова и сравни. Если не можешь объяснить механизм на одном примере, допиши свой пример прямо в тематическую заметку.
