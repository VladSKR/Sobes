# Kotlin Dispatchers

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
