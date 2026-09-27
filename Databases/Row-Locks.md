# Row-level lock в PostgreSQL

## 3. Row-level lock в PostgreSQL

### Как возникает автоматически

```sql
-- UPDATE автоматически ставит exclusive row-level lock
UPDATE users SET age = 10 WHERE user_id = 1;
-- Строка заблокирована до COMMIT или ROLLBACK
-- Другие транзакции которые хотят UPDATE эту строку — ждут
```

### Разбор примера с собеса

```
Начало: user_id=1, age=5

1. T1: begin
2. T2: begin
3. T1: UPDATE age=10  → T1 захватила row lock
4. T2: UPDATE age=15  → T2 ЖДЁТ (строка заблокирована T1)
5. T2: commit         → T2 ещё в очереди, не выполнилась
6. T1: commit         → T1 освобождает lock, age=10
                      → T2 получает lock, выполняет UPDATE age=15, коммитит

Итог: age = 15 (не 10!)
```

**Порядок в коде ≠ порядок выполнения.** T2 физически не может обновить строку пока T1 не закоммитит.

### Виды блокировок в PostgreSQL

```sql
-- 1. Автоматическая при UPDATE/DELETE — exclusive
UPDATE users SET age = 10 WHERE id = 1; -- exclusive lock

-- 2. SELECT FOR UPDATE — явная блокировка при чтении
SELECT * FROM users WHERE id = 1 FOR UPDATE;
-- Используется для: прочитать → обработать → обновить без race condition

-- 3. SELECT FOR SHARE — разделяемая блокировка
SELECT * FROM users WHERE id = 1 FOR SHARE;
-- Блокирует писателей, разрешает читателей

-- 4. SKIP LOCKED — пропустить заблокированные строки
SELECT * FROM messages WHERE status = 'pending'
FOR UPDATE SKIP LOCKED
LIMIT 10;
-- Идеально для очередей задач — каждый воркер берёт свои строки
```

---
