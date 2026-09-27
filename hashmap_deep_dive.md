# HashMap — полный разбор от и до

---

## Структура в памяти

HashMap внутри — это **массив бакетов** (`Node[] table`). Каждый бакет — linked list из Node объектов.

```
Node[] table (capacity = 16 по умолчанию):

table[0]  → null
table[1]  → Node("cat", 1) → null
table[2]  → null
table[3]  → Node("dog", 2) → Node("god", 3) → null  ← коллизия, linked list
table[4]  → null
...
table[15] → null
```

Каждый Node:
```java
static class Node<K,V> {
    final int hash;    // закэшированный хэш ключа
    final K key;
    V value;
    Node<K,V> next;    // следующий элемент в бакете (linked list)
}
```

---

## Как вычисляется индекс бакета

### Шаг 1 — перемешивание битов хэша

```java
static final int hash(Object key) {
    int h = key.hashCode();
    return h ^ (h >>> 16);  // XOR старших 16 бит с младшими 16 битами
}
```

Зачем: плохие реализации `hashCode()` часто различаются только в старших битах.
Без перемешивания — все элементы падали бы в один бакет.

```
hashCode = 0xABCD1234
h >>> 16  = 0x0000ABCD
XOR       = 0xABCD1234 ^ 0x0000ABCD = 0xABCD_BE_F9  ← биты перемешаны
```

### Шаг 2 — индекс через побитовое AND

```java
index = hash & (capacity - 1)
```

**Почему `&` а не `%`:**

`%` — деление, дорогая операция CPU. `&` — одна инструкция, в разы быстрее.

`&` даёт тот же результат что `%` **только если capacity — степень двойки**:

```
capacity     = 16  = 0001 0000
capacity - 1 = 15  = 0000 1111  ← маска из единиц

hash = 25 = 0001 1001

25 & 15:
  0001 1001
& 0000 1111
-----------
  0000 1001 = 9  ← индекс бакета

25 % 16 = 9  ← тот же результат, но медленнее
```

Если capacity = 10 (не степень двойки):
```
capacity - 1 = 9 = 0000 1001  ← не сплошные единицы, маска сломана
& с этой маской ≠ остаток от деления на 10
```

**Поэтому capacity всегда степень двойки: 16, 32, 64, 128...**

---

## Операция put — пошагово

```java
map.put("dog", 42);
```

1. Вычисляем `hash("dog")` через перемешивание
2. `index = hash & (capacity - 1)` → находим бакет
3. **Бакет пуст** → создаём Node, кладём в массив
4. **Бакет занят** → идём по linked list:
   - Для каждого Node проверяем: `hash == node.hash && key.equals(node.key)`
   - Нашли совпадение → обновляем value
   - Дошли до конца → добавляем новый Node в хвост
5. Если в бакете стало **>= 8 элементов** → linked list → красно-чёрное дерево
6. Если общий размер map >= threshold → **rehashing**

---

## LoadFactor и Rehashing

### LoadFactor

```java
HashMap<K,V> map = new HashMap<>();
// capacity   = 16  (начальный размер массива)
// loadFactor = 0.75 (по умолчанию)
// threshold  = capacity * loadFactor = 16 * 0.75 = 12
```

Когда добавляется **12-й элемент** (threshold достигнут) — **rehashing**:

```
1. Создаётся новый массив capacity * 2 = 32
2. Все существующие элементы перераспределяются по новым бакетам
3. threshold = 32 * 0.75 = 24
```

### Почему loadFactor = 0.75

Компромисс между памятью и производительностью:
- Высокий loadFactor (0.9) → меньше памяти, больше коллизий, медленнее
- Низкий loadFactor (0.5) → больше памяти, меньше коллизий, быстрее
- 0.75 — математически оптимальное значение для равномерного распределения

### Оптимизация — задать capacity заранее

```java
// Плохо — много rehashing при заполнении
Map<String, String> map = new HashMap<>();
// добавляем 1000 элементов → rehashing при 12, 24, 48, 96, 192, 384, 768

// Хорошо — один раз выделить нужный размер
// Формула: capacity = expectedSize / loadFactor + 1
int expectedSize = 1000;
Map<String, String> map = new HashMap<>(expectedSize * 2); // с запасом
// или точно:
Map<String, String> map = new HashMap<>((int)(expectedSize / 0.75 + 1));
```

