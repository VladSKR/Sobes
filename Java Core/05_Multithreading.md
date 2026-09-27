# Многопоточность

#java #concurrency #threads #core

---

## JMM

> **Java Memory Model определяет правила видимости данных, допустимые переупорядочивания инструкций и гарантии синхронизации между потоками. Без отношения happens-before один поток не обязан видеть изменения другого. `volatile`, `synchronized`, `Lock` и атомарные классы создают эти гарантии, но делают это по-разному: `volatile` обеспечивает видимость и порядок, `synchronized` и `Lock` — видимость плюс взаимное исключение, а атомики — атомарные операции на основе CAS с необходимыми гарантиями памяти.**


## Runnable vs Callable

|                    | Runnable/lounch   | Callable/async<V>          |
| ------------------ | ----------------- | -------------------------- |
| Метод              | `run()`           | `call()`                   |
| Возвращает         | void              | V                          |
| Checked exceptions | ❌                 | ✅                          |
| Использование      | "fire-and-forget" | нужен результат/исключение |

```java
Runnable r = () -> System.out.println("done");

Callable<String> c = () -> {
    // может бросать checked exceptions
    return "result";
};

ExecutorService ex = Executors.newFixedThreadPool(3);
Future<String> future = ex.submit(c);
```

---

## Volatile

Гарантирует **видимость** изменений переменной между потоками.

```java
private volatile boolean running = true;

// Поток 1
running = false;

// Поток 2
while (running) { ... } // увидит изменение без volatile — не гарантировано
```

**Что даёт volatile:**
1. Чтение/запись из главной памяти (не из кэша потока)
2. Запрет reordering инструкций вокруг переменной

**Что НЕ даёт volatile:**
- Атомарность составных операций (`i++` — не атомарно даже с volatile)

---

## Future

```java
Future<String> future = executor.submit(() -> "result");

future.isDone();        // завершена?
future.isCancelled();   // отменена?
future.cancel(true);    // отмена (true = прервать поток)
future.get();           // блокирует до завершения
future.get(2, TimeUnit.SECONDS); // таймаут → TimeoutException
```

### Ограничения Future
- Нет цепочек (chaining)
- Нет комбинирования результатов
- Нет callback при завершении
- Нет обработки исключений в цепочках

### Паттерн "Таймаут с fallback"
```java
public <T> T executeWithTimeout(Callable<T> task, T fallback, long timeout, TimeUnit unit) {
    Future<T> future = executor.submit(task);
    try {
        return future.get(timeout, unit);
    } catch (TimeoutException e) {
        future.cancel(true);
        return fallback;
    } catch (Exception e) {
        return fallback;
    }
}
```

---

## CompletableFuture

Реализация Future с поддержкой callback, chaining, комбинирования.

```java
// Создание
CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> "result");

// С кастомным пулом
ExecutorService pool = Executors.newFixedThreadPool(3);
CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> "result", pool);

// Явное завершение
CompletableFuture<String> cf = new CompletableFuture<>();
cf.complete("value");
cf.completeExceptionally(new RuntimeException());
```

### Основные методы

```java
// Преобразование результата
cf.thenApply(s -> s.toUpperCase())        // синхронно
cf.thenApplyAsync(s -> s.toUpperCase())   // в другом потоке

// Потребление (void)
cf.thenAccept(s -> System.out.println(s))

// Выполнить после завершения (без доступа к результату)
cf.thenRun(() -> System.out.println("done"))

// Обработка исключения
cf.exceptionally(ex -> "fallback value")

// Обработка и успеха, и ошибки
cf.handle((result, ex) -> {
    if (ex != null) return "error";
    return result.toUpperCase();
})
```

### Комбинирование

```java
// Оба завершатся → объединить результаты
cf1.thenCombine(cf2, (r1, r2) -> r1 + r2)

// Оба завершатся → выполнить действие
cf1.thenAcceptBoth(cf2, (r1, r2) -> process(r1, r2))

// Первый завершившийся
CompletableFuture.anyOf(cf1, cf2, cf3)

// Все завершатся
CompletableFuture.allOf(cf1, cf2, cf3)
    .thenRun(() -> System.out.println("all done"))

// Цепочка зависимых задач
CompletableFuture.supplyAsync(() -> fetchUser(id))
    .thenApply(user -> enrichUser(user))
    .thenApply(user -> user.getName())
    .thenAccept(System.out::println)
    .exceptionally(ex -> { log(ex); return null; })
```

---

## Отслеживание исключений в потоках

```java
// Исключение в потоке — main НЕ упадёт
Thread t = new Thread(() -> {
    throw new RuntimeException("ошибка в потоке");
});

// Установить обработчик
t.setUncaughtExceptionHandler((thread, ex) -> {
    System.err.println("Поток " + thread.getName() + " упал: " + ex);
});

t.start();

// Через ExecutorService
Future<?> f = executor.submit(() -> { throw new RuntimeException(); });
try {
    f.get(); // !! только здесь узнаем об исключении
} catch (ExecutionException e) {
    Throwable cause = e.getCause();
}
```

---

## Синхронизация

```java
// synchronized метод
synchronized void method() { }

// synchronized блок
synchronized (lockObject) { }

// ReentrantLock
Lock lock = new ReentrantLock();
lock.lock();
try {
    // критическая секция
} finally {
    lock.unlock(); // всегда в finally
}

// Атомарные операции
AtomicInteger counter = new AtomicInteger(0);
counter.incrementAndGet();
counter.compareAndSet(expected, newValue);
```

---

## Жизненный цикл потока

```
NEW → RUNNABLE → (BLOCKED / WAITING / TIMED_WAITING) → TERMINATED
```

| Состояние | Причина |
|---|---|
| NEW | Создан, не запущен |
| RUNNABLE | Выполняется или готов к выполнению |
| BLOCKED | Ждёт монитор (synchronized) |
| WAITING | wait(), join(), park() |
| TIMED_WAITING | sleep(), wait(timeout), join(timeout) |
| TERMINATED | Завершён |
