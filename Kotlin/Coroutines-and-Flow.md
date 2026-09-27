# Kotlin coroutines: Continuation, отмена, Flow

## За две минуты

- `suspend` приостанавливает корутину на подходящем API, не обязательно поток и не делает JDBC неблокирующим.
- Компилятор сохраняет состояние выполнения в Continuation, а dispatcher выбирает поток продолжения.
- Cold Flow запускает upstream при подписке; SharedFlow/StateFlow существуют независимо от конкретного подписчика.

---

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

## Virtual threads и отличие

Java virtual thread сохраняет собственную модель Thread и может быть смонтирован на carrier; при поддерживаемом блокирующем I/O JVM освобождает carrier и возобновляет virtual thread после готовности операции. Coroutine явно приостанавливается в suspend API через continuation. Ни один подход не отменяет лимит соединений БД/памяти/внешнего сервиса: при двух миллионах запросов понадобятся bounded queues, backpressure и admission control. Для JDK 21 есть нюансы pinning; оценивай конкретную версию и операции.

Это были реальные пробелы: где хранится «стек» coroutine, как она продолжает работу, почему `async` иногда выполняется последовательно и чем hot отличается от cold. Понимание базового dispatcher уже было хорошим.

Источники: [Kotlin coroutines](https://kotlinlang.org/docs/coroutines-overview.html), [Flow](https://kotlinlang.org/docs/coroutines-flow.html), [JDK 21 virtual threads](https://docs.oracle.com/en/java/javase/21/core/virtual-threads.html).