---

## Коллизии и TREEIFY_THRESHOLD

**Коллизия** — два разных ключа попали в один бакет (одинаковый индекс, но разные ключи).

```java
// Пример намеренно плохого hashCode
class BadKey {
    @Override
    public int hashCode() { return 42; } // все объекты в один бакет!
}
// Все элементы в table[42 & 15] = table[10]
// HashMap превращается в linked list → O(n) для всех операций
```

### Эволюция бакета

```
0-7 элементов:   linked list    O(n) поиск в худшем случае
8+ элементов:    красно-чёрное дерево  O(log n) поиск
```

Константы:
```java
static final int TREEIFY_THRESHOLD = 8;   // list → tree
static final int UNTREEIFY_THRESHOLD = 6; // tree → list (при удалении)
static final int MIN_TREEIFY_CAPACITY = 64; // минимальный capacity для treeify
                                             // иначе — rehashing вместо tree
```

**Важно:** treeify происходит только если `capacity >= 64`. Если capacity меньше — HashMap делает rehashing вместо превращения в дерево, потому что rehashing эффективнее при маленьком размере.

---

## Java 8 — CAS вместо synchronized в бакете

В Java 8 вставка в пустой бакет использует **CAS** (Compare And Swap) без блокировки.
`synchronized` применяется только при коллизии (вставка в непустой бакет).

Это важно для `ConcurrentHashMap` — там та же оптимизация.

---

## get — пошагово

```java
map.get("dog");
```

1. `hash("dog")` → индекс бакета
2. Идём по linked list / дереву в бакете
3. Для каждого Node: `hash == node.hash && (key == node.key || key.equals(node.key))`
4. Нашли → возвращаем value
5. Дошли до конца → null

**Почему сначала сравниваем hash, потом equals:**
`hash` — int, сравнение за O(1). `equals` — может быть дорогим (строки, сложные объекты). Сначала дешёвая проверка, потом дорогая.

---

## null ключи и значения

```java
map.put(null, "value");  // null ключ разрешён, всегда в бакете 0
map.put("key", null);    // null значение разрешён
map.get("missing");      // возвращает null
map.get("key");          // тоже null если value = null
```

**Проблема:**
```java
// Нельзя отличить "ключ не найден" от "ключ есть, значение null"
String val = map.get("key"); // null — но почему?

// Решение:
if (map.containsKey("key")) {
    String v = map.get("key"); // точно есть ключ
}
// или:
map.getOrDefault("key", "default");
```

---

## equals и hashCode — контракт

Если переопределяешь `equals` — **обязан** переопределить `hashCode`.

**Контракт:**
- Если `a.equals(b)` → `a.hashCode() == b.hashCode()` (обязательно)
- Если `a.hashCode() == b.hashCode()` → `a.equals(b)` может быть false (коллизия — нормально)

**Что сломается без hashCode:**
```java
class User {
    String name;

    @Override
    public boolean equals(Object o) {
        return ((User) o).name.equals(this.name);
    }
    // hashCode НЕ переопределён → использует Object.hashCode() = адрес в памяти
}

User u1 = new User("Ivan"); // hashCode = 12345 (адрес)
User u2 = new User("Ivan"); // hashCode = 67890 (другой адрес)

map.put(u1, "admin");
map.get(u2); // null!

// Почему:
// put(u1): hash(12345) → бакет 5
// get(u2): hash(67890) → бакет 17 → пусто → null
// До equals вообще не доходит
```

**Правильная реализация:**
```java
class User {
    String name;
    int age;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof User)) return false;
        User user = (User) o;
        return age == user.age && Objects.equals(name, user.name);
    }

    @Override
    public int hashCode() {
        return Objects.hash(name, age); // использует те же поля что equals
    }
}
```

**Правило:** hashCode должен использовать **те же поля** что и equals.

---

## Изменение ключа после вставки

