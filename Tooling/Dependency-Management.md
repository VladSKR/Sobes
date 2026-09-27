# Управление зависимостями

## Maven/Gradle — Dependency Management

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
