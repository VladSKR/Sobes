# Пробелы Senior Java Developer — Детальный разбор

---

## 1. Многопоточность: synchronized, ReentrantLock, volatile

### volatile

Переменная помечается `volatile` чтобы сказать JVM: **не кэшируй это в регистре процессора или L1/L2 кэше потока — всегда читай из основной памяти**.

**Проблема без volatile:**
```java
// Поток 1 пишет
boolean running = true;

// Поток 2 читает — может никогда не увидеть изменение
// потому что JIT закэшировал значение в регистре
while (running) {
    doWork();
}
```

**С volatile:**
```java
volatile boolean running = true;
// Поток 2 всегда читает актуальное значение из RAM
```

**Что volatile гарантирует:**
- Видимость (visibility) — изменение в одном потоке сразу видно другим
- Happens-before — всё что было до записи в volatile, видно после чтения

**Что volatile НЕ гарантирует:**
- Атомарность составных операций

```java
volatile int counter = 0;
counter++; // НЕ атомарно! Это read → increment → write — три операции
// Два потока могут прочитать одновременно counter=5, оба запишут 6 вместо 7
```

**Когда использовать:**
- Простой флаг (boolean) который один поток пишет, другой читает
- Double-checked locking (см. ниже)

---

### synchronized

Обеспечивает **взаимное исключение** (mutual exclusion) — только один поток выполняет блок кода одновременно.

Каждый объект в Java имеет **монитор** (intrinsic lock). `synchronized` захватывает этот монитор.

```java
// На методе — монитор this
public synchronized void increment() {
    counter++;
}

// На блоке — можно выбрать любой объект как монитор
private final Object lock = new Object();

public void increment() {
    synchronized (lock) {
        counter++;
    }
}

// На статическом методе — монитор Class объекта
public static synchronized void staticMethod() {
    // монитор MyClass.class
}
```

**Что гарантирует:**
- Атомарность блока
- Видимость — при выходе из synchronized все записи flush в RAM

**Минусы:**
- Нельзя прервать ожидание (нет таймаута)
- Нельзя попробовать захватить без блокировки (tryLock)
- Нет разделения read/write блокировок — читатели тоже блокируют друг друга

---

### ReentrantLock

Явная блокировка с расширенными возможностями.

```java
private final ReentrantLock lock = new ReentrantLock();

public void increment() {
    lock.lock();
    try {
        counter++;
    } finally {
        lock.unlock(); // ВСЕГДА в finally, иначе deadlock при исключении
    }
}
```

**Дополнительные возможности:**

```java
// Попробовать без ожидания
if (lock.tryLock()) {
    try {
        // захватили
    } finally {
        lock.unlock();
    }
} else {
    // не захватили — делаем что-то другое
}

// С таймаутом
if (lock.tryLock(100, TimeUnit.MILLISECONDS)) {
    // ...
}

// Прерываемое ожидание
lock.lockInterruptibly(); // бросит InterruptedException если поток прерван

// Честная очередь (fair) — потоки получают блокировку в порядке ожидания
ReentrantLock fairLock = new ReentrantLock(true);
```

**ReentrantReadWriteLock — читатели не блокируют друг друга:**

```java
private final ReadWriteLock rwLock = new ReentrantReadWriteLock();

public String read() {
    rwLock.readLock().lock();
    try {
        return data; // много потоков могут читать одновременно
    } finally {
        rwLock.readLock().unlock();
    }
}

public void write(String value) {
    rwLock.writeLock().lock();
    try {
        data = value; // эксклюзивный доступ
    } finally {
        rwLock.writeLock().unlock();
    }
}
```

---

### Сравнение

| | synchronized | ReentrantLock | volatile |
|---|---|---|---|
| Атомарность | ✅ | ✅ | ❌ |
| Видимость | ✅ | ✅ | ✅ |
| Таймаут | ❌ | ✅ | — |
| tryLock | ❌ | ✅ | — |
| Прерывание | ❌ | ✅ | — |
| Read/Write разделение | ❌ | ✅ (RRWL) | — |
| Производительность | Хуже (JVM overhead) | Лучше при конкуренции | Лучший |

