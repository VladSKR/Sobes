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

## Разбор интервью: три части SpringBootApplication и минимальный starter

`@SpringBootApplication` объединяет:
- **`@SpringBootConfiguration`** — специальный `@Configuration` главного класса; Spring обрабатывает его `@Bean`-методы. Это **не дополнительный scanner**.
- **`@ComponentScan`** — находит `@Component`, `@Service`, `@Repository`, `@Configuration` в пакете приложения и подпакетах. Поэтому соседний `PaymentConfig` с `@Bean fun fraudClient()` находится именно scanner-ом.
- **`@EnableAutoConfiguration`** — использует `AutoConfigurationImportSelector` и записи `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`. Это не поиск всех аннотаций `@Bean` во всех JAR.

Ключевой порядок: на этапе обработки configuration classes регистрируются **BeanDefinition**, затем создаются non-lazy singleton экземпляры. `@ConditionalOnMissingBean` проверяет уже зарегистрированные **определения** пользовательских бинов; не обязательно уже созданные объекты. При стандартной конфигурации user beans обычно объявляются раньше auto-config и автоконфигурация уступает им.

Если конфигурация `com.external.PaymentConfig` лежит вне scanned packages, подключи **явно** `@Import(PaymentConfig::class)`. Для стартера, который должен включаться лишь при зависимости, используй `@AutoConfiguration` + файл `AutoConfiguration.imports`. В файле указываются **полные имена классов автоконфигураций**, а не `@Bean`-методы или `PaymentClient`.

Самый простой starter:

```kotlin
// в библиотеке
class PaymentClient

@AutoConfiguration
@ConditionalOnClass(PaymentClient::class)
class PaymentAutoConfiguration {
    @Bean
    @ConditionalOnMissingBean(PaymentClient::class)
    fun paymentClient(): PaymentClient = PaymentClient()
}
```

```text
# src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
com.mybank.PaymentAutoConfiguration
```

В Gradle библиотека подключает `spring-boot-autoconfigure` с версией через BOM; consumer подключает опубликованный JAR, и Spring Boot подхватывает конфигурацию. Для production starter часто разделяют модуль `autoconfigure` и модуль зависимостей `starter`, а поведение проверяют `ApplicationContextRunner`. `@ConditionalOnClass` проверяет наличие класса, `@ConditionalOnMissingBean` — отсутствие пользовательского определения. В зависимости от версии/optional dependency можно использовать `name = ["fully.qualified.Class"]`.

**Проверь себя.** Почему конфигурация `PaymentConfig` в своём пакете находится без `AutoConfiguration.imports`, а конфигурация в стороннем JAR не находится автоматически обычным component scan?

Потому что scan ограничен пакетами приложения; для внешней конфигурации нужен явный `@Import` или механизм auto-configuration из зарегистрированных imports.

## Two-Minute Drill

- **Из интервью:** вспомни сквозной сценарий выше, его ограничения и вопрос для самопроверки.

- Spring Boot добавляет автоконфигурацию, starters и удобный запуск приложения вокруг Spring Framework.
- Стартер — набор зависимостей; автоконфигурация подключает бины по условиям и может уступить пользовательскому bean.

**Проверь себя.** Почему добавление JDBC starter не обязательно создаст стандартный DataSource?

Пользовательский бин может удовлетворить зависимость, тогда условная автоконфигурация отступит.
