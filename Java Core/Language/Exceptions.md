# Exceptions (Исключения)

## Two-Minute Drill

- Checked исключение компилятор заставляет обработать или объявить; RuntimeException и Error — unchecked.
- `finally` может скрыть первое исключение; try-with-resources сохраняет исключение закрытия как suppressed.
- Ошибка в дочернем потоке не падает автоматически в вызвавший поток.

---

#java #exceptions #core

---

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

Если в дочернем потоке брошено исключение — **main поток не упадёт**:

```java
Thread t = new Thread(() -> {
    throw new RuntimeException("в потоке"); // main не знает
});
t.start();

// Чтобы поймать:
t.setUncaughtExceptionHandler((thread, ex) -> {
    System.out.println("Поймали: " + ex.getMessage());
});
```

Через Future/CompletableFuture исключение оборачивается и бросается при `get()`:
```java
Future<?> f = executor.submit(() -> { throw new RuntimeException(); });
try {
    f.get(); // бросит ExecutionException
} catch (ExecutionException e) {
    Throwable cause = e.getCause(); // оригинальное исключение
}
```

---

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
