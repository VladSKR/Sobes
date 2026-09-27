# CAS, Atomic* и ConcurrentHashMap

**CAS = Compare-And-Set / Compare-And-Swap:** сравнить текущее значение с ожидаемым и атомарно заменить, только если совпало. `compareAndSet(expected, update)` возвращает `false` при конфликте; типичный алгоритм повторяет попытку с новым значением.

```java
AtomicInteger count = new AtomicInteger();
count.incrementAndGet(); // атомарный инкремент
```

`volatile` видит изменения, но `count++` теряет обновления. `AtomicInteger` обеспечивает атомарность своей операции. При высокой конкуренции повторы CAS могут быть дорогими; `LongAdder` полезен для счётчиков, когда нужен быстрый инкремент и допустимо чтение суммы без атомарного общего snapshot.

`ConcurrentHashMap` допускает одновременную работу с разными ключами. В реализации JDK 8+ используются CAS для части случаев (например, пустой бакет) и синхронизация при некоторых изменениях занятого бакета. Фраза «там вообще нет `synchronized`» неверна; также это **не** делает атомарной произвольную последовательность вызовов.

```java
// Два потока могут одновременно пройти проверку.
if (!map.containsKey(key)) map.put(key, value);

// Одна атомарная операция для данного ключа.
map.putIfAbsent(key, value);
map.compute(key, (k, old) -> old == null ? 1 : old + 1);
```

При выборе операции подумай о семантике: `putIfAbsent` вставляет готовое значение; `computeIfAbsent` вычисляет при отсутствии (функция должна быть короткой и без опасных побочных эффектов); `merge` удобен для накопления. `HashMap` обычный и не предназначен для конкурентных записей; его разбор — [[Java Core/Collections/HashMap]].

**Вопросы:** если CAS вернул false, обновление произошло? Нет. Если `containsKey` и `put` каждый потокобезопасен, безопасна ли их композиция? Нет.

Источник: [ConcurrentHashMap JDK 21](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/ConcurrentHashMap.html).
