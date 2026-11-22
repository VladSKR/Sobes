# @Transactional: от прокси до COMMIT

## Путь внешнего вызова

```text
client → Spring proxy → TransactionInterceptor
       → PlatformTransactionManager → ресурс/транзакция
       → target.method() → commit или rollback → cleanup
```

`@Transactional` применяется к вызову, который прошёл через прокси. При обычном вызове `this.otherMethod()` внутри того же объекта прокси обходится и настройки аннотации у `otherMethod` не применяются. Исправление: вынести транзакционный метод в другой bean и вызывать его извне либо использовать явный `TransactionTemplate`. Поведение конкретного прокси зависит от конфигурации; не делай вывод только по наличию интерфейса.

В JDBC менеджер связывает `Connection` с текущим потоком через механизм Spring `TransactionSynchronizationManager`, выставляет границы транзакции и по завершении освобождает ресурс. В JPA `JpaTransactionManager` координирует `EntityManager` и его транзакцию; репозиторий Spring Data работает через EntityManager, даже когда ты напрямую его не пишешь. `EntityManager` управляет persistence context: изменения **managed** entity могут попасть в БД через dirty checking/flush без явного `save()`. `flush` отправляет SQL, но не означает commit; rollback отменит изменения транзакции.

```java
@Transactional
public void rename(long id, String name) {
    User user = entityManager.find(User.class, id); // managed
    user.setName(name); // flush/commit синхронизирует состояние
}
```

По умолчанию Spring откатывает при неперехваченном `RuntimeException` или `Error`; checked exception сам по себе не требует rollback. Для `IOException` при необходимости укажи `@Transactional(rollbackFor = IOException.class)`. Перехваченное внутри метода исключение не попадёт в interceptor: если нужен rollback, пробрось его или выстави rollback-only. `@Transactional` охватывает **свои** ресурсы: запрос в Redis, HTTP-платёж и Kafka publish не становятся частью той же DB-транзакции автоматически.

Для Reactor/R2DBC применяют reactive transaction management, где контекст транзакции идёт через Reactor Context; обычный thread-bound подход не переносится автоматически через произвольное переключение потоков. См. [[Reactive/Reactive-Streams-and-WebFlux]].

## Как propagation меняет границу

Пусть внешний `placeOrder()` уже открыл транзакцию. Вызов другого бина с `REQUIRED` присоединяет внутренний метод к ней. Если внутренний метод при сбое пометил её rollback-only, перехват исключения во внешнем методе не восстановит возможность commit. При попытке фиксации Spring может выбросить `UnexpectedRollbackException`.

`REQUIRES_NEW` приостанавливает внешнюю транзакцию и открывает независимую. Например, запись аудита может сохраниться при откате внешней операции. Для JDBC понадобятся дополнительные соединения, поскольку внешняя транзакция удерживает свой ресурс. `NESTED` использует savepoint при поддержке менеджера и ресурса, а окончательный commit остаётся общим. Эти настройки применяются, когда вызов проходит через транзакционный interceptor.

[Spring transaction propagation](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/tx-propagation.html)

## Что отвечать на собеседовании

1. Назови proxy и `TransactionInterceptor`.
2. Объясни выбранный `PlatformTransactionManager` и ресурс: JDBC connection или JPA EntityManager.
3. Скажи про self-invocation и границы rollback.
4. На примере managed entity отдели `save()`, `flush` и `commit`.
5. Назови внешние побочные эффекты и [[Architecture/Transactional-Outbox-and-Idempotency|outbox]].

[Контейнер и прокси](Overview.md) · [Состояния сущностей JPA](../Data/Overview.md)

## Two-Minute Drill

- Внешний вызов идёт через proxy и TransactionInterceptor к TransactionManager, который управляет ресурсом.
- `flush` отправляет SQL, `commit` фиксирует транзакцию; managed entity может обновиться без явного `save`.
- HTTP/Redis/Kafka не откатываются автоматически вместе с SQL.

**Проверь себя.** Почему внешний метод может получить UnexpectedRollbackException после перехвата внутренней ошибки?

Метод REQUIRED мог пометить общую транзакцию rollback-only, поэтому зафиксировать её уже невозможно.

Источники: [Spring transaction internals](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/tx-decl-explained.html), [rollback rules](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/rolling-back.html), [proxy limitations](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/annotations.html).
