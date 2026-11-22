# Kotlin Dispatchers

## Что такое Dispatcher

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

## Dispatchers.Default

- Обычно использует общий пул для CPU задач; точная степень параллелизма зависит от среды и настройки
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

## Dispatchers.IO

- Параллелизм блокирующего I/O по умолчанию ограничен параметром, связанным с `max(64, число ядер)`; его можно настроить
- Оптимизирован под блокирующие I/O операции
- Использовать: JDBC, файлы, блокирующие HTTP клиенты, любой блокирующий код

```kotlin
withContext(Dispatchers.IO) {
    val result = jdbcTemplate.query("SELECT ...") // OK
    val file = File("data.txt").readText()        // OK
    val response = blockingHttpClient.get(url)    // OK
}
```

**Важно:** IO и Default могут использовать одни и те же потоки планировщика. Переключение контекста не обязано означать переход на другой конкретный поток.

---

## Dispatchers.Unconfined

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

## Dispatchers.Main

- UI поток в Android/Swing/JavaFX
- На бэкенде без UI фреймворка — **не существует**, бросит исключение

---

## Комбинирование с CoroutineContext

```kotlin
// Dispatcher + трейсинг контекст вместе
withContext(Dispatchers.IO + CoroutineName("load-user")) {
    // IO dispatcher и имя корутины — разные элементы одного контекста
    userRepository.findById(id)
}
```

`CoroutineContext` — это map, элементы складываются через `+`.

---

## limitedParallelism — представление диспетчера

```kotlin
// Получить представление с ограниченным параллелизмом
val limitedDispatcher = Dispatchers.IO.limitedParallelism(10)

withContext(limitedDispatcher) {
    // не больше 10 одновременно исполняемых задач этого dispatcher
    // для ограничения числа незавершённых запросов используй Semaphore
}
```

---

## Structured Concurrency и Dispatchers

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

## Место исполнения и лимит операции

Представь 100 корутин, каждая открывает запрос через неблокирующий HTTP-клиент и приостанавливается до ответа. `limitedParallelism(10)` разрешает десяти участкам исполнять код одновременно. После приостановки место освобождается, поэтому незавершённых HTTP-запросов может стать 100. Если провайдер допускает максимум десять запросов одновременно, удерживай разрешение на весь вызов.

```kotlin
private val permits = Semaphore(10)

suspend fun load(id: Long): User =
    permits.withPermit { nonBlockingClient.load(id) }
```

Используй `kotlinx.coroutines.sync.Semaphore` и `withPermit`. При блокирующем JDBC понадобится `withContext(Dispatchers.IO)`, а pool соединений продолжит ограничивать БД. Во JVM `IO` и `Default` совместно используют потоки планировщика. Представления `IO.limitedParallelism` эластичны и имеют собственные лимиты, так что два представления по 50 могут вместе запустить больше работы, чем основной лимит IO.

[Dispatchers.IO](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-i-o.html) · [Контекст и диспетчеры](https://kotlinlang.org/docs/coroutine-context-and-dispatchers.html)

## Two-Minute Drill

- Default подходит для CPU задач, IO — для обычного блокирующего I/O; ни один dispatcher не исправит сам по себе перегрузку внешнего сервиса.
- `limitedParallelism(10)` ограничивает одновременно исполняемые задачи на view dispatcher, но не все незавершённые запросы.
- Для ограничения запросов к API используй semaphore/лимит на самой операции.

**Проверь себя.** Можно ли с limitedParallelism(1) иметь два HTTP-запроса в работе?

Да. После suspend диспетчер может исполнять следующую корутину. Для общего лимита незавершённых запросов удерживай permit до результата.
