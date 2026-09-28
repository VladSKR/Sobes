# Lock, Condition, wait/notify и interrupt

## За две минуты

- `ReentrantLock` даёт взаимное исключение и видимость изменений, как `synchronized`, но позволяет попробовать захват без ожидания, задать таймаут, прервать ожидание и создать несколько `Condition`.
- После **успешного** захвата освобождай lock в `finally`. Если `lockInterruptibly()` прервался или `tryLock()` вернул `false`, освобождать нечего.
- `Condition.await()` освобождает связанный lock. После сигнала поток сначала снова захватывает lock, а затем проверяет условие в `while`. Сам сигнал не делает условие истинным.
- `new ReentrantLock(true)` старается выдавать lock в порядке ожидания, но `tryLock()` без таймаута может обойти очередь.

---

## ReentrantLock: захват и освобождение

```java
ReentrantLock lock = new ReentrantLock();

lock.lock();
try {
    updateSharedState();
} finally {
    lock.unlock();
}
```

`lock()` захватывает lock даже при повторном вызове тем же потоком: он **reentrant**. Каждый успешный захват увеличивает счётчик владения, каждый `unlock()` уменьшает; другие потоки смогут войти, когда счётчик станет нулём. `unlock()` из потока, который не владеет lock, бросит `IllegalMonitorStateException`.

Как и монитор `synchronized`, `ReentrantLock` обеспечивает видимость изменений между освобождением lock одним потоком и последующим захватом другим. Но выход из `synchronized` освобождает монитор автоматически, а явный lock требует `finally`: без него исключение оставит lock захваченным.

### Способы захвата

| Вызов | Если lock занят | Прерывание ожидающего потока |
|---|---|---|
| `lock()` | Ждёт | Не прекращает ожидание из-за interrupt |
| `lockInterruptibly()` | Ждёт | Бросает `InterruptedException` до захвата |
| `tryLock()` | Сразу возвращает `false` | Не ждёт |
| `tryLock(timeout, unit)` | Ждёт не дольше срока, затем `false` | Бросает `InterruptedException` |

```java
try {
    lock.lockInterruptibly();
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
    return; // lock не захвачен
}
try {
    updateSharedState();
} finally {
    lock.unlock();
}
```

После `InterruptedException` при ожидании захвата не вызывай `unlock()`. Если метод может пробросить исключение вызывающему коду, вместо перехвата объяви `throws InterruptedException`. При перехвате, когда пробросить нельзя, обычно восстанавливают флаг и завершают работу.

```java
if (lock.tryLock()) {
    try {
        updateSharedState();
    } finally {
        lock.unlock();
    }
} else {
    doSomethingElse();
}
```

Вариант с таймаутом обрабатывается так же: `if (lock.tryLock(200, TimeUnit.MILLISECONDS)) { try { ... } finally { lock.unlock(); } }`; вызывающий метод объявляет `throws InterruptedException` или обрабатывает прерывание. `finally` нужен только в ветке успешного захвата.

### Fair lock

`new ReentrantLock(true)` включает справедливый режим: при конкуренции lock обычно получает поток, ожидавший дольше. Это порядок предоставления **lock**, а не обещание, что поток раньше выполнит код: планировщик может задержать уже разбуженный поток. Справедливость может снизить пропускную способность.

Важное исключение: `tryLock()` **без таймаута** захватывает свободный lock сразу, даже если другие потоки стоят в очереди fair lock. `tryLock(0, TimeUnit.SECONDS)` учитывает справедливость. Для простого критического участка обычно достаточно `synchronized`; явный lock выбирают, когда нужна одна из перечисленных возможностей.

## Condition: отдельная очередь ожидания

`Condition` создаётся от конкретного lock: `Condition notEmpty = lock.newCondition();`. У одного lock может быть несколько условий, например `notEmpty` и `notFull`: ожидающие этих событий находятся в разных очередях. `signal()` выбирает одного ожидающего на **этой** Condition, `signalAll()` сигнализирует всем ожидающим на ней. Оба метода вызываются при владении связанным lock, иначе будет `IllegalMonitorStateException`.

```java
private final ReentrantLock lock = new ReentrantLock();
private final Condition notEmpty = lock.newCondition();
private final Queue<String> queue = new ArrayDeque<>();

public String take() throws InterruptedException {
    lock.lockInterruptibly();
    try {
        while (queue.isEmpty()) {
            notEmpty.await();
        }
        return queue.remove();
    } finally {
        lock.unlock();
    }
}

public void put(String item) {
    lock.lock();
    try {
        queue.add(item);
        notEmpty.signal();
    } finally {
        lock.unlock();
    }
}
```

Последовательность для `take()`:

1. Пока очередь пуста, `await()` **полностью освобождает** связанный lock и переводит поток в ожидание на `notEmpty`.
2. `put()` меняет состояние под тем же lock и вызывает `signal()`. Это переводит ожидающий поток к борьбе за lock; `put()` ещё владеет им до `unlock()`.
3. Ожидающий поток **повторно захватывает** lock, восстанавливая число прежних захватов, затем `await()` возвращает управление. Цикл `while` проверяет `queue.isEmpty()` заново.

Проверка в `while` нужна, потому что возможно ложное пробуждение; кроме того, другой поток может забрать элемент до того, как разбуженный поток получит lock. Даже после `signal()` ожидающий может вместо обычного возврата получить `InterruptedException` после повторного захвата; `finally` всё равно освободит lock. Fair lock влияет на очередь повторного **захвата lock**, а не превращает `signal()` в немедленную передачу управления или гарантию истинности условия.

## Связь с монитором и interrupt

`Object.wait()`/`notify()` требуют владеть **тем же** монитором; `wait()` освобождает его на время ожидания, `Thread.sleep()` не освобождает. `notify()` тоже не передаёт монитор немедленно. `Condition` аналогично привязан к своему `Lock`, но даёт несколько независимых очередей ожидания.

`Thread.interrupted()` возвращает и **сбрасывает** флаг текущего потока, `isInterrupted()` читает его без сброса. `InterruptedException` обычно очищает флаг; при невозможности пробросить исключение восстанови его через `Thread.currentThread().interrupt()` и заверши работу, если этого требует логика остановки.

## Проверь себя

- Почему `unlock()` стоит после успешного захвата в `finally`, а не безусловно в `finally` вокруг `lockInterruptibly()`?
- Что делает поток после `signal()`, прежде чем продолжить код после `await()`? Почему проверка условия остаётся в `while`?
- Как `tryLock()` без таймаута ведёт себя при `new ReentrantLock(true)`?
- Когда ты выберешь `synchronized`, а когда `ReentrantLock`?

Источники: [ReentrantLock](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/locks/ReentrantLock.html), [Condition](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/locks/Condition.html), [Lock](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/locks/Lock.html).