**Правило выбора:**
- Простой флаг → `volatile`
- Простая атомарная операция → `synchronized` или `AtomicInteger`
- Нужен таймаут / tryLock / прерывание → `ReentrantLock`
- Много читателей, мало писателей → `ReentrantReadWriteLock`

---

### happens-before

Концепция из Java Memory Model. Если операция A happens-before B — все изменения A видны B.

Устанавливается:
- Выход из `synchronized` → happens-before → вход следующего потока в тот же монитор
- Запись в `volatile` → happens-before → чтение того же volatile
- `Thread.start()` → happens-before → любое действие в запущенном потоке
- Любое действие в потоке → happens-before → `Thread.join()` в другом потоке

---

### Double-checked locking (пример volatile на практике)

```java
public class Singleton {
    // volatile ОБЯЗАТЕЛЕН
    private static volatile Singleton instance;

    public static Singleton getInstance() {
        if (instance == null) {                    // первая проверка без блокировки
            synchronized (Singleton.class) {
                if (instance == null) {            // вторая проверка с блокировкой
                    instance = new Singleton();    // без volatile другой поток может
                }                                  // увидеть частично инициализированный объект
            }
        }
        return instance;
    }
}
```

Без `volatile` — JIT может переупорядочить инструкции создания объекта, и другой поток увидит ненулевой указатель на незавершённый объект.

---

## 2. Kotlin Dispatchers

### Что такое Dispatcher

Dispatcher определяет **на каком потоке или пуле потоков** выполняется корутина.

```kotlin
// Явное указание
withContext(Dispatchers.IO) {
    // код выполняется в IO пуле
}

// Наследуется от родительского скоупа если не указан
launch {
    // dispatcher = dispatcher родителя
}
```

---

### Dispatchers.Default

- Пул потоков = `max(2, число ядер CPU)`
- Оптимизирован под CPU-intensive задачи
- Использовать: парсинг, сортировка, вычисления, сериализация

```kotlin
withContext(Dispatchers.Default) {
    val sorted = largeList.sortedBy { it.price } // CPU работа
    val parsed = heavyJson.map { parseComplexObject(it) }
}
```

**Антипаттерн — блокирующий I/O на Default:**
```kotlin
withContext(Dispatchers.Default) {
    jdbcTemplate.query("SELECT ...") // ПЛОХО — блокирует CPU поток
    // CPU пул = 8 потоков, все заняты ожиданием БД
    // остальные корутины встают
}
```

---

### Dispatchers.IO

- Пул потоков = `max(64, число ядер CPU)` (настраивается через `kotlinx.coroutines.io.parallelism`)
- Оптимизирован под блокирующие I/O операции
- Использовать: JDBC, файлы, блокирующие HTTP клиенты, любой блокирующий код

```kotlin
withContext(Dispatchers.IO) {
    val result = jdbcTemplate.query("SELECT ...") // OK
    val file = File("data.txt").readText()        // OK
    val response = blockingHttpClient.get(url)    // OK
}
```

**Важно:** IO и Default **разделяют потоки** через механизм `LimitedParallelism`. IO создаёт потоки поверх Default пула но с отдельным лимитом.

---

### Dispatchers.Unconfined

- Запускает корутину в **текущем потоке** до первой точки suspension
- После suspension продолжает в потоке который возобновил корутину
- Использовать: только для тестов или специфичных случаев

```kotlin
// НЕ использовать в production коде без понимания последствий
withContext(Dispatchers.Unconfined) {
    println(Thread.currentThread().name) // текущий поток
    delay(100)
    println(Thread.currentThread().name) // DefaultExecutor (поток delay)
}
```

---

### Dispatchers.Main

- UI поток в Android/Swing/JavaFX
- На бэкенде без UI фреймворка — **не существует**, бросит исключение

---

### Комбинирование с CoroutineContext

```kotlin
// Dispatcher + трейсинг контекст вместе
withContext(Dispatchers.IO + observationRegistry.asContextElement()) {
    // IO пул + Micrometer трейсинг сохранён
    userRepository.findById(id)
}
```

`CoroutineContext` — это map, элементы складываются через `+`.

---

