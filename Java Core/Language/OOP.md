# ООП

## За две минуты

- Инкапсуляция прячет состояние за контрактом, полиморфизм вызывает реализацию по фактическому типу объекта.
- `equals/hashCode` образуют контракт для hash-коллекций.
- Рефлексия позволяет изучать класс и вызывать члены во время выполнения; это не обязательный способ для обычного кода.

---

#java #oop #core

---

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

```java
public class Object {
    public boolean equals(Object obj)     // по умолчанию: ==
    public int hashCode()                  // по умолчанию: идентичностный хэш, не обязан быть адресом
    public String toString()               // по умолчанию: ClassName@hashCode
    public Class<?> getClass()             // рефлексия — реальный тип
    protected Object clone()               // клонирование
    protected void finalize()              // устарел
    public final void wait()               // мониторы
    public final void notify()
    public final void notifyAll()
}
```

---



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
