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

Источник: [Spring @Async](https://docs.spring.io/spring-framework/reference/integration/scheduling.html).
