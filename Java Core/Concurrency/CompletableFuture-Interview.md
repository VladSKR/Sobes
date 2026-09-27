# CompletableFuture — параллельная отправка SMS с таймаутом

## За две минуты

- Параллельные отправки требуют ограниченного пула и таймаута у самого сетевого клиента.
- `orTimeout` завершает future для ожидающего, но не обязательно останавливает уже запущенный send.
- Повтор SMS после неизвестного исхода требует идемпотентности или проверки статуса.

---

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

### Параллельная отправка без ложной гарантии отмены

```java
ExecutorService ioPool = Executors.newFixedThreadPool(10);
List<CompletableFuture<SendResult>> futures = messages.stream()
    .map(message -> CompletableFuture.supplyAsync(
        () -> smsGateway.sendWithClientTimeout(message), ioPool))
    .toList();

CompletableFuture<Void> all =
    CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]));
// all.join() дождётся завершения всех; ошибка одного future приведёт к CompletionException.
```

Настрой таймаут непосредственно у HTTP/SMS клиента и ограничь входящую очередь. Если добавить `orTimeout(5, SECONDS)`, future завершится исключением, но уже запущенная блокирующая отправка **может продолжиться**. Нельзя сразу помечать SMS как неотправленное и повторять его новым запросом без ключа идемпотентности/проверки результата провайдера. Запись состояния «отправлено» отделяй от сетевой отправки и проектируй восстановление после падения процесса. При закрытии приложения останови `ioPool`.

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
            } catch (ex: CancellationException) {
                throw ex // не проглатываем отмену родительской задачи
            } catch (ex: Exception) {
                logger.error("Error for message: ${message.id}", ex)
            }
        }
    }.awaitAll()                               // ждём все корутины
}
```

При блокирующем `smsGateway.send` корутинный `withTimeout` также не гарантирует физическую остановку сетевого вызова: таймаут нужен у самого клиента.

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
        .map(CompletableFuture::join)  // join не ждёт после allOf, но может бросить исключение задачи
        .collect(Collectors.toList()));
```

---
