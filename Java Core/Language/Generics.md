# Generics (Дженерики)

## За две минуты

- Generics дают проверку типов на компиляции; erasure удаляет часть сведений о типе из исполняемого кода.
- `? extends T` удобно читать как producer, `? super T` — записывать как consumer.
- Нельзя создать `new T()` без предоставленного способа создания.

---

#java #generics #core

---

## Основы

Дженерики добавлены в Java 5 для обеспечения типобезопасности на этапе компиляции.

```java
List<String> list = new ArrayList<>();
list.add("hello");
String s = list.get(0); // не нужен каст
```

---

## Type Erasure (Стирание типов)

**Ключевой факт: в рантайме типов дженериков НЕТ.**

При компиляции все параметры типов стираются — это называется **type erasure**.

```java
List<String> strings = new ArrayList<>();
List<Integer> ints = new ArrayList<>();

// В рантайме оба — просто List
strings.getClass() == ints.getClass(); // true
```

### Что происходит при компиляции

```java
// Исходник
List<String> list = new ArrayList<String>();
list.add("hello");
String s = list.get(0);

// После erasure (байткод)
List list = new ArrayList();
list.add("hello");
String s = (String) list.get(0); // компилятор вставляет каст
```

### Следствия type erasure

1. Нельзя создать `new T()` или `new T[]`
2. Нельзя использовать `instanceof List<String>` (только `instanceof List<?>`)
3. Нельзя создать generic массив: `new List<String>[10]` — ошибка
4. Нет перегрузки методов только по generic-типу:

```java
// ОШИБКА КОМПИЛЯЦИИ — после erasure оба метода одинаковые
void process(List<String> list) {}
void process(List<Integer> list) {}
```

---

## Wildcards

### Unbounded: `<?>`
```java
void printList(List<?> list) {
    for (Object o : list) System.out.println(o);
}
```

### Upper bounded: `<? extends T>` — Producer
```java
double sum(List<? extends Number> list) {
    double result = 0;
    for (Number n : list) result += n.doubleValue();
    return result;
}
```

### Lower bounded: `<? super T>` — Consumer
```java
void addNumbers(List<? super Integer> list) {
    list.add(1);
    list.add(2);
}
```

---

## PECS

**Producer Extends, Consumer Super**

| Роль | Wildcard | Можно | Нельзя |
|---|---|---|---|
| Producer | `? extends T` | читать как T | писать |
| Consumer | `? super T` | писать T | читать как T (только Object) |

---

## Bounded Type Parameters

```java
// T должен реализовывать Comparable
public <T extends Comparable<T>> T max(T a, T b) {
    return a.compareTo(b) > 0 ? a : b;
}

// Несколько ограничений
public <T extends Comparable<T> & Serializable> void save(T obj) {}
```

---

## Generic методы

```java
public <T> List<T> repeat(T item, int times) {
    List<T> result = new ArrayList<>();
    for (int i = 0; i < times; i++) result.add(item);
    return result;
}
```

---

## Реификация (Reification)

Тип называется **реифицированным** если его информация полностью доступна в рантайме.

- Реифицированные: `int`, `String`, `List`, `List<?>`
- НЕ реифицированные: `List<String>`, `T`, `List<? extends Number>`

Массивы — **ковариантны** и реифицированы:
```java
Object[] arr = new String[10]; // компилируется
arr[0] = 42; // ArrayStoreException в рантайме
```

Generics — **инвариантны**:
```java
List<Object> list = new ArrayList<String>(); // ОШИБКА КОМПИЛЯЦИИ
```
