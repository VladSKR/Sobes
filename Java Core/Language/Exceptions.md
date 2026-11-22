# Exceptions (Исключения)

## Иерархия

Все объекты, которые можно выбросить через `throw`, наследуются от `Throwable`. Его две основные ветви — `Error` и `Exception`.

| Ветвь | Проверка компилятора | Примеры |
|---|---|---|
| `Error` | Unchecked | `OutOfMemoryError`, `StackOverflowError`, `LinkageError` |
| `RuntimeException` → `Exception` | Unchecked | `IllegalArgumentException` → `NumberFormatException`, `IndexOutOfBoundsException` → `ArrayIndexOutOfBoundsException`, `ArithmeticException` |
| Остальные обычные подтипы `Exception` | Checked | `IOException` и его подтипы `FileNotFoundException`, `SocketException`, а также отдельная ветвь `SQLException` |

## Checked vs Unchecked

| | Checked | Unchecked (RuntimeException) |
|---|---|---|
| Обязателен try-catch / throws | ✅ Да | ❌ Нет |
| Типичные случаи | Ошибка чтения файла, прерывание ожидания | Неверный аргумент, нарушение состояния, исключение прикладного фреймворка |
| Примеры | IOException, SQLException | NPE, IllegalArgument |



## Multicatch

```java
// До Java 7
try {
    // код
} catch (IOException e) {
    handle(e);
} catch (SQLException e) {
    handle(e);
}

// Java 7+: multicatch
try {
    // код
} catch (IOException | SQLException e) {
    handle(e); // e — неявно final
}
```

**Ограничение multicatch:** нельзя поймать связанные типы:
```java
// ОШИБКА: FileNotFoundException — подтип IOException
catch (IOException | FileNotFoundException e) {} // compile error
```

---

## Nested Try Blocks

```java
try {
    try {
        throw new RuntimeException("inner");
    } catch (RuntimeException e) {
        System.out.println("Inner catch: " + e.getMessage());
        throw new RuntimeException("rethrown"); // внешний try поймает
    }
} catch (RuntimeException e) {
    System.out.println("Outer catch: " + e.getMessage()); // "rethrown"
}
```

---

## Exception Propagation

Если подходящего catch нет, исключение поднимается по стеку вызовов. Так работает и checked исключение, если промежуточные методы объявили его через `throws`.

```
methodC() → throws NPE
methodB() → не перехватывает → propagates
methodA() → не перехватывает → propagates
main()    → JVM вызывает UncaughtExceptionHandler
```

```java
void methodA() {
    methodB(); // если methodB бросит — летит сюда
}

void methodB() {
    methodC(); // если methodC бросит — летит сюда
}

void methodC() {
    throw new RuntimeException("error");
}
```

---

## Подавление исключений (Suppressed Exceptions)

Если тело `try` и затем обычный `finally` выбросят исключения, вызывающий получит исключение из `finally`. Первоначальное не попадёт автоматически в `getSuppressed()`.

```java
try {
    throw new RuntimeException("original");
} finally {
    throw new RuntimeException("finally"); // это исключение заменяет original
    // до вызывающего доходит только "finally"
}
```

### Как try-with-resources сохраняет обе ошибки
```java
try (Resource r = new Resource()) {
    throw new RuntimeException("original");
    // r.close() тоже бросает исключение
}
// До вызывающего доходит "original"
// Исключение из close() — подавлено, но доступно через getSuppressed()
```

```java
Throwable[] suppressed = e.getSuppressed();
```

---

## UncaughtExceptionHandler

Когда исключение не перехвачено ни одним catch — вызывается `UncaughtExceptionHandler`.

**Почему выводится последнее исключение:**
Поток использует свой UncaughtExceptionHandler; ThreadGroup может делегировать обработку default handler. Обработчик получает исключение, дошедшее до границы потока. Если finally бросил новое исключение, первоначальное теряется до вызова handler — см. раздел о подавлении исключений выше.

```java
Thread.setDefaultUncaughtExceptionHandler((thread, throwable) -> {
    System.err.println("Uncaught in " + thread.getName() + ": " + throwable);
});
```

---

## Исключения в потоках

Раскрутка стека заканчивается в том потоке, где возникло исключение. Для `Thread` установи handler до `start()`. Для `submit()` проверь результат задачи, потому что стандартный пул сохраняет исключение в `Future`. [Пример Thread и обработчика](../Concurrency/00_Overview.md#отслеживание-исключений-в-потоках) · [Ожидание Future](../Concurrency/Executors-CompletableFuture.md#future-и-блокирующее-ожидание)

## Полезные паттерны

```java
// Rethrow с сохранением типа (Java 7+)
catch (Exception e) {
    throw e; // компилятор понимает реальный тип
}

// Обёртка в unchecked
catch (SQLException e) {
    throw new RuntimeException("DB error", e); // cause сохранён
}

// Проверить тип и перебросить
catch (Exception e) {
    if (e instanceof IOException) throw (IOException) e;
    throw new RuntimeException(e);
}
```

## Как сохранить диагностику на границе слоя

Если JDBC бросил `SQLException`, инфраструктурный слой может перевести его в своё исключение и сохранить причину: `throw new StorageException("Cannot load order " + id, sqlError)`. `getCause()` показывает, какая ошибка привела к новой. `getSuppressed()` показывает дополнительные ошибки в ходе работы с ресурсами, например сбой `close()`. По одной строке сообщения эти случаи не различить. Обрабатывай ожидаемый конфликт отдельно, чтобы он не выглядел для клиента как общая ошибка сервера.

## Two-Minute Drill

- Checked исключение компилятор заставляет обработать или объявить; RuntimeException и Error — unchecked.
- `finally` может скрыть первое исключение; try-with-resources сохраняет исключение закрытия как suppressed.
- Ошибка в дочернем потоке не падает автоматически в вызвавший поток.

**Проверь себя.** При ошибке записи и ошибке close какую увидит вызывающий в try-with-resources?

Исключение записи останется основным, ошибка close будет в getSuppressed().
