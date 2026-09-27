# Аннотации Java

## За две минуты

- Аннотация — метаданные на классе, методе или поле; сама по себе она не запускает поведение.
- `@Target` ограничивает место применения, `@Retention` — как долго метаданные сохраняются.
- Код, который читает аннотацию (компилятор, процессор, Spring), придаёт ей смысл.

## Пример

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Audited { }

class Service {
    @Audited
    void transfer() { }
}
```

`RUNTIME` позволяет искать `@Audited` у метода через reflection во время работы программы. `SOURCE` остаётся только в исходном коде и не попадает в class-файл; `CLASS` попадает в class-файл, но не обязана быть доступной обычному reflection API во время выполнения. Это не значит, что `@Audited` уже ведёт audit: нужен код, который обнаружит её и выполнит действие. [[Spring/Core/Reflection]].
