# Оптимистичная блокировка — реализация

## 4. Оптимистичная блокировка — реализация

### Суть

Не блокируем строку при чтении. При записи проверяем — не изменил ли кто-то данные пока мы работали. Если изменил — retry.

### Реализация через версию в SQL

```sql
-- Добавить колонку версии
ALTER TABLE users ADD COLUMN version INTEGER DEFAULT 0;

-- Чтение
SELECT id, age, version FROM users WHERE id = 1;
-- Получили: id=1, age=5, version=3

-- Обновление с проверкой версии
UPDATE users
SET age = 10, version = version + 1
WHERE id = 1 AND version = 3;  -- проверяем что версия не изменилась

-- Если вернул 0 строк — кто-то уже изменил данные → retry
```

### Пример двух конкурентных транзакций

```
T1 читает: age=5, version=3
T2 читает: age=5, version=3

T1: UPDATE SET age=10, version=4 WHERE id=1 AND version=3 → 1 строка ✅
T2: UPDATE SET age=15, version=4 WHERE id=1 AND version=3 → 0 строк ❌
    → T2 делает retry: перечитывает age=10, version=4
    → UPDATE SET age=15, version=5 WHERE id=1 AND version=4 → 1 строка ✅
```

### Реализация в JPA/Hibernate

```java
@Entity
public class User {
    @Id
    private Long id;
    private int age;

    @Version                    // Hibernate сам управляет версией
    private Integer version;    // автоматически проверяет при UPDATE
}

// При конфликте Hibernate бросает:
// OptimisticLockException → можно поймать и сделать retry
```

### Оптимистичная vs Пессимистичная

| | Оптимистичная | Пессимистичная |
|---|---|---|
| Блокировка БД | ❌ | ✅ SELECT FOR UPDATE |
| Конфликты | Редкие | Частые |
| Deadlock | Невозможен | Возможен |
| Производительность | Лучше при редких конфликтах | Лучше при частых конфликтах |
| Retry | Нужен | Не нужен |

---
