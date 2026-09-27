# Темы с собеседования — детальный разбор

---

## 1. CompletableFuture — параллельная отправка SMS с таймаутом

### Исходная проблема

```kotlin
// Последовательная обработка — один зависший send блокирует всё
messages.forEach { message ->
    try {
        smsGateway.send(message)           // если зависнет — всё стоит
        smsService.save(message.copy(isSent = true))
    } catch (ex: Exception) {
        logger.error(ex)
    }
}
```

### Решение через CompletableFuture с таймаутом

```java
// Java вариант
ExecutorService executor = Executors.newFixedThreadPool(10);

List<CompletableFuture<Void>> futures = messages.stream()
    .map(message -> CompletableFuture
        .runAsync(() -> {
            try {
                smsGateway.send(message);
                smsService.save(message.copy(true));
            } catch (Exception ex) {
                logger.error("Failed to send: {}", message.getId(), ex);
            }
        }, executor)
        .orTimeout(5, TimeUnit.SECONDS)           // таймаут на каждое сообщение
        .exceptionally(ex -> {
            logger.error("Timeout or error for message: {}", message.getId());
            return null;
        })
    )
    .collect(Collectors.toList());

// Ждём завершения всех
CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();
```

### Kotlin вариант через корутины

```kotlin
// Более идиоматично для Kotlin проекта
suspend fun sendMessages(messages: List<SmsEntity>) = coroutineScope {
    messages.map { message ->
        async {
            try {
                withTimeout(5000L) {           // таймаут 5 секунд
                    smsGateway.send(message)
                    smsService.save(message.copy(isSent = true))
                }
            } catch (ex: TimeoutCancellationException) {
                logger.error("Timeout for message: ${message.id}")
            } catch (ex: Exception) {
                logger.error("Error for message: ${message.id}", ex)
            }
        }
    }.awaitAll()                               // ждём все корутины
}
```

### Основные методы CompletableFuture

```java
// Создание
CompletableFuture.runAsync(() -> doWork());              // без результата
CompletableFuture.supplyAsync(() -> compute());          // с результатом
CompletableFuture.supplyAsync(() -> compute(), executor); // свой пул потоков

// Цепочка
future
    .thenApply(result -> transform(result))    // преобразование результата
    .thenAccept(result -> consume(result))     // потребление без возврата
    .thenRun(() -> doAfter());                 // действие после завершения

// Таймаут (Java 9+)
future.orTimeout(5, TimeUnit.SECONDS);         // бросает TimeoutException
future.completeOnTimeout(default, 5, SECONDS); // возвращает default при таймауте

// Комбинирование
CompletableFuture.allOf(f1, f2, f3).join();   // ждать все
CompletableFuture.anyOf(f1, f2, f3).join();   // ждать первый

// Обработка ошибок
future.exceptionally(ex -> fallback);          // при исключении
future.handle((result, ex) -> {                // и успех и ошибка
    if (ex != null) return fallback;
    return result;
});
```

### Важные нюансы

```java
// 1. Без executor — использует ForkJoinPool.commonPool()
// Если задачи блокирующие (I/O) — commonPool забивается
// Всегда передавай свой executor для I/O задач
CompletableFuture.runAsync(() -> blockingCall(), ioExecutor);

// 2. join() vs get()
future.get();    // бросает checked исключения (InterruptedException, ExecutionException)
future.join();   // бросает unchecked (CompletionException) — удобнее в стримах

// 3. allOf не возвращает результаты — нужно собирать вручную
List<CompletableFuture<String>> futures = ...;
CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
    .thenApply(v -> futures.stream()
        .map(CompletableFuture::join)  // join безопасен — все уже завершены
        .collect(Collectors.toList()));
```

---

## 2. Параллельные джобы между инстансами — Quartz Scheduler

### Проблема с @Scheduled + Shedlock

```
Shedlock — гарантирует что только ОДИН инстанс выполняет джобу
Это нужно чтобы не было дублирования
Но это означает — нет горизонтального масштабирования джоб
```

### Quartz Scheduler — решение

