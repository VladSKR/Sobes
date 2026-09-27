# Deadlock между двумя аккаунтами

## 5. Deadlock между двумя аккаунтами

### Проблема

```
Account A → Account B (T1):
  T1: lock(A) → пытается lock(B)

Account B → Account A (T2):
  T2: lock(B) → пытается lock(A)

T1 ждёт T2, T2 ждёт T1 → deadlock
```

### Решение 1 — упорядочивание по ID (твой ответ)

```java
// Всегда блокируем аккаунты в одном порядке — по ID
void transfer(Account from, Account to, BigDecimal amount) {
    Account first  = from.getId() < to.getId() ? from : to;
    Account second = from.getId() < to.getId() ? to : from;

    synchronized(first) {       // всегда меньший ID первый
        synchronized(second) {
            from.debit(amount);
            to.credit(amount);
        }
    }
}

// T1: transfer(A=1, B=2) → lock(1) → lock(2) — порядок: 1,2
// T2: transfer(B=2, A=1) → lock(1) → lock(2) — порядок: 1,2 (тот же!)
// Deadlock невозможен — оба всегда берут блокировки в одном порядке
```

### Решение 2 — SELECT FOR UPDATE с упорядочиванием (SQL уровень)

```sql
-- Всегда выбираем оба аккаунта одним запросом в порядке ID
SELECT * FROM accounts
WHERE id IN (1, 2)
ORDER BY id          -- фиксированный порядок блокировки
FOR UPDATE;

-- PostgreSQL заблокирует строки в порядке ORDER BY
-- Deadlock между двумя транзакциями невозможен
```

### Мегавариант — сериализация через очередь

Это решение которое упомянул интервьюер — убрать конкурентность полностью.

```
Все операции с аккаунтом → в очередь (Kafka/Redis)
Один воркер на аккаунт обрабатывает последовательно
Deadlock физически невозможен — нет конкурентного доступа
```

```java
// Ключ сообщения в Kafka = account_id
// Все операции одного аккаунта → одна партиция → один consumer
// Строго последовательная обработка на уровне аккаунта

producer.send(new ProducerRecord<>(
    "transfers",
    accountId.toString(),    // ключ → одна партиция
    transferEvent
));
```

**Преимущества:**
- Deadlock невозможен физически
- Горизонтальное масштабирование — разные аккаунты на разных воркерах
- Audit log из коробки — Kafka хранит все события

---
