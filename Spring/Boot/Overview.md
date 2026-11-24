# Spring Boot: запуск приложения, автоконфигурация и собственный стартер

Spring Boot расширяет Spring Framework соглашениями, starter-зависимостями, управлением совместимыми версиями и **условной автоконфигурацией**. Основная практическая задача — подключать инфраструктурные компоненты (HTTP-клиенты, драйверы, метрики) без повторения одинаковой конфигурации во всех сервисах.

## Три механизма внутри @SpringBootApplication

```kotlin
@SpringBootApplication
class Application

fun main(args: Array<String>) {
    runApplication<Application>(*args)
}
```

Аннотация объединяет три разных назначения:

| Аннотация | Какую задачу решает | Что происходит |
|---|---|---|
| `@SpringBootConfiguration` | Помечает основной класс как конфигурацию | Метааннотирована `@Configuration`; её методы `@Bean` участвуют в регистрации bean definitions |
| `@ComponentScan` | Находит компоненты приложения | Сканирует пакет основного класса и подпакеты, включая `@Service`, `@Repository`, `@Configuration` |
| `@EnableAutoConfiguration` | Подключает конфигурации из библиотек | Импортирует перечисленные в `AutoConfiguration.imports` классы, фильтруя их по условиям |

Это **не три последовательных прохода, создающих готовые объекты**. В ходе обработки конфигураций Spring регистрирует `BeanDefinition` (описание бина), затем во время инициализации контекста создаёт обычные eager singletons. Порядок разбирается в [Spring Boot Internals](Internals.md).

## @Configuration, @Bean, @ComponentScan и @Import

Класс конфигурации может предоставлять фабричные методы для объектов, исходный код которых нельзя или не нужно аннотировать:

```kotlin
class PaymentClient(val endpoint: String)

@Configuration
class PaymentConfiguration {
    @Bean
    fun paymentClient(): PaymentClient = PaymentClient("https://payments.example")
}
```

Если `Application` размещён в `com.example`, а `PaymentConfiguration` — в `com.example.payments`, то `@ComponentScan` найдёт конфигурацию: `@Configuration` сама помечена `@Component`. Spring обработает `@Bean`-метод, зарегистрирует определение `PaymentClient` и позже создаст объект. Сам `PaymentClient` не обязан иметь `@Component`.

Если `PaymentConfiguration` находится в **`org.external`**, вне сканируемых пакетов, класс можно импортировать явно:

```kotlin
@SpringBootApplication
@Import(PaymentConfiguration::class)
class Application
```

`@Import` — **явное решение приложения**, оно не требует ни изменения `@ComponentScan`, ни публикации starter.

## Как работает AutoConfiguration

Spring Boot 3.x находит *имена классов автоконфигураций* в ресурсе внутри JAR:

```text
META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
```

`AutoConfigurationImportSelector` импортирует кандидатов. Условия определяют, какие определения бинов разрешено зарегистрировать:

- `@ConditionalOnClass` — класс доступен в classpath; полезно для необязательных библиотек.
- `@ConditionalOnProperty` — конфигурационный параметр соответствует требуемому значению.
- `@ConditionalOnMissingBean` — подходящий пользовательский бин не зарегистрирован; стандартная конфигурация **отступает**.

`@ConditionalOnMissingBean` обычно оценивает **зарегистрированные определения**, а не уже созданные экземпляры. Автоконфигурации обрабатываются после пользовательских объявлений, поэтому пользовательский `PaymentClient` может заменить стандартный **без перезаписи созданного объекта**. Порядок между самими автоконфигурациями может задаваться `@AutoConfigureBefore`/`@AutoConfigureAfter` или эквивалентными параметрами `@AutoConfiguration`.

## Минимальный starter: PaymentClient

Цель: добавить библиотеку в Gradle другого сервиса и получить готовый бин `PaymentClient` без `@Import` в приложении. Для учебного примера достаточно одного Gradle-модуля, объединяющего starter и autoconfigure; в больших библиотеках их разделяют.

```text
payment-spring-boot-starter/
  build.gradle.kts
  src/main/kotlin/com/example/payment/PaymentClient.kt
  src/main/kotlin/com/example/payment/PaymentAutoConfiguration.kt
  src/main/resources/META-INF/spring/
    org.springframework.boot.autoconfigure.AutoConfiguration.imports
```

Класс клиента (без Spring-аннотаций):

```kotlin
package com.example.payment

class PaymentClient {
    fun pay(amount: Long): String = "accepted: $amount"
}
```

Автоконфигурация:

```kotlin
package com.example.payment

import org.springframework.boot.autoconfigure.AutoConfiguration
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean
import org.springframework.context.annotation.Bean

@AutoConfiguration
class PaymentAutoConfiguration {
    @Bean
    @ConditionalOnMissingBean(PaymentClient::class)
    fun paymentClient(): PaymentClient = PaymentClient()
}
```

Содержимое `AutoConfiguration.imports`:

```text
com.example.payment.PaymentAutoConfiguration
```

Зависимость библиотеки (версия Spring Boot задаётся согласованной BOM/dependency management):

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-autoconfigure")
}
```

После публикации JAR сервис-потребитель добавляет `implementation("com.example:payment-spring-boot-starter:1.0.0")` и внедряет `PaymentClient` через конструктор. В `AutoConfiguration.imports` перечислены **классы автоконфигурации**, а не `@Bean`-методы и не сам `PaymentClient`. `@Bean` — аннотация метода, не класса.

Для необязательной SDK-зависимости добавляют, например, `@ConditionalOnClass(name = ["org.external.PaymentSdk"])` к автоконфигурации или отдельной вложенной конфигурации. Строковая форма помогает избежать жёсткой загрузки отсутствующего класса при анализе метаданных. Для Kotlin Spring-проектов типично использование `kotlin("plugin.spring")`, раскрывающего нужные Spring классы для проксирования.

## Как проверить автоконфигурацию

Тест через `ApplicationContextRunner` должен подтверждать **оба** сценария: при отсутствии пользовательского бина создаётся `PaymentClient`; при наличии своего определения остаётся именно оно. Диагностировать не сработавшее условие можно через conditions report (`--debug`) или Actuator `conditions` при безопасной настройке доступа.

## Two-Minute Drill

- `@SpringBootConfiguration` — основной `@Configuration`; `@ComponentScan` — обнаружение компонентов в пакетах; `@EnableAutoConfiguration` — условный импорт библиотечных конфигураций.
- `@Import` явным образом подключает конфигурационный класс вне сканирования.
- `AutoConfiguration.imports` содержит полные имена `@AutoConfiguration`-классов, не бинов; сами бины объявляются `@Bean`-методами.
- `@ConditionalOnMissingBean` позволяет пользовательским `BeanDefinition` иметь приоритет над стандартными.
- Регистрация определения бина предшествует созданию экземпляра.

**Вопрос 1.** Почему `@Configuration` в подпакете приложения обнаруживается без `AutoConfiguration.imports`?  
**Ответ:** это компонент, который обнаружит `@ComponentScan`.

**Вопрос 2.** Зачем `@ConditionalOnMissingBean` в starter?  
**Ответ:** чтобы библиотека не создавала альтернативный `PaymentClient`, когда приложение уже определило свою реализацию.

Источник: [Spring Boot — Creating Your Own Auto-configuration](https://docs.spring.io/spring-boot/reference/features/developing-auto-configuration.html).
