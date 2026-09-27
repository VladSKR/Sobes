# Spring Boot

## За две минуты

- Spring Boot добавляет автоконфигурацию, starters и удобный запуск приложения вокруг Spring Framework.
- Стартер — набор зависимостей; автоконфигурация подключает бины по условиям и может уступить пользовательскому bean.

---

## Преимущества Spring Boot

1. **Dependency Management** — согласованные версии зависимостей через BOM
2. **Автоконфигурация** — `@EnableAutoConfiguration` настраивает компоненты по classpath
3. **Встроенный сервер** — Tomcat/Jetty/Undertow, не нужен внешний WAR деплой
4. **Actuator** — готовые `/health`, `/metrics`, `/info` endpoints
5. **Нет XML-конфигурации**
6. **Стартеры** — `spring-boot-starter-*` собирают нужные зависимости

---

---

## Свой стартер

```
my-starter/
├── src/main/java/
│   └── com/example/
│       ├── MyProperties.java       (@ConfigurationProperties)
│       └── MyAutoConfiguration.java (@Configuration + @ConditionalOnClass...)
└── src/main/resources/
    └── META-INF/spring/
        └── org.springframework.boot.autoconfigure.AutoConfiguration.imports
```

```
# AutoConfiguration.imports
com.example.MyAutoConfiguration
```

---
