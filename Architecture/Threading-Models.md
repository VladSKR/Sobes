# Thread per request vs Event Loop vs Virtual Threads

## 6. Thread per request vs Event Loop vs Virtual Threads

### Thread per request (Spring MVC)

```
Запрос 1 → Поток 1 (занят до ответа)
Запрос 2 → Поток 2 (занят до ответа)
...
Запрос 51 → очередь ожидания (или отказ если очередь полна)

Tomcat defaults:
  max-threads: 200      (максимум потоков)
  accept-count: 100     (размер очереди)
  → максимум 300 одновременных соединений
```

**Проблема:** поток заблокирован пока ждёт БД/HTTP — OS thread занят но ничего не делает.

### Что происходит с 51-м запросом

```
50 потоков заняты → 51-й запрос идёт в очередь accept-count
Если очередь тоже полна → Connection refused
Если в очереди есть место → ждёт пока освободится поток

Настройка:
server:
  tomcat:
    max-threads: 200
    accept-count: 100
```

### Event Loop (WebFlux)

```
4 потока (= CPU ядра) обрабатывают тысячи запросов
Поток не блокируется на I/O — отдаёт управление другим запросам
При получении ответа — продолжает обработку

Минус: весь код должен быть реактивным (Mono/Flux)
       нельзя использовать блокирующие библиотеки
```

### Virtual Threads — Java 21 (лучшее решение без WebFlux)

```java
// Виртуальный поток — очень дешёвый (~1KB vs ~1MB для OS потока)
// Можно создать миллион виртуальных потоков
// При блокировании на I/O — OS поток освобождается автоматически
// Блокирующий код работает как прежде — не нужен реактивный стиль
```

```yaml
# Spring Boot 3.2+ — одна строка конфига
spring:
  threads:
    virtual:
      enabled: true
# Tomcat начинает использовать виртуальные потоки для каждого запроса
```

```java
// Или вручную
@Bean
public TomcatProtocolHandlerCustomizer<?> virtualThreadsProtocol() {
    return handler -> handler.setExecutor(Executors.newVirtualThreadPerTaskExecutor());
}
```

### Сравнение

| | MVC + OS Threads | WebFlux | Virtual Threads |
|---|---|---|---|
| Модель | thread per request | event loop | virtual thread per request |
| Код | обычный блокирующий | реактивный Mono/Flux | обычный блокирующий |
| Потоки | ~200 OS | ~CPU ядер | миллионы виртуальных |
| Миграция | — | сложная, переписывать | простая, одна строка |
| Java версия | любая | любая | 21+ |
| Подходит | низкая нагрузка | высокая нагрузка, I/O | высокая нагрузка, I/O |

### Connection Pool — важная деталь

```
Virtual Threads решают проблему потоков
Но connection pool к БД остаётся ограниченным!

100_000 виртуальных потоков → все хотят соединение с БД
HikariCP по умолчанию: 10 соединений

Решение: увеличить pool или использовать R2DBC
spring:
  datasource:
    hikari:
      maximum-pool-size: 50
```

---
