# Executor Framework и CompletableFuture

## За две минуты

- Executor задаёт, где исполняется задача; размер пула и очередь определяют нагрузку при наплыве задач.
- `thenCompose` разворачивает зависимый future, `thenCombine` объединяет независимые результаты.
- Отмена future и timeout ожидания не гарантируют остановку удалённой операции.

---

## Executor: что ограничивает параллелизм

`Executor` принимает задачу; `ExecutorService` управляет пулом и возвращает `Future`. У `ThreadPoolExecutor` есть core/max pool, очередь, keepAlive и rejection policy. Размер пула сам по себе не защищает память: если очередь бесконечна, входящий поток задач способен заполнить её. Для блокирующего I/O учитывай также лимиты БД и внешних сервисов.

`Future.get()` ждёт результат, `get(timeout)` ограничивает ожидание, `cancel(true)` запрашивает прерывание, но не гарантирует остановку I/O. `CompletableFuture` добавляет зависимые стадии и комбинирование; без явного executor методы `supplyAsync/runAsync` обычно используют `ForkJoinPool.commonPool()`. Для блокирующих вызовов передавай выделенный и ограниченный executor.

| Задача | Метод |
|---|---|
| `T → U` синхронно | `thenApply` |
| `T → CompletableFuture<U>`, убрать вложенность | `thenCompose` |
| два независимых результата | `thenCombine` |
| дождаться всех | `allOf` + получить значения исходных future |
| fallback при ошибке | `exceptionally` или `handle` |

```java
CompletableFuture<User> user =
    CompletableFuture.supplyAsync(() -> loadUser(id), ioPool);
CompletableFuture<Limits> limits =
    CompletableFuture.supplyAsync(() -> loadLimits(id), ioPool);

CompletableFuture<Decision> decision =
    user.thenCombine(limits, (u, l) -> evaluate(u, l));
```

`orTimeout` завершает **сам future** ошибкой по истечении времени, но не обещает остановить блокирующую операцию или отменить внешний запрос. Ограничивай таймауты у клиента, размер пула, очередь и общее время операции. После использования `ExecutorService` закрывай/останавливай его; при `InterruptedException` продумывай отмену и восстановление флага.

## CountDownLatch vs Semaphore

`CountDownLatch(n)` позволяет ждать, пока n событий вызовут `countDown()`; счётчик одноразовый. `Semaphore(k)` выдаёт до k разрешений на одновременную работу и возвращает их через `release()` в `finally`. Это разные задачи: «дождаться старта трёх» и «не больше десяти внешних запросов».

По тесту были пробелы в `thenCompose/thenCombine/allOf` и лимитах параллелизма; подробный пример SMS находится в [[Java Core/Concurrency/CompletableFuture-Interview]].

Источники: [CompletableFuture JDK 21](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/CompletableFuture.html), [ThreadPoolExecutor](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/ThreadPoolExecutor.html).
