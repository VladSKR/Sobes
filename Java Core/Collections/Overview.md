# Коллекции

#java #collections #core

---

## Иерархия коллекций

```
Collection
├── List
│   ├── ArrayList
│   └── LinkedList
├── Set
│   ├── HashSet
│   ├── LinkedHashSet
│   └── TreeSet
└── Queue
    └── Deque

Map
├── HashMap
├── LinkedHashMap
├── TreeMap
├── Hashtable
├── IdentityHashMap
└── EnumMap
```

---

## List

### ArrayList vs LinkedList

| | ArrayList | LinkedList |
|---|---|---|
| Структура | Resizable array | Doubly-linked list |
| get(i) | O(1) | O(n) |
| add/remove середина | O(n) | O(1) |
| Память | Меньше | Больше (узлы) |

**ArrayList** — внутри массив + счётчик size. При добавлении увеличивается, при удалении уменьшается.

**LinkedList** — двусторонняя очередь взаимосвязанных элементов. Каждый элемент указывает на предыдущий и следующий.

- **Queue** — FIFO
- **Deque** — FIFO/LIFO

---

## Map

### HashMap
- Не гарантирует порядок итерации
- Добавление/удаление может изменить порядок

### TreeMap
- Итерация по натуральному порядку ключей (или по Comparator)

### LinkedHashMap
- Поддерживает doubly-linked list через все записи
- Порядок итерации = порядок вставки ключей

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
- Случайный порядок
- Основан на HashMap (ключи = элементы, значения = заглушка)

### EnumSet
- Специализированный Set для enum
- Внутри — битовый вектор (очень быстро и компактно)

---

## Как работает HashMap

### Внутренняя структура
- Массив `Node[] table` (бакеты)
- Каждый бакет — связный список (или дерево при переполнении)

### Алгоритм put(key, value)
1. Вычислить `hash(key)`
2. Определить индекс бакета: `index = hash & (capacity - 1)`
3. Если бакет пуст — вставить
4. Если занят — проверить equals() для каждого элемента в цепочке
5. Совпадение → обновить; нет совпадения → добавить в конец

### LoadFactor и resize
- `loadFactor` по умолчанию = **0.75**
- `threshold = capacity * loadFactor`
- При превышении threshold → resize (удвоение capacity, rehash всех элементов)

### Переполнение бакета (Treeify)
- До Java 8: связный список в бакете
- Java 8+: если в бакете > **8** элементов → преобразование в **красно-чёрное дерево** (TreeNode)
- Обратно в список — если < **6** элементов

### HashSet
- Реализован через HashMap
- Элементы Set = ключи HashMap
- Значение = константа `PRESENT` (заглушка)

---

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

❌ **Нельзя** через обычный for-each — `ConcurrentModificationException`

✅ **Нужно** через `Iterator.remove()`:
```java
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    String s = it.next();
    if (s.equals("delete")) {
        it.remove(); // безопасно
    }
}
```

✅ Или через `removeIf()`:
```java
list.removeIf(s -> s.equals("delete"));
```

---

## Concurrent коллекции

| Коллекция               | Аналог     | Особенность                                                        |
| ----------------------- | ---------- | ------------------------------------------------------------------ |
| `ConcurrentHashMap`     | HashMap    | Сегментированная блокировка (Java 8: CAS + synchronized по бакету) |
| `CopyOnWriteArrayList`  | ArrayList  | Копирует массив при каждой записи. Безопасно для чтения            |
| `CopyOnWriteArraySet`   | HashSet    | На основе CopyOnWriteArrayList                                     |
| `ConcurrentLinkedQueue` | LinkedList | Lock-free, алгоритм Майкла-Скотта                                  |
| `BlockingQueue`         | —          | put/take блокируются при полной/пустой очереди                     |
| `ArrayBlockingQueue`    | —          | Ограниченная, блокирующая                                          |
| `LinkedBlockingQueue`   | —          | Опционально ограниченная                                           |
| `PriorityBlockingQueue` | —          | С приоритетом + блокировка                                         |

### Преимущества Concurrent коллекций
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
