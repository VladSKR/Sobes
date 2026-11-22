# Вопросы после интервью

## CompletableFuture

[Глава про executor и Future](../Java%20Core/Concurrency/Executors-CompletableFuture.md)

- [ ] runAsync / supplyAsync — разница
- [ ] orTimeout / completeOnTimeout
- [ ] allOf / anyOf
- [ ] exceptionally / handle
- [ ] Для блокирующего I/O определить отдельный ограниченный executor и таймаут клиента

## Quartz

[Глава про Quartz](../Operations/Quartz-Scheduling.md)

- [ ] job-store-type: jdbc
- [ ] isClustered: true
- [ ] JobDetail + Trigger
- [ ] Отличие от @Scheduled + Shedlock

### PostgreSQL блокировки:
- [ ] UPDATE автоматически ставит row-level lock
- [ ] SELECT FOR UPDATE — явная блокировка
- [ ] SKIP LOCKED — для очередей задач
- [ ] Порядок выполнения ≠ порядок в коде

### Оптимистичная блокировка:
- [ ] Колонка version в таблице
- [ ] WHERE id=? AND version=? при UPDATE
- [ ] 0 строк → конфликт: решить, повторять ли **всю** транзакцию
- [ ] @Version в JPA

### Deadlock:
- [ ] Упорядочивание блокировок по ID
- [ ] Как гарантировать согласованный порядок захвата строк и когда всё равно нужен retry
- [ ] Почему сериализация через Kafka по одному ключу не решает автоматически перевод между двумя счетами

### Threads:
- [ ] server.tomcat.threads.max, max-connections и accept-count
- [ ] Что происходит при заполненном пуле Tomcat и очередях; почему нет универсального «51-й запрос»
- [ ] Virtual Threads Java 21 — spring.threads.virtual.enabled=true
- [ ] Connection pool остаётся узким местом даже с Virtual Threads
