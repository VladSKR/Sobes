# Spring Boot

## Преимущества Spring Boot

1. **Dependency Management** — согласованные версии зависимостей через BOM
2. **Автоконфигурация** — `@EnableAutoConfiguration` настраивает компоненты по classpath
3. **Встроенный сервер** — Tomcat/Jetty/Undertow, не нужен внешний WAR деплой
4. **Actuator** — готовые `/health`, `/metrics`, `/info` endpoints
5. **Java-конфигурация** — типичный способ описания бинов. При необходимости можно импортировать XML
6. **Стартеры** — `spring-boot-starter-*` собирают нужные зависимости

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

## Автоконфигурация от зависимости до бина

При добавлении стартера JDBC в classpath появляются Spring JDBC и пул соединений. Автоконфигурация проверяет подходящие классы, настройки и уже зарегистрированные определения. Если приложение объявило собственный `DataSource`, типичная конфигурация пула отступает. BOM отвечает за версии зависимостей, starter подключает их, auto-configuration добавляет компоненты.

В собственном стартере можно объявить минимальную конфигурацию:

```java
@AutoConfiguration
@EnableConfigurationProperties(ClientProperties.class)
@ConditionalOnClass(ApiClient.class)
class ClientAutoConfiguration {
    @Bean
    @ConditionalOnMissingBean
    ApiClient apiClient(ClientProperties properties) {
        return new ApiClient(properties.baseUrl());
    }
}
```

Зарегистрируй полное имя класса в `AutoConfiguration.imports`. В тесте с `ApplicationContextRunner` проверь два случая: появился стандартный бин и пользовательская реализация оставлена единственной. Для диагностики условия можно увидеть в отчёте при запуске с `--debug` или через Actuator conditions при настроенном доступе.

[Свой стартер и автоконфигурация](https://docs.spring.io/spring-boot/reference/features/developing-auto-configuration.html)

## Two-Minute Drill

- Spring Boot добавляет автоконфигурацию, starters и удобный запуск приложения вокруг Spring Framework.
- Стартер — набор зависимостей; автоконфигурация подключает бины по условиям и может уступить пользовательскому bean.

**Проверь себя.** Почему добавление JDBC starter не обязательно создаст стандартный DataSource?

Пользовательский бин может удовлетворить зависимость, тогда условная автоконфигурация отступит.