```java
List<String> key = new ArrayList<>();
key.add("a");
map.put(key, "value");

key.add("b"); // изменили ключ!
map.get(key); // null — hashCode изменился, бакет другой

// Правило: используй только иммутабельные объекты как ключи
// String, Integer, Long — идеальные ключи
// List, массивы — плохие ключи
```

---

## HashMap vs LinkedHashMap vs TreeMap

```java
// HashMap — нет гарантий порядка
Map<String, Integer> hashMap = new HashMap<>();
hashMap.put("b", 2); hashMap.put("a", 1); hashMap.put("c", 3);
// итерация: случайный порядок

// LinkedHashMap — порядок вставки
Map<String, Integer> linkedMap = new LinkedHashMap<>();
linkedMap.put("b", 2); linkedMap.put("a", 1); linkedMap.put("c", 3);
// итерация: b, a, c (порядок вставки)

// TreeMap — сортировка по ключу (красно-чёрное дерево)
Map<String, Integer> treeMap = new TreeMap<>();
treeMap.put("b", 2); treeMap.put("a", 1); treeMap.put("c", 3);
// итерация: a, b, c (алфавитный порядок)
// get/put: O(log n) вместо O(1)
```

---

## HashMap vs Hashtable vs ConcurrentHashMap

| | HashMap | Hashtable | ConcurrentHashMap |
|---|---|---|---|
| Потокобезопасность | ❌ | ✅ | ✅ |
| Механизм блокировки | — | synchronized на весь объект | synchronized на бакет |
| null ключи | ✅ | ❌ | ❌ |
| Производительность | Лучшая | Хуже всех | Хорошая под нагрузкой |
| Использовать | Однопоточно | Никогда | Многопоточно |

**Hashtable устарел** — не используй. Вместо него `ConcurrentHashMap`.

### ConcurrentHashMap — атомарные операции

```java
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();

// НЕ атомарно — race condition
if (!map.containsKey(key)) {
    map.put(key, 1);
}

// Атомарно
map.putIfAbsent(key, 1);
map.computeIfAbsent(key, k -> expensiveCompute(k));
map.compute(key, (k, v) -> v == null ? 1 : v + 1);
map.merge(key, 1, Integer::sum); // удобно для счётчиков
```

---

## Сложность операций

| Операция | Лучший случай | Худший случай | После Java 8 (с деревом) |
|---|---|---|---|
| get | O(1) | O(n) все в один бакет | O(log n) |
| put | O(1) | O(n) | O(log n) |
| remove | O(1) | O(n) | O(log n) |
| containsKey | O(1) | O(n) | O(log n) |

Амортизированная сложность при равномерном распределении — **O(1)**.

---

## Частые ошибки

**1. Сравнение через == вместо equals:**
```java
map.get(new Integer(5)) // до Java 9 создаёт новый объект
// Правильно: map.get(5) — автобоксинг использует кэш
```

**2. Модификация во время итерации:**
```java
for (String key : map.keySet()) {
    map.remove(key); // ConcurrentModificationException!
}

// Правильно:
map.entrySet().removeIf(e -> shouldRemove(e.getKey()));
// или через iterator:
Iterator<Map.Entry<String,Integer>> it = map.entrySet().iterator();
while (it.hasNext()) {
    if (shouldRemove(it.next().getKey())) it.remove();
}
```

**3. Не задать начальный capacity:**
```java
// Если знаешь размер — задай сразу
Map<String, String> map = new HashMap<>(expectedSize * 2);
```

---

## Итоговая схема

```
put("key", value):
    ↓
hash("key") = hashCode ^ (hashCode >>> 16)
    ↓
index = hash & (capacity - 1)
    ↓
table[index] == null?
    ├── да → Node("key", value) в бакет
    └── нет → идём по linked list / дереву
              ├── найден ключ → обновляем value
              └── не найден → добавляем Node в конец
                              ↓
                         size >= TREEIFY_THRESHOLD (8)?
                              ├── да + capacity >= 64 → treeify
                              └── да + capacity < 64  → rehashing
                         
size >= threshold (capacity * 0.75)?
    └── да → rehashing (capacity * 2)
```