Quartz хранит джобы в БД и сам координирует между инстансами.

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-quartz</artifactId>
</dependency>
```

```yaml
spring:
  quartz:
    job-store-type: jdbc          # хранить джобы в БД
    jdbc:
      initialize-schema: always   # создать таблицы автоматически
    properties:
      org.quartz:
        jobStore:
          isClustered: true       # кластерный режим — главное для распределения
          clusterCheckinInterval: 5000
        scheduler:
          instanceId: AUTO        # уникальный ID каждого инстанса
```

```java
// Создание джобы
@Component
public class SmsJob implements Job {
    @Autowired
    private SmsService smsService;

    @Override
    public void execute(JobExecutionContext context) {
        smsService.sendPending();
    }
}

// Регистрация
@Configuration
public class QuartzConfig {
    @Bean
    public JobDetail smsJobDetail() {
        return JobBuilder.newJob(SmsJob.class)
            .withIdentity("smsJob")
            .storeDurably()
            .build();
    }

    @Bean
    public Trigger smsJobTrigger() {
        return TriggerBuilder.newTrigger()
            .forJob(smsJobDetail())
            .withSchedule(CronScheduleBuilder.cronSchedule("0/30 * * * * ?"))
            .build();
    }
}
```

### Quartz vs @Scheduled + Shedlock

| | @Scheduled + Shedlock | Quartz |
|---|---|---|
| Хранение джоб | в памяти | в БД |
| Кластеризация | ❌ один инстанс | ✅ распределение между инстансами |
| Retry при падении | ❌ | ✅ |
| История выполнения | ❌ | ✅ |
| Сложность | низкая | высокая |
| Когда использовать | простые джобы | критичные распределённые джобы |

### Как Quartz распределяет джобы

```
Инстанс 1, 2, 3 — все подключены к одной БД с таблицами Quartz
Каждый инстанс регулярно делает check-in в БД (heartbeat)
Когда наступает время джобы — один инстанс захватывает её через lock в БД
Остальные видят что джоба захвачена — не трогают
Если инстанс упал (нет heartbeat) — другой инстанс подхватывает его джобы
```

---

## 3. Row-level lock в PostgreSQL

### Как возникает автоматически

```sql
-- UPDATE автоматически ставит exclusive row-level lock
UPDATE users SET age = 10 WHERE user_id = 1;
-- Строка заблокирована до COMMIT или ROLLBACK
-- Другие транзакции которые хотят UPDATE эту строку — ждут
```

### Разбор примера с собеса

```
Начало: user_id=1, age=5

1. T1: begin
2. T2: begin
3. T1: UPDATE age=10  → T1 захватила row lock
4. T2: UPDATE age=15  → T2 ЖДЁТ (строка заблокирована T1)
5. T2: commit         → T2 ещё в очереди, не выполнилась
6. T1: commit         → T1 освобождает lock, age=10
                      → T2 получает lock, выполняет UPDATE age=15, коммитит

Итог: age = 15 (не 10!)
```

**Порядок в коде ≠ порядок выполнения.** T2 физически не может обновить строку пока T1 не закоммитит.

### Виды блокировок в PostgreSQL

```sql
-- 1. Автоматическая при UPDATE/DELETE — exclusive
UPDATE users SET age = 10 WHERE id = 1; -- exclusive lock

-- 2. SELECT FOR UPDATE — явная блокировка при чтении
SELECT * FROM users WHERE id = 1 FOR UPDATE;
-- Используется для: прочитать → обработать → обновить без race condition

-- 3. SELECT FOR SHARE — разделяемая блокировка
SELECT * FROM users WHERE id = 1 FOR SHARE;
-- Блокирует писателей, разрешает читателей

-- 4. SKIP LOCKED — пропустить заблокированные строки
SELECT * FROM messages WHERE status = 'pending'
FOR UPDATE SKIP LOCKED
LIMIT 10;
-- Идеально для очередей задач — каждый воркер берёт свои строки
```

---

## 4. Оптимистичная блокировка — реализация

### Суть

Не блокируем строку при чтении. При записи проверяем — не изменил ли кто-то данные пока мы работали. Если изменил — retry.

### Реализация через версию в SQL

```sql
-- Добавить колонку версии
ALTER TABLE users ADD COLUMN version INTEGER DEFAULT 0;

