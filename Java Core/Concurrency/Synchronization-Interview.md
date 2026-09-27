# Многопоточность: synchronized, ReentrantLock, volatile

## 1. Многопоточность: synchronized, ReentrantLock, volatile

### volatile

Переменная помечается `volatile` чтобы сказать JVM: **не кэшируй это в регистре процессора или L1/L2 кэше потока — всегда читай из основной памяти**.

**Проблема без volatile:**
```java
// Поток 1 пишет
boolean running = true;

// Поток 2 читает — может никогда не увидеть изменение
// потому что JIT закэшировал значение в регистре
while (running) {
    doWork();
}
```

**С volatile:**
```java
volatile boolean running = true;
// Поток 2 всегда читает актуальное значение из RAM
```

**Что volatile гарантирует:**
- Видимость (visibility) — изменение в одном потоке сразу видно другим
- Happens-before — всё что было до записи в volatile, видно после чтения

**Что volatile НЕ гарантирует:**
- Атомарность составных операций

```java
volatile int counter = 0;
counter++; // НЕ атомарно! Это read → increment → write — три операции
// Два потока могут прочитать одновременно counter=5, оба запишут 6 вместо 7
```

**Когда использовать:**
- Простой флаг (boolean) который один поток пишет, другой читает
- Double-checked locking (см. ниже)

---

### synchronized

Обеспечивает **взаимное исключение** (mutual exclusion) — только один поток выполняет блок кода одновременно.

Каждый объект в Java имеет **монитор** (intrinsic lock). `synchronized` захватывает этот монитор.

```java
// На методе — монитор this
public synchronized void increment() {
    counter++;
}

// На блоке — можно выбрать любой объект как монитор
private final Object lock = new Object();

public void increment() {
    synchronized (lock) {
        counter++;
    }
}

// На статическом методе — монитор Class объекта
public static synchronized void staticMethod() {
    // монитор MyClass.class
}
```

**Что гарантирует:**
- Атомарность блока
- Видимость — при выходе из synchronized все записи flush в RAM

**Минусы:**
- Нельзя прервать ожидание (нет таймаута)
- Нельзя попробовать захватить без блокировки (tryLock)
- Нет разделения read/write блокировок — читатели тоже блокируют друг друга

---

### ReentrantLock

Явная блокировка с расширенными возможностями.

```java
private final ReentrantLock lock = new ReentrantLock();

public void increment() {
    lock.lock();
    try {
        counter++;
    } finally {
        lock.unlock(); // ВСЕГДА в finally, иначе deadlock при исключении
    }
}
```

**Дополнительные возможности:**

```java
// Попробовать без ожидания
if (lock.tryLock()) {
    try {
        // захватили
    } finally {
        lock.unlock();
    }
} else {
    // не захватили — делаем что-то другое
}

// С таймаутом
if (lock.tryLock(100, TimeUnit.MILLISECONDS)) {
    // ...
}

// Прерываемое ожидание
lock.lockInterruptibly(); // бросит InterruptedException если поток прерван

// Честная очередь (fair) — потоки получают блокировку в порядке ожидания
ReentrantLock fairLock = new ReentrantLock(true);
```

**ReentrantReadWriteLock — читатели не блокируют друг друга:**

```java
private final ReadWriteLock rwLock = new ReentrantReadWriteLock();

public String read() {
    rwLock.readLock().lock();
    try {
        return data; // много потоков могут читать одновременно
    } finally {
        rwLock.readLock().unlock();
    }
}

public void write(String value) {
    rwLock.writeLock().lock();
    try {
        data = value; // эксклюзивный доступ
    } finally {
        rwLock.writeLock().unlock();
    }
}
```

---

### Сравнение

| | synchronized | ReentrantLock | volatile |
|---|---|---|---|
| Атомарность | ✅ | ✅ | ❌ |
| Видимость | ✅ | ✅ | ✅ |
| Таймаут | ❌ | ✅ | — |
| tryLock | ❌ | ✅ | — |
| Прерывание | ❌ | ✅ | — |
| Read/Write разделение | ❌ | ✅ (RRWL) | — |
| Производительность | Хуже (JVM overhead) | Лучше при конкуренции | Лучший |

**Правило выбора:**
- Простой флаг → `volatile`
- Простая атомарная операция → `synchronized` или `AtomicInteger`
- Нужен таймаут / tryLock / прерывание → `ReentrantLock`
- Много читателей, мало писателей → `ReentrantReadWriteLock`

---

### happens-before

Концепция из Java Memory Model. Если операция A happens-before B — все изменения A видны B.

Устанавливается:
- Выход из `synchronized` → happens-before → вход следующего потока в тот же монитор
- Запись в `volatile` → happens-before → чтение того же volatile
- `Thread.start()` → happens-before → любое действие в запущенном потоке
- Любое действие в потоке → happens-before → `Thread.join()` в другом потоке

---

### Double-checked locking (пример volatile на практике)

```java
public class Singleton {
    // volatile ОБЯЗАТЕЛЕН
    private static volatile Singleton instance;

    public static Singleton getInstance() {
        if (instance == null) {                    // первая проверка без блокировки
            synchronized (Singleton.class) {
                if (instance == null) {            // вторая проверка с блокировкой
                    instance = new Singleton();    // без volatile другой поток может
                }                                  // увидеть частично инициализированный объект
            }
        }
        return instance;
    }
}
```

Без `volatile` — JIT может переупорядочить инструкции создания объекта, и другой поток увидит ненулевой указатель на незавершённый объект.

---
