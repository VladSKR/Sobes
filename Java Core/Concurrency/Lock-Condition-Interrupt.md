# Lock, Condition, wait/notify и interrupt

## ReentrantLock: правильный шаблон

```java
Lock lock = new ReentrantLock();
lock.lock();
try {
    updateSharedState();
} finally {
    lock.unlock();
}
```

`unlock()` обязателен в `finally` **после успешного захвата**. Если исключение вылетит из критической секции, `synchronized` сам освободит монитор; `ReentrantLock` без `finally` останется захваченным. Преимущества явного Lock — `tryLock()`, ограниченное ожидание, `lockInterruptibly()` и отдельные `Condition`.

```java
try {
    lock.lockInterruptibly();  // может бросить до входа в try ниже
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
    return;
}
try {
    updateSharedState();
} finally {
    lock.unlock();
}
```

Здесь `InterruptedException` при ожидании означает, что lock **не был захвачен** и вызывать `unlock()` нельзя. Обычный `lock()` не бросает `InterruptedException` во время ожидания; `lockInterruptibly()` бросает. Таймаут: `if (!lock.tryLock(200, TimeUnit.MILLISECONDS)) return;`, далее `try/finally`.

## Condition и монитор

`Condition.await()` временно освобождает связанный lock, а перед возвратом повторно его захватывает. `signal()` будит ожидающего, но тот продолжит работу, только вновь получив lock. Условия всегда проверяют в `while`: пробуждение может быть ложным или условие успеет измениться.

```java
lock.lock();
try {
    while (queue.isEmpty()) {
        notEmpty.await();
    }
    takeItem();
} finally {
    lock.unlock();
}
```

`Object.wait()`/`notify()` требуют владеть **тем же** монитором; `wait()` освобождает его, `sleep()` нет. `notify()` не передаёт монитор немедленно. `Thread.interrupted()` возвращает и **сбрасывает** флаг текущего потока, `isInterrupted()` читает его без сброса. При перехвате `InterruptedException` в инфраструктурном коде обычно заверши работу или восстанови флаг, если не можешь пробросить исключение.

## Где ты путался

Ты верно заметил необходимость `finally`, но сначала считал освобождение при исключении преимуществом Lock: это наоборот дополнительная ответственность. В примере с прерыванием ожидающего T2 исключение возникает в `lockInterruptibly()` до входа в защищённый блок. Следующий контрольный вопрос: какие два условия должны выполниться, чтобы `await()` вернулся и поток мог продолжить работу?

Источники: [Lock/Condition JDK 21](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/locks/package-summary.html).
