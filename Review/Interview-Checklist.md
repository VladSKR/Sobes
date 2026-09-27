# Итоговый чеклист

## Итоговый чеклист

### CompletableFuture:
- [ ] runAsync / supplyAsync — разница
- [ ] orTimeout / completeOnTimeout
- [ ] allOf / anyOf
- [ ] exceptionally / handle
- [ ] Всегда передавать свой executor для I/O

### Quartz:
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
- [ ] 0 строк → retry
- [ ] @Version в JPA

### Deadlock:
- [ ] Упорядочивание блокировок по ID
- [ ] SELECT FOR UPDATE ORDER BY id
- [ ] Сериализация через Kafka — мегавариант

### Threads:
- [ ] max-threads Tomcat, accept-count
- [ ] 51-й запрос → очередь → отказ
- [ ] Virtual Threads Java 21 — spring.threads.virtual.enabled=true
- [ ] Connection pool остаётся узким местом даже с Virtual Threads
