# Управление зависимостями

## Maven/Gradle — Dependency Management

Зависимость — библиотека, которую модуль использует при компиляции, тестировании или запуске. Если `service` зависит от Boot starter, а starter тянет библиотеку JSON, для `service` библиотека JSON будет транзитивной зависимостью.

| Раздел родительского POM | Что получает дочерний модуль |
|---|---|
| `dependencies` | Наследует сами объявления зависимостей |
| `dependencyManagement` | Наследует правила выбора версии и scope. Библиотека попадает в граф через прямую или транзитивную зависимость |

Например, родитель задаёт согласованную версию `jackson-databind` в `dependencyManagement`. В дочернем модуле можно объявить зависимость без версии, либо получить её транзитивно. Чтобы понять фактическое разрешение, запусти `mvn dependency:tree`.

### Maven BOM (Bill of Materials)
```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId>
            <version>3.2.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<!-- Теперь версии можно не указывать -->
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>
```

### Scopes Maven
| Scope | Компиляция | Тесты | Runtime |
|---|---|---|---|
| compile (default) | ✅ | ✅ | ✅ |
| test | ❌ | ✅ | ❌ |
| provided | ✅ | ✅ | ❌ (сервер предоставит) |
| runtime | ❌ | ✅ | ✅ |


### Тот же BOM в Gradle

```kotlin
dependencies {
    implementation(platform("org.springframework.boot:spring-boot-dependencies:3.2.0"))
    implementation("org.springframework.boot:spring-boot-starter-web")
}
```

Gradle импортирует ограничения версий через `platform`. `implementation` добавляет starter, а BOM задаёт согласованный набор версий. Итоговый граф можно посмотреть через `./gradlew dependencies --configuration runtimeClasspath`.

## Два модуля, две версии одной библиотеки

Пусть приложение подключает клиенты A и B, которые транзитивно запрашивают разные версии JSON-библиотеки. Сначала посмотри итоговый граф и правило выбора версии. Maven обычно выбирает ближайшую зависимость при отсутствии управления версией. Gradle имеет свой механизм согласования конфликтов и constraints. BOM задаёт общий набор версий для выбранного выпуска платформы. После выравнивания запусти тест интеграции: конфликт байткода может проявиться как `NoSuchMethodError` только при выполнении соответствующей ветки.

## Two-Minute Drill

- Maven BOM централизует согласованные версии зависимостей; scope определяет, где зависимость нужна.
- `dependencyManagement` сам по себе не добавляет библиотеку в classpath проекта.

**Проверь себя.** Родительский POM управляет версией библиотеки, но модуль и все его зависимости обходятся без неё. Появится ли библиотека в runtime classpath? Нет, управление версией само по себе её не подключает.

[Maven Dependency Mechanism](https://maven.apache.org/guides/introduction/introduction-to-dependency-mechanism.html) · [Gradle Platforms](https://docs.gradle.org/current/userguide/platforms.html)