### limitedParallelism — кастомный пул

```kotlin
// Создать пул с ограниченным параллелизмом
val limitedDispatcher = Dispatchers.IO.limitedParallelism(10)

withContext(limitedDispatcher) {
    // максимум 10 потоков одновременно
    // полезно для rate limiting к внешним API
}
```

---

### Structured Concurrency и Dispatchers

```kotlin
// Дочерние корутины наследуют dispatcher от родителя
val scope = CoroutineScope(Dispatchers.IO)

scope.launch {
    // IO
    launch {
        // тоже IO (наследует)
    }
    withContext(Dispatchers.Default) {
        // переключились на Default
        launch {
            // Default (наследует)
        }
    }
}
```

---

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

## 4. ZGC vs G1

### G1 — Garbage First

**Архитектура:**
- Heap делится на регионы одинакового размера (1MB - 32MB)
- Регионы динамически назначаются как Eden, Survivor, Old, Humongous
- GC приоритизирует регионы с наибольшим количеством мусора (отсюда название Garbage First)

**Stop-the-World паузы в G1:**
```
Minor GC (Young):  10-200ms  — зависит от размера Young generation
Mixed GC:          50-500ms  — очищает Young + часть Old
Full GC:           секунды   — если concurrent marking не успевает
```

**Когда G1 — правильный выбор:**
- Heap 4GB - 32GB
- Паузы до 200ms допустимы
- Серверные приложения без жёстких latency требований

---

### ZGC — Z Garbage Collector

**Ключевое отличие:** почти вся работа выполняется **параллельно с приложением** (concurrent).

**Stop-the-World паузы в ZGC:**
```
Всегда < 1-10ms независимо от размера heap
Даже на heap 1TB — пауза те же миллисекунды
```

**Как ZGC это делает:**

Использует **цветные указатели** (colored pointers) — в самих битах указателя хранит метаданные о состоянии объекта. Это позволяет делать маркировку и перемещение объектов без остановки приложения.

```
Обычный указатель:  [адрес объекта]
ZGC указатель:      [метаданные 4 бита][адрес объекта]
                     ↑ finalizable, remapped, marked0, marked1
```

**Фазы ZGC:**
```
STW: Pause Mark Start     (~1ms)  — корневые объекты
Concurrent: Mark          (параллельно с приложением)
STW: Pause Mark End       (~1ms)  — финализация маркировки
Concurrent: Prepare Reloc (параллельно)
STW: Pause Relocate Start (~1ms)  — начало перемещения
Concurrent: Relocate      (параллельно)
```

---

### Сравнение

| | G1 | ZGC |
|---|---|---|
| Паузы | 10ms - 500ms | < 10ms всегда |
| Зависимость паузы от heap | Да, растёт | Нет |
| Throughput | Выше | Немного ниже (~15%) |
| Подходящий heap | 4-32GB | 8MB - 16TB |
| Java версия | Java 9+ (default) | Java 15+ (production ready) |

---

### Когда переходить на ZGC

- Heap > 32GB
- Latency критична: trading, real-time системы, крипто-биржи
- Паузы G1 вызывают таймауты в downstream сервисах
- SLA требует p99 latency < 100ms

**Включение:**
```bash
java -XX:+UseZGC -Xmx16g -jar app.jar

# ZGC с generational режимом (Java 21+, лучший throughput)
java -XX:+UseZGC -XX:+ZGenerational -Xmx16g -jar app.jar
```

---

### Для крипто-биржи

G1 с паузой 200ms = 200ms без обработки ордеров. На высоколиквидном рынке это потери.

ZGC — правильный выбор для такой системы. Стоит поднять вопрос о переходе.

---

## 5. Race Condition — паттерны обнаружения

### Что такое race condition

Результат программы зависит от порядка выполнения потоков. Порядок недетерминирован.

```java
// Классический пример
int counter = 0;

// Поток 1          // Поток 2
read counter (0)    read counter (0)
increment → 1       increment → 1
write 1             write 1
// Ожидали 2, получили 1
```

---

### Check-then-act антипаттерн

