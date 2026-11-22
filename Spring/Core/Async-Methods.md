# @Async: как запустить метод асинхронно

`@Async` в Spring обычно работает через прокси: внешний вызов bean передаёт выполнение в настроенный `Executor`. Одной аннотации недостаточно: нужен `@EnableAsync`, управляемый Spring bean и подходящий executor. `this.asyncMethod()` внутри того же bean обходит прокси и выполняется обычным вызовом.

```java
@Configuration
@EnableAsync
class AsyncConfig {
    @Bean
    Executor taskExecutor() {
        return new ThreadPoolTaskExecutor(); // настрой размер и очередь в реальном сервисе
    }
}

@Service
class NotificationService {
    @Async
    public CompletableFuture<Void> sendAsync(String id) {
        send(id);
        return CompletableFuture.completedFuture(null);
    }
}
```

В примере конфигурация намеренно схематична: при реальном использовании `ThreadPoolTaskExecutor` следует инициализировать/настроить core/max pool, ограниченную очередь и rejection policy. Возврат `void` усложняет наблюдение ошибок; для результата/ошибки предпочти `CompletableFuture<T>` и обработку исключений. Поток другой — thread-bound контекст транзакции вызывающего **не переносится автоматически**; @Async метод может открыть свою транзакцию через внешний proxy. Для надёжной доставки после сбоя процесса нужен broker/outbox, а не только локальный пул.

Этот вопрос был на скрининге: «какой аннотацией запустить отдельно и что ещё нужно». Связанные темы: [[Java Core/Concurrency/Executors-CompletableFuture]] и [[Spring/Core/Transactions-and-JPA]].

## Куда попадают ошибка и перегрузка

Вызови асинхронный метод из другого Spring-бина и передай имя пула через `@Async("notificationExecutor")`. Настрой именованный `ThreadPoolTaskExecutor`, например core 4, max 8 и очередь 32. Пока очередь свободна, задачи сверх core обычно накапливаются в ней. Когда исчерпаны пул и очередь, отправка может завершиться `TaskRejectedException` уже у вызывающего кода. Исключение из **принятой** задачи приходит через возвращённый future. У `void`-метода оно обрабатывается через `AsyncUncaughtExceptionHandler`.

Если уведомление должно уйти после создания заказа, запуск фонового метода внутри ещё открытой транзакции может обогнать commit. Фоновая работа также потеряется при остановке процесса. Для устойчивой схемы с повтором запиши событие вместе с заказом и дай отдельному обработчику забрать его после commit. [Transactional Outbox](../../Architecture/Transactional-Outbox-and-Idempotency.md)

## Two-Minute Drill

- `@Async` передаёт вызов в Executor, когда метод вызван через Spring proxy.
- Вызов `this.method()` не проходит через proxy; транзакция вызывающего потока не переезжает в другой поток.
- Для долговечной доставки используй broker/outbox, а не один локальный пул.

**Проверь себя.** Перенесётся ли открытая транзакция в метод @Async?

Нет. Исполнение в другом потоке требует собственной транзакционной границы, если она нужна.

Источник: [Spring @Async](https://docs.spring.io/spring-framework/reference/integration/scheduling.html).
