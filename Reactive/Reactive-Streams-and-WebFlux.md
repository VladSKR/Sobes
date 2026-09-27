# Reactive Streams, Reactor и WebFlux

## Backpressure — протокол спроса

`Publisher` отправляет сигналы `onSubscribe`, затем по запросу `Subscription.request(n)` выдаёт не больше запрошенного числа элементов, заканчивает `onComplete` либо `onError`. `request(n)` — способ подписчика регулировать спрос, **не просто очередь из лишних HTTP-запросов**. Для источника 1000 событий/с и потребителя 100/с нужно уменьшать производство, ограничивать конкуренцию/буфер либо явно решать, что делать при перегрузке; одна только очередь без предела рано или поздно заполнит память. Некоторые внешние источники не умеют замедляться — тогда нужны buffer/drop/error policies и контроль у границы системы.

`Mono<T>` выдаёт 0–1 значение, `Flux<T>` — 0..N. `map` преобразует обычное значение, `flatMap` запускает publisher для каждого элемента и допускает переплетение результатов, `concatMap` сохраняет последовательность за счёт последовательной подписки. `subscribeOn` влияет на контекст выполнения подписки upstream, `publishOn` переключает выполнение дальнейших операторов downstream. Прямой `block()` внутри event-loop может остановить обработку других запросов; блокирующий JDBC/файлы требуют отдельного пула или перехода на неблокирующий драйвер.

```java
Flux.fromIterable(ids)
    .flatMap(id -> webClient.get().uri("/users/{id}", id)
        .retrieve().bodyToMono(User.class), 8); // ограничение concurrency
```

В WebFlux маленькое число event-loop потоков обслуживает множество соединений, если цепочки не блокируют их. Потоки не отменяют лимиты CPU, памяти, connection pool, downstream и очередей. `@Transactional` для R2DBC использует reactive transaction manager и Reactor Context, см. [[Spring/Core/Transactions-and-JPA]].

**Быстрый ответ:** backpressure — сигнал спроса внутри потока данных; защита входящего API от 2 млн запросов требует отдельно rate limit, bulkhead, bounded concurrency, таймауты и перегрузочную политику. Это было главным недоразумением в нашей беседе.

Источники: [Reactive Streams specification](https://www.reactive-streams.org/), [Reactor reference](https://projectreactor.io/docs/core/release/reference/).
