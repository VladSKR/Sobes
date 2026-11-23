# Java Stream и parallelStream: Spliterator, ForkJoinPool

`Stream` — ленивый pipeline операций над источником данных. `parallelStream()` разрешает выполнение частей pipeline конкурентно, но не создаёт `CompletableFuture` на каждый элемент и не имеет отношения к Kotlin `Dispatchers.IO`. Предпосылки: [коллекции](Overview.md), [Executor Framework](../Concurrency/Executors-CompletableFuture.md).

## Порядок выполнения

```java
List<Integer> result = numbers.parallelStream()
    .map(this::cpuIntensiveCalculation)
    .toList();
```

Пока нет терминальной операции `toList()`, pipeline лишь описывается. После её вызова:
1. Источник предоставляет `Spliterator`, который может разделять данные на части через `trySplit()`.
2. Fork/Join-задачи исполняют части pipeline на worker-потоках.
3. Worker, у которого закончилась работа, может украсть задачу у другого (**work stealing**).
4. Частичные результаты объединяются согласно семантике терминальной операции.

Стандартный параллельный stream обычно использует `ForkJoinPool.commonPool()`; вызывающий поток тоже может участвовать. При 8 доступных CPU типичный target parallelism общего пула — 7 worker-потоков, **не** 100 потоков для 100 элементов; размер зависит от конфигурации JVM/пула. Worker-потоки не закреплены за конкретными ядрами CPU.

## Когда это полезно, а когда нет

Хороший кандидат: достаточно много **независимой CPU-bound** работы, которую удобно разделить, без shared mutable state. Переключение на parallel не гарантирует ускорение: дробление задач, синхронизация, объединение результатов и давление на память тоже имеют цену.

```java
users.parallelStream()
    .map(user -> restClient.get().uri("/users/{id}", user.id()).retrieve().body(User.class))
    .toList();
```

Это плохой выбор для массового блокирующего HTTP: worker ждёт I/O и не исполняет другие задачи. Если 8 потоков реально обрабатывают 100 независимых запросов по 1 секунде, нижняя оценка времени ~13 секунд (при идеальной равномерности), не 1 секунда. Кроме того, перегрузка общего пула влияет на другие его потребители. Для blocking I/O — отдельный bounded `Executor` с осмысленной очередью и политикой отказа; для неблокирующих вызовов — соответствующий I/O-клиент.

## Порядок, side effects и ограничения

- `forEach` на параллельном stream не сохраняет порядок вывода; `forEachOrdered` сохраняет encounter order ценой ограничений параллельности.
- `toList()` сохраняет encounter order для упорядоченного источника, хотя обработка частей происходит не в порядке поступления.
- Лямбды должны по возможности быть stateless и non-interfering. `ArrayList.add` из разных потоков — гонка.
- `Spliterator` разных источников разделяется с разной эффективностью; `ArrayList` обычно легче делить, чем некоторые другие источники.
- Это не общий асинхронный API: терминальная операция блокирует вызывающий код до получения результата (если не обёрнута отдельно).

## Two-Minute Drill

- `parallelStream` обычно использует `ForkJoinPool.commonPool`, `Spliterator` и work stealing, не `CompletableFuture` или `Dispatchers.IO`.
- Грубая параллельность определяется пулом и CPU, а не количеством элементов; вызывающий поток может участвовать.
- Независимые CPU-bound преобразования — кандидат; блокирующие HTTP/SQL внутри commonPool могут исчерпать workers.

**Проверь себя.** Что произойдёт, если `map` делает 100 блокирующих HTTP-запросов по секунде на машине с 8 CPU?

Несколько запросов выполняются одновременно, остальные ждут; общий пул ограничен, и другие его задачи тоже могут задержаться.

Источники: [Java Stream](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/stream/Stream.html) · [ForkJoinPool](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/ForkJoinPool.html) · [Spliterator](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/Spliterator.html).
