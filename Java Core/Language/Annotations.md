# Аннотации Java

## За две минуты

- Retention задаёт, доступна ли аннотация в исходнике, байткоде или через reflection во время выполнения.
- `@Target` ограничивает места применения; для чтения аннотации рефлексией нужна `RUNTIME`.

---

## Аннотации — Retention Policy

```java
@Retention(RetentionPolicy.SOURCE)   // удаляется компилятором (@Override)
@Retention(RetentionPolicy.CLASS)    // в байткоде, недоступна через рефлексию (default)
@Retention(RetentionPolicy.RUNTIME)  // доступна через рефлексию (@Transactional, @Autowired)
```

---
