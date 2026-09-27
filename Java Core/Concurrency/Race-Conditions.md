# Race Condition — паттерны обнаружения

## За две минуты

- Race condition — результат зависит от взаимного порядка конкурентных действий.
- `if (!exists) insert` и `read → update` небезопасны без атомарного механизма.
- Для БД подходят unique constraint, условный UPDATE или блокировка в одной транзакции.

---

## 5. Race Condition — паттерны обнаружения

### Что такое race condition

Результат программы зависит от порядка выполнения потоков. Порядок недетерминирован.

```java
// Классический пример
int counter = 0;

// Поток 1          // Поток 2
read counter (0)    read counter (0)
increment → 1       increment → 1
write 1             write 1
// Ожидали 2, получили 1
```

---

### Check-then-act антипаттерн

```java
// ОПАСНО — между проверкой и действием может влезть другой поток
if (attempts < 3) {         // Поток 1 и Поток 2 оба видят attempts = 2
    attempts++;             // оба инкрементируют
    validateOtp(code);      // оба проходят проверку
}
```

**Решение — атомарная операция:**
```java
// AtomicInteger
AtomicInteger attempts = new AtomicInteger(0);
int current = attempts.incrementAndGet();
if (current > 3) {
    throw new TooManyAttemptsException();
}

// Redis INCR (атомарен на уровне Redis)
Long attempts = redisTemplate.opsForValue().increment("otp:attempts:" + userId);
if (attempts > 3) {
    blockUser(userId);
}
```

---

### Паттерны где нужно искать race condition

**1. Read-Modify-Write:**
```java
// Любая операция вида: читаем → меняем → пишем
value = map.get(key);
value++;
map.put(key, value); // ОПАСНО без синхронизации
```

**2. Check-Then-Act:**
```java
if (!map.containsKey(key)) {    // проверка
    map.put(key, compute());    // действие — между ними может влезть
}
// Решение:
map.computeIfAbsent(key, k -> compute()); // атомарно
```

**3. Lazy Initialization:**
```java
if (instance == null) {         // два потока проходят проверку
    instance = new Heavy();     // создают два объекта
}
// Решение: volatile + double-checked locking или Holder pattern
```

**4. Составные операции с коллекциями:**
```java
// ConcurrentHashMap не защищает составные операции
if (!concurrentMap.containsKey(key)) {
    concurrentMap.put(key, value); // ОПАСНО даже с ConcurrentHashMap
}
// Решение:
concurrentMap.putIfAbsent(key, value); // атомарно
```

---

### Финансовые системы — особое внимание

В финансовых системах race condition = потеря денег или двойное списание.

**Классическая проблема — двойное списание:**
```kotlin
// Поток 1 и Поток 2 одновременно проводят операцию
val balance = accountRepo.findBalance(userId) // оба читают 1000
if (balance >= amount) {                      // оба проходят
    accountRepo.deduct(userId, amount)        // оба списывают
}
// Итого: списали дважды
```

**Решения:**
```sql
-- Оптимистичная блокировка через версию
UPDATE accounts 
SET balance = balance - :amount, version = version + 1
WHERE id = :id AND version = :expectedVersion AND balance >= :amount
-- Если version изменилась — 0 строк обновлено → retry

-- Пессимистичная блокировка
SELECT * FROM accounts WHERE id = :id FOR UPDATE
-- Другие транзакции ждут
```

---
