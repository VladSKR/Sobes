# Kotlin coroutines: Continuation, отмена, Flow

## Что происходит при suspend

Компилятор превращает suspend-функцию в state machine с `Continuation`: при реальной приостановке нужные локальные значения сохраняются в объекте состояния, вызов освобождает физический стек текущего потока, а continuation позже возобновляется на потоке выбранного dispatcher. `suspend` само по себе **не запускает новый поток** и не делает блокирующий JDBC неблокирующим. Готовность I/O сообщает соответствующий неблокирующий API/механизм событий; coroutine scheduler лишь исполняет продолжение.

`launch` возвращает Job, `async` возвращает Deferred результата. Для параллельности стартуй несколько `async` до первого `await`; два последовательных `withContext` выполняются последовательно. `coroutineScope` ждёт детей, при обычном падении ребёнка отменяет siblings; `supervisorScope` изолирует сбои детей, но ты обязан наблюдать их ошибки. `CancellationException` — нормальный сигнал отмены: общий `catch (Exception)` не должен её проглатывать.

```kotlin
suspend fun combine(): Result = coroutineScope {
    val a = async { serviceA.load() }
    val b = async { serviceB.load() }
    Result(a.await(), b.await())
}
```

`Dispatchers.Default` для CPU, `Dispatchers.IO` для обычного блокирующего I/O, если нельзя использовать неблокирующий клиент. R2DBC/WebClient обычно не требуют `Dispatchers.IO` ради ожидания ответа; сам тяжёлый CPU-код всё равно надо вынести. Подробнее — [[Kotlin/Dispatchers]].

**Cold Flow** исполняет upstream заново при каждой подписке; `SharedFlow` и `StateFlow` — hot, живут в своём scope и могут иметь нескольких подписчиков. `StateFlow` хранит текущее значение, новый collector его получает; `SharedFlow` конфигурируется для событий/replay. У Flow и Reactive Streams разные API, хотя оба описывают асинхронные потоки значений.

## Одна приостановка по шагам

Рассмотрим `suspend fun title(id: Long): String = client.load(id).title`. Если клиенту надо ждать сеть, он сохраняет continuation через адаптер API. Компилятор представляет возобновляемый участок как состояние с меткой следующей инструкции и нужными локальными значениями. Функция возвращает управление до границы запуска корутины, и поток может взяться за другую работу. Когда клиент получает ответ, он возобновляет continuation с результатом, а dispatcher решает, где исполнять следующий шаг. Если ответ уже был готов, фактической приостановки может не произойти.

Два вызова `val a = loadA(); val b = loadB()` последовательны даже внутри suspend-функции. Два `async` до первого `await()` позволяют запросам находиться в работе одновременно. Если один дочерний вызов в `coroutineScope` упал, общий scope отменит другой. Для границ и отмены см. [диспетчеры](Dispatchers.md) и [исключения](Exceptions.md).

В `flow { emit(repository.load()) }` чтение повторится для каждого `collect`. `stateIn(scope, started, initial)` превращает вычисление в общее состояние с последним значением и управлением запуска через `SharingStarted`. Для одноразовых событий выбери семантику replay и переполнения. Например, `SharedFlow` с `replay=0` не выдаст новому подписчику старое событие.

## Virtual threads и отличие

Java virtual thread сохраняет собственную модель Thread и может быть смонтирован на carrier; при поддерживаемом блокирующем I/O JVM освобождает carrier и возобновляет virtual thread после готовности операции. Coroutine явно приостанавливается в suspend API через continuation. Ни один подход не отменяет лимит соединений БД/памяти/внешнего сервиса: при двух миллионах запросов понадобятся bounded queues, backpressure и admission control. Для JDK 21 есть нюансы pinning; оценивай конкретную версию и операции.

[Модели исполнения серверов](../Architecture/Threading-Models.md) сопоставляют ожидание в MVC, WebFlux и virtual threads.

## Two-Minute Drill

- `suspend` приостанавливает корутину на подходящем API, не обязательно поток и не делает JDBC неблокирующим.
- Компилятор сохраняет состояние выполнения в Continuation, а dispatcher выбирает поток продолжения.
- Cold Flow запускает upstream при подписке; SharedFlow/StateFlow существуют независимо от конкретного подписчика.

**Проверь себя.** Почему два collect могут дважды обратиться к репозиторию?

Тело холодного Flow выполняется для каждой подписки. Для совместного состояния можно создать hot flow с управляемым scope.

Источники: [Kotlin coroutines](https://kotlinlang.org/docs/coroutines-overview.html), [Flow](https://kotlinlang.org/docs/coroutines-flow.html), [JDK 21 virtual threads](https://docs.oracle.com/en/java/javase/21/core/virtual-threads.html).
