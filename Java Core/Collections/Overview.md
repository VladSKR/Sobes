# Коллекции

## Two-Minute Drill

- `Collection` объединяет List/Set/Queue, а Map — отдельная ветка интерфейсов.
- HashMap выбирают для доступа по ключу, ArrayList — для большинства списков, Deque — для очереди.
- При коллизии сравнивают ключи, а не значения.

---

#java #collections #core

---

## Иерархия коллекций

Через `Iterable<T>` объект предоставляет `iterator()`, которым пользуется цикл `for-each`. `Collection<T>` расширяет `Iterable<T>`. `Map<K,V>` имеет отдельный контракт, а обходят его через представления `keySet()`, `values()` и `entrySet()`.

| Интерфейс | Что определяет | Примеры реализаций |
|---|---|---|
| `List` | Упорядоченная последовательность, доступ по индексу | `ArrayList`, `LinkedList` |
| `Set` / `SortedSet` | Уникальные элементы, у sorted-ветви элементам задан порядок | `HashSet`, `LinkedHashSet`, `TreeSet`, `EnumSet` |
| `Queue` | Как получить следующий элемент | `PriorityQueue`, где извлекается наименьший элемент по порядку сравнения |
| `Deque` → `Queue` | Доступ с обоих концов | `ArrayDeque`, `LinkedList` |
| `Map` / `SortedMap` | По ключу хранится значение, у sorted-ветви задан порядок ключей | `HashMap`, `LinkedHashMap`, `TreeMap`, `EnumMap`, `IdentityHashMap` |

## List

### ArrayList vs LinkedList

| | ArrayList | LinkedList |
|---|---|---|
| Структура | Resizable array | Doubly-linked list |
| get(i) | O(1) | O(n) |
| add/remove середина по индексу | O(n) из-за сдвига | O(n) из-за поиска узла; само связывание O(1) |
| Память | Меньше | Больше (узлы) |

**ArrayList** — внутри массив + счётчик size. При добавлении увеличивается, при удалении уменьшается.

**LinkedList** — двусторонняя очередь взаимосвязанных элементов. Каждый элемент указывает на предыдущий и следующий.

- **Queue** задаёт дисциплину очереди. FIFO распространена, у `PriorityQueue` другой порядок.
- **Deque** можно применять как FIFO или как стек LIFO.

---

## Map

### HashMap
- Не гарантирует порядок итерации
- Добавление/удаление может изменить порядок

### TreeMap
- Итерация по натуральному порядку ключей (или по Comparator)

### LinkedHashMap
- Поддерживает doubly-linked list через все записи
- По умолчанию итерация идёт в порядке вставки. Можно создать вариант с порядком доступа

### Hashtable
- Synchronized, в отличие от HashMap

### IdentityHashMap
- Сравнивает ключи через `==` (ссылки), а не `equals()`
- Нарушает контракт Map намеренно

### EnumMap
- Специализированная реализация для enum-ключей
- Все ключи должны принадлежать одному типу enum
- Очень эффективна по памяти и скорости

---

## Set

### HashSet
- Нет дубликатов
- Порядок обхода не гарантирован
- Основан на HashMap (ключи = элементы, значения = заглушка)

### LinkedHashSet и TreeSet

`LinkedHashSet` поддерживает порядок добавления и хранит дополнительные связи между элементами. `TreeSet` хранит элементы в порядке сравнения. Если comparator вернул 0, дерево считает элементы дубликатами. Например, comparator по длине строк сочтёт `"aa"` и `"bb"` одинаковыми для целей набора.

### EnumSet
- Специализированный Set для enum
- Внутри — битовый вектор (очень быстро и компактно)

---

## Как работает HashMap

`HashMap` хранит записи по бакетам. Чтобы найти ключ, мапа сначала вычисляет индекс бакета из хеша, затем проверяет совпавшие записи по хешу и равенству ключа. Подробный путь `put` и `get`, load factor, resize и превращение бакета в дерево показаны в [главе HashMap](HashMap.md). Внутри `HashSet` элементы служат ключами такой мапы, а значения заменены общей заглушкой.

## Iterable и Iterator

```java
public interface Iterable<T> {
    Iterator<T> iterator();
}

public interface Iterator<T> {
    boolean hasNext();
    T next();
    void remove(); // optional
}
```

### Удаление элементов при итерации

При структурном изменении обычного `ArrayList` во время for-each итератор может выбросить `ConcurrentModificationException`. Fail-fast помогает обнаруживать ошибку, но не гарантирует её обнаружение.

Чтобы удалить текущий элемент, используй `Iterator.remove()`, если итератор поддерживает удаление.
```java
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    String s = it.next();
    if (s.equals("delete")) {
        it.remove(); // безопасно
    }
}
```

Для удаления по условию подходит `removeIf()`, если коллекция изменяема.
```java
list.removeIf(s -> s.equals("delete"));
```

---

## Concurrent коллекции

| Коллекция               | Аналог     | Особенность                                                        |
| ----------------------- | ---------- | ------------------------------------------------------------------ |
| `ConcurrentHashMap`     | HashMap    | В OpenJDK 21 применяет CAS и синхронизацию по бакету для отдельных путей записи |
| `CopyOnWriteArrayList`  | ArrayList  | Копирует массив при каждой записи. Безопасно для чтения            |
| `CopyOnWriteArraySet`   | HashSet    | На основе CopyOnWriteArrayList                                     |
| `ConcurrentLinkedQueue` | LinkedList | Lock-free, алгоритм Майкла-Скотта                                  |
| `BlockingQueue`         | —          | put/take блокируются при полной/пустой очереди                     |
| `ArrayBlockingQueue`    | —          | Ограниченная, блокирующая                                          |
| `LinkedBlockingQueue`   | —          | Опционально ограниченная                                           |
| `PriorityBlockingQueue` | —          | С приоритетом + блокировка                                         |

### Преимущества concurrent-коллекций
- Потокобезопасность без явной синхронизации
- Лучшая производительность, чем `Collections.synchronizedXxx()`
- Не бросают `ConcurrentModificationException` при итерации

---

## PECS

**Producer Extends, Consumer Super**

```java
// Producer (читаем из коллекции) → extends
public void process(List<? extends Number> producer) {
    Number n = producer.get(0); // OK
    // producer.add(1); // ОШИБКА — нельзя писать
}

// Consumer (пишем в коллекцию) → super
public void fill(List<? super Integer> consumer) {
    consumer.add(1); // OK
    // Integer n = consumer.get(0); // нельзя — только Object
}
```

---

## Контракт equals/hashCode

1. `equals()` должен быть: рефлексивным, симметричным, транзитивным, консистентным
2. Если `a.equals(b)` → `a.hashCode() == b.hashCode()`
3. Обратное **не** обязательно (коллизии хэшей допустимы)
4. **Всегда** переопределяй оба метода вместе

```java
// Нарушение: переопределили equals, но не hashCode
// → объект не найти в HashMap/HashSet
```
