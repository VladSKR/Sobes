# ООП

## Четыре принципа

### 1. Абстракция
Выделение существенных характеристик объекта, игнорирование несущественных.
```java
abstract class Shape {
    abstract double area(); // что делает — понятно, как — скрыто
}
```

### 2. Инкапсуляция
Скрытие внутреннего состояния, доступ только через публичный интерфейс.
```java
class BankAccount {
    private double balance; // скрыто
    public void deposit(double amount) { // контролируемый доступ
        if (amount > 0) balance += amount;
    }
}
```

### 3. Наследование
Создание нового класса на основе существующего.
```java
class Animal { void eat() {} }
class Dog extends Animal { void bark() {} }
```

### 4. Полиморфизм
Один интерфейс — много реализаций.
```java
Shape s = new Circle(5); // compile-time тип Shape, runtime тип Circle
s.area(); // вызов метода Circle
```

---

## Динамическое vs Статическое связывание

### Статическое (Static / Early Binding)
- Разрешается на этапе **компиляции**
- Применяется к: `static` методам, `private` методам, `final` методам, перегруженным методам
```java
class A {
    static void hello() { System.out.println("A"); }
}
class B extends A {
    static void hello() { System.out.println("B"); }
}
A obj = new B();
obj.hello(); // "A" — статическое связывание по типу переменной
```

### Динамическое (Dynamic / Late Binding)
- Разрешается в **рантайме** через vtable
- Применяется к: переопределённым (`@Override`) методам
```java
class A {
    void hello() { System.out.println("A"); }
}
class B extends A {
    @Override
    void hello() { System.out.println("B"); }
}
A obj = new B();
obj.hello(); // "B" — динамическое связывание по реальному типу
```

---

## Композиция, Агрегация, Ассоциация

### Ассоциация
Общая связь между объектами. Оба живут независимо.
```java
class Teacher { }
class Student {
    Teacher teacher; // ассоциация — teacher живёт отдельно
}
```

### Агрегация (слабая принадлежность)
"has-a", но дочерний объект может существовать без родителя.
```java
class Department {
    List<Employee> employees; // агрегация — employee может уйти в другой отдел
}
```

### Композиция (сильная принадлежность)
"owns-a" — дочерний объект не может существовать без родителя.
```java
class House {
    private final Room room = new Room(); // композиция — комната не живёт без дома
}
```

---



## Object — базовые методы

| Метод | Назначение |
|---|---|
| `equals(Object)`, `hashCode()` | Равенство и хеш. В `Object` равенство определяется идентичностью ссылки. Равные объекты обязаны иметь одинаковый хеш |
| `toString()` | Строка для диагностики. Реализация по умолчанию включает имя класса и шестнадцатеричный хеш |
| `getClass()` | Возвращает `Class<?>` фактического типа, например здесь можно получить метаданные рефлексией |
| `clone()` | Защищённый метод поверхностного копирования у объектов, поддерживающих `Cloneable` |
| `wait()`, `wait(long)`, `wait(long, int)` | При владении монитором временно отпускают его и ждут. Перегрузки задают миллисекунды и долю в наносекундах |
| `notify()`, `notifyAll()` | Сигнализируют одному или всем ожидающим на том же мониторе. Получателям ещё нужно заново захватить монитор |
| `finalize()` | Исторический способ финализации, deprecated for removal. Вызов и время финализации не гарантированы |

Контракт `equals` требует рефлексивности, симметрии, транзитивности и стабильности результата, пока участвующее состояние не меняется. Для ненулевого `x` вызов `x.equals(null)` даёт `false`. В hash-коллекции поля ключа, участвующие в `equals/hashCode`, должны сохранять значения, пока ключ хранится в таблице.

[Монитор, wait и notify](../Concurrency/Lock-Condition-Interrupt.md)

## Рефлексия

```java
Class<?> clazz = obj.getClass();       // через объект
Class<?> clazz = String.class;          // через литерал
Class<?> clazz = Class.forName("java.lang.String"); // через имя

// Методы
Method[] methods = clazz.getDeclaredMethods();
Method m = clazz.getDeclaredMethod("methodName", String.class);
m.setAccessible(true);
m.invoke(obj, "arg");

// Поля
Field f = clazz.getDeclaredField("fieldName");
f.setAccessible(true);
f.get(obj);

// Конструкторы
Constructor<?> c = clazz.getDeclaredConstructor();
Object instance = c.newInstance();
```

## От сигнатуры к вызову метода

Пусть у `Printer` есть перегрузки `print(Object)` и `print(String)`, а аргумент объявлен как `Object value = "hello"`. На этапе компиляции выбирается `print(Object)` по объявленным типам. Если сам `printer` ссылается на подкласс, который переопределил именно `print(Object)`, на исполнении выбирается реализация подкласса. Перегрузка выбирает сигнатуру, переопределение — реализацию этой сигнатуры.

## Two-Minute Drill

- Инкапсуляция прячет состояние за контрактом, полиморфизм вызывает реализацию по фактическому типу объекта.
- `equals/hashCode` образуют контракт для hash-коллекций.
- Рефлексия позволяет изучать класс и вызывать члены во время выполнения; это не обязательный способ для обычного кода.

**Проверь себя.** У аргумента объявленный тип Object, а фактический String. Какая перегрузка выбирается?

Перегрузка по Object, затем для этой сигнатуры учитывается переопределение у фактического класса получателя.
