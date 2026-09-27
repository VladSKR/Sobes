# Kotlin Dispatchers

## За две минуты

- Default подходит для CPU задач, IO — для обычного блокирующего I/O; ни один dispatcher не исправит сам по себе перегрузку внешнего сервиса.
- `limitedParallelism(10)` ограничивает одновременно исполняемые задачи на view dispatcher, но не все незавершённые запросы.
- Для ограничения запросов к API используй semaphore/лимит на самой операции.

---

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

### Dispatchers.IO

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

**Важно:** IO и Default **разделяют потоки** через механизм `LimitedParallelism`. IO и Default разделяют потоки; переключение контекста не обязано означать переход на другой конкретный поток.

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
withContext(Dispatchers.IO + CoroutineName("load-user")) {
    // IO dispatcher и имя корутины — разные элементы одного контекста
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
    // не больше 10 одновременно исполняемых задач этого dispatcher
    // для ограничения числа незавершённых запросов используй Semaphore
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
