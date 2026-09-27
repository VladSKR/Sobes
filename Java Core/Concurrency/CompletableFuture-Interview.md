# CompletableFuture — параллельная отправка SMS с таймаутом

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