-- Чтение
SELECT id, age, version FROM users WHERE id = 1;
-- Получили: id=1, age=5, version=3

-- Обновление с проверкой версии
UPDATE users
SET age = 10, version = version + 1
WHERE id = 1 AND version = 3;  -- проверяем что версия не изменилась

-- Если вернул 0 строк — кто-то уже изменил данные → retry
```

### Пример двух конкурентных транзакций

```
T1 читает: age=5, version=3
T2 читает: age=5, version=3

T1: UPDATE SET age=10, version=4 WHERE id=1 AND version=3 → 1 строка ✅
T2: UPDATE SET age=15, version=4 WHERE id=1 AND version=3 → 0 строк ❌
    → T2 делает retry: перечитывает age=10, version=4
    → UPDATE SET age=15, version=5 WHERE id=1 AND version=4 → 1 строка ✅
```

### Реализация в JPA/Hibernate

```java
@Entity
public class User {
    @Id
    private Long id;
    private int age;

    @Version                    // Hibernate сам управляет версией
    private Integer version;    // автоматически проверяет при UPDATE
}

// При конфликте Hibernate бросает:
// OptimisticLockException → можно поймать и сделать retry
```

### Оптимистичная vs Пессимистичная

| | Оптимистичная | Пессимистичная |
|---|---|---|
| Блокировка БД | ❌ | ✅ SELECT FOR UPDATE |
| Конфликты | Редкие | Частые |
| Deadlock | Невозможен | Возможен |
| Производительность | Лучше при редких конфликтах | Лучше при частых конфликтах |
| Retry | Нужен | Не нужен |

---

## 5. Deadlock между двумя аккаунтами

### Проблема

```
Account A → Account B (T1):
  T1: lock(A) → пытается lock(B)

Account B → Account A (T2):
  T2: lock(B) → пытается lock(A)

T1 ждёт T2, T2 ждёт T1 → deadlock
```

### Решение 1 — упорядочивание по ID (твой ответ)

```java
// Всегда блокируем аккаунты в одном порядке — по ID
void transfer(Account from, Account to, BigDecimal amount) {
    Account first  = from.getId() < to.getId() ? from : to;
    Account second = from.getId() < to.getId() ? to : from;

    synchronized(first) {       // всегда меньший ID первый
        synchronized(second) {
            from.debit(amount);
            to.credit(amount);
        }
    }
}

// T1: transfer(A=1, B=2) → lock(1) → lock(2) — порядок: 1,2
// T2: transfer(B=2, A=1) → lock(1) → lock(2) — порядок: 1,2 (тот же!)
// Deadlock невозможен — оба всегда берут блокировки в одном порядке
```

### Решение 2 — SELECT FOR UPDATE с упорядочиванием (SQL уровень)

```sql
-- Всегда выбираем оба аккаунта одним запросом в порядке ID
SELECT * FROM accounts
WHERE id IN (1, 2)
ORDER BY id          -- фиксированный порядок блокировки
FOR UPDATE;

-- PostgreSQL заблокирует строки в порядке ORDER BY
-- Deadlock между двумя транзакциями невозможен
```

### Мегавариант — сериализация через очередь

Это решение которое упомянул интервьюер — убрать конкурентность полностью.

```
Все операции с аккаунтом → в очередь (Kafka/Redis)
Один воркер на аккаунт обрабатывает последовательно
Deadlock физически невозможен — нет конкурентного доступа
```

```java
// Ключ сообщения в Kafka = account_id
// Все операции одного аккаунта → одна партиция → один consumer
// Строго последовательная обработка на уровне аккаунта

producer.send(new ProducerRecord<>(
    "transfers",
    accountId.toString(),    // ключ → одна партиция
    transferEvent
));
```

**Преимущества:**
- Deadlock невозможен физически
- Горизонтальное масштабирование — разные аккаунты на разных воркерах
- Audit log из коробки — Kafka хранит все события

---

## 6. Thread per request vs Event Loop vs Virtual Threads

### Thread per request (Spring MVC)

```
Запрос 1 → Поток 1 (занят до ответа)
Запрос 2 → Поток 2 (занят до ответа)
...
Запрос 51 → очередь ожидания (или отказ если очередь полна)

