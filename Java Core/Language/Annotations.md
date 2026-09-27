# Аннотации Java

## Аннотации — Retention Policy

```java
@Retention(RetentionPolicy.SOURCE)   // удаляется компилятором (@Override)
@Retention(RetentionPolicy.CLASS)    // в байткоде, недоступна через рефлексию (default)
@Retention(RetentionPolicy.RUNTIME)  // доступна через рефлексию (@Transactional, @Autowired)
```

---