```java
// ОПАСНО — между проверкой и действием может влезть другой поток
if (attempts < 3) {         // Поток 1 и Поток 2 оба видят attempts = 2
    attempts++;             // оба инкрементируют
    validateOtp(code);      // оба проходят проверку
}
```

**Решение — атомарная операция:**
```java
// AtomicInteger
AtomicInteger attempts = new AtomicInteger(0);
int current = attempts.incrementAndGet();
if (current > 3) {
    throw new TooManyAttemptsException();
}

// Redis INCR (атомарен на уровне Redis)
Long attempts = redisTemplate.opsForValue().increment("otp:attempts:" + userId);
if (attempts > 3) {
    blockUser(userId);
}
```

---

### Паттерны где нужно искать race condition

**1. Read-Modify-Write:**
```java
// Любая операция вида: читаем → меняем → пишем
value = map.get(key);
value++;
map.put(key, value); // ОПАСНО без синхронизации
```

**2. Check-Then-Act:**
```java
if (!map.containsKey(key)) {    // проверка
    map.put(key, compute());    // действие — между ними может влезть
}
// Решение:
map.computeIfAbsent(key, k -> compute()); // атомарно
```

**3. Lazy Initialization:**
```java
if (instance == null) {         // два потока проходят проверку
    instance = new Heavy();     // создают два объекта
}
// Решение: volatile + double-checked locking или Holder pattern
```

**4. Составные операции с коллекциями:**
```java
// ConcurrentHashMap не защищает составные операции
if (!concurrentMap.containsKey(key)) {
    concurrentMap.put(key, value); // ОПАСНО даже с ConcurrentHashMap
}
// Решение:
concurrentMap.putIfAbsent(key, value); // атомарно
```

---

### Финансовые системы — особое внимание

В финансовых системах race condition = потеря денег или двойное списание.

**Классическая проблема — двойное списание:**
```kotlin
// Поток 1 и Поток 2 одновременно проводят операцию
val balance = accountRepo.findBalance(userId) // оба читают 1000
if (balance >= amount) {                      // оба проходят
    accountRepo.deduct(userId, amount)        // оба списывают
}
// Итого: списали дважды
```

**Решения:**
```sql
-- Оптимистичная блокировка через версию
UPDATE accounts 
SET balance = balance - :amount, version = version + 1
WHERE id = :id AND version = :expectedVersion AND balance >= :amount
-- Если version изменилась — 0 строк обновлено → retry

-- Пессимистичная блокировка
SELECT * FROM accounts WHERE id = :id FOR UPDATE
-- Другие транзакции ждут
```

---

## Итоговый чеклист для подготовки

### Многопоточность — минимум знать:
- [ ] volatile: видимость, не атомарность
- [ ] synchronized: монитор, happens-before
- [ ] ReentrantLock: tryLock, таймаут, fairness
- [ ] ReadWriteLock: когда использовать
- [ ] AtomicInteger/AtomicReference
- [ ] ConcurrentHashMap и его атомарные операции
- [ ] ThreadLocal — что это и где используется (например в Spring транзакциях)
- [ ] ExecutorService, CountDownLatch, Semaphore

### Kotlin корутины — минимум знать:
- [ ] Dispatchers.IO vs Default vs Unconfined
- [ ] structured concurrency — отмена дочерних при падении родителя
- [ ] CoroutineScope и его жизненный цикл
- [ ] SupervisorJob — когда дочерние не отменяют друг друга
- [ ] Flow vs Channel

### Hibernate — минимум знать:
- [ ] batch_size настройка
- [ ] IDENTITY vs SEQUENCE — почему IDENTITY ломает батчинг
- [ ] N+1 проблема и решения (JOIN FETCH, EntityGraph, batch fetching)
- [ ] LazyInitializationException — почему возникает

### GC — минимум знать:
- [ ] G1: регионы, паузы, когда использовать
- [ ] ZGC: colored pointers, паузы <10ms, когда переходить
- [ ] Как читать GC логи
- [ ] OutOfMemoryError vs StackOverflowError — разница

### Race condition — рефлекс:
- [ ] Видеть check-then-act паттерн
- [ ] Знать атомарные альтернативы
- [ ] В финансовых системах — думать об этом в первую очередь
