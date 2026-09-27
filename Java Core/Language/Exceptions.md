# Exceptions (Исключения)

#java #exceptions #core

---

## Иерархия

```
Throwable
├── Error (не обрабатываем)
│   ├── OutOfMemoryError
│   ├── StackOverflowError
│   └── ...
└── Exception
    ├── RuntimeException (unchecked)
    │   ├── NullPointerException
    │   ├── IllegalArgumentException
    │   ├── ArrayIndexOutOfBoundsException
    │   └── ...
    └── Checked Exceptions
        ├── IOException
        ├── SQLException
        └── ...
```

---

## Checked vs Unchecked

| | Checked | Unchecked (RuntimeException) |
|---|---|---|
| Обязателен try-catch / throws | ✅ Да | ❌ Нет |
| Когда | Внешние ресурсы (IO, DB) | Программные ошибки |
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
    handle(e); // e — effectively final
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

Unchecked исключения автоматически поднимаются по стеку вызовов:

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

Возникает когда в `finally` или try-with-resources тоже бросается исключение:

```java
try {
    throw new RuntimeException("original");
} finally {
    throw new RuntimeException("finally"); // original ПОДАВЛЯЕТСЯ
    // до вызывающего доходит только "finally"
}
```

### Try-with-resources решает проблему
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
JVM ищет handler в порядке: поток → ThreadGroup → default handler.
По умолчанию — выводится stack trace последнего непойманного исключения.

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