Tomcat defaults:
  max-threads: 200      (максимум потоков)
  accept-count: 100     (размер очереди)
  → максимум 300 одновременных соединений
```

**Проблема:** поток заблокирован пока ждёт БД/HTTP — OS thread занят но ничего не делает.

### Что происходит с 51-м запросом

```
50 потоков заняты → 51-й запрос идёт в очередь accept-count
Если очередь тоже полна → Connection refused
Если в очереди есть место → ждёт пока освободится поток

Настройка:
server:
  tomcat:
    max-threads: 200
    accept-count: 100
```

### Event Loop (WebFlux)

```
4 потока (= CPU ядра) обрабатывают тысячи запросов
Поток не блокируется на I/O — отдаёт управление другим запросам
При получении ответа — продолжает обработку

Минус: весь код должен быть реактивным (Mono/Flux)
       нельзя использовать блокирующие библиотеки
```

### Virtual Threads — Java 21 (лучшее решение без WebFlux)

```java
// Виртуальный поток — очень дешёвый (~1KB vs ~1MB для OS потока)
// Можно создать миллион виртуальных потоков
// При блокировании на I/O — OS поток освобождается автоматически
// Блокирующий код работает как прежде — не нужен реактивный стиль
```

```yaml
# Spring Boot 3.2+ — одна строка конфига
spring:
  threads:
    virtual:
      enabled: true
# Tomcat начинает использовать виртуальные потоки для каждого запроса
```

```java
// Или вручную
@Bean
public TomcatProtocolHandlerCustomizer<?> virtualThreadsProtocol() {
    return handler -> handler.setExecutor(Executors.newVirtualThreadPerTaskExecutor());
}
```

### Сравнение

| | MVC + OS Threads | WebFlux | Virtual Threads |
|---|---|---|---|
| Модель | thread per request | event loop | virtual thread per request |
| Код | обычный блокирующий | реактивный Mono/Flux | обычный блокирующий |
| Потоки | ~200 OS | ~CPU ядер | миллионы виртуальных |
| Миграция | — | сложная, переписывать | простая, одна строка |
| Java версия | любая | любая | 21+ |
| Подходит | низкая нагрузка | высокая нагрузка, I/O | высокая нагрузка, I/O |

### Connection Pool — важная деталь

```
Virtual Threads решают проблему потоков
Но connection pool к БД остаётся ограниченным!

100_000 виртуальных потоков → все хотят соединение с БД
HikariCP по умолчанию: 10 соединений

Решение: увеличить pool или использовать R2DBC
spring:
  datasource:
    hikari:
      maximum-pool-size: 50
```

---

## Итоговый чеклист

### CompletableFuture:
- [ ] runAsync / supplyAsync — разница
- [ ] orTimeout / completeOnTimeout
- [ ] allOf / anyOf
- [ ] exceptionally / handle
- [ ] Всегда передавать свой executor для I/O

### Quartz:
- [ ] job-store-type: jdbc
- [ ] isClustered: true
- [ ] JobDetail + Trigger
- [ ] Отличие от @Scheduled + Shedlock

### PostgreSQL блокировки:
- [ ] UPDATE автоматически ставит row-level lock
- [ ] SELECT FOR UPDATE — явная блокировка
- [ ] SKIP LOCKED — для очередей задач
- [ ] Порядок выполнения ≠ порядок в коде

### Оптимистичная блокировка:
- [ ] Колонка version в таблице
- [ ] WHERE id=? AND version=? при UPDATE
- [ ] 0 строк → retry
- [ ] @Version в JPA

### Deadlock:
- [ ] Упорядочивание блокировок по ID
- [ ] SELECT FOR UPDATE ORDER BY id
- [ ] Сериализация через Kafka — мегавариант

### Threads:
- [ ] max-threads Tomcat, accept-count
- [ ] 51-й запрос → очередь → отказ
- [ ] Virtual Threads Java 21 — spring.threads.virtual.enabled=true
- [ ] Connection pool остаётся узким местом даже с Virtual Threads
