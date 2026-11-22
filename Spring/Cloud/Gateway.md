# Spring Cloud Gateway

## Как работает маршрут

Downstream — сервис, к которому Gateway перенаправляет запрос. Например, клиент просит `/api/users/42`, Gateway выбирает маршрут по Path и отправляет запрос сервису пользователей. Фильтр может убрать `/api` или добавить заголовок, но бизнес-операцию выполняет уже целевой сервис.

Пример использует конфигурацию Spring Cloud Gateway 4.2 для WebFlux. Версии с другим starter или префиксом следует настраивать по их справочнику.

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://USER-SERVICE  # load balanced
          predicates:
            - Path=/api/users/**
          filters:
            - StripPrefix=1
            - AddRequestHeader=X-Gateway, public-api
```

**Возможности:**
- Routing по path, header, method
- Фильтры (rate limiting, auth, logging)
- Load balancing через Spring Cloud LoadBalancer
- Circuit Breaker интеграция

## Фильтры и повтор вызова

Для `GET /api/users/42` сначала проверяется predicate `Path`. После выбора маршрута `StripPrefix=1` передаёт downstream путь `/users/42`. URI `lb://USER-SERVICE` требует разрешения имени сервиса через Spring Cloud LoadBalancer. Фильтры могут менять запрос до отправки и ответ при возвращении. Конфигурационный префикс зависит от версии и варианта Gateway, поэтому сверяй его с выбранным starter.

Если клиент получил 404, проверь, совпал ли маршрут и какой путь дошёл до сервиса. При 503 проверяй наличие инстансов и доступность downstream. Повтор запроса с телом может потребовать буферизации и увеличивает нагрузку. Для POST оплаты нужны явная семантика повтора и стабильный ключ операции. [Circuit Breaker](../../Architecture/Distributed-Systems.md#circuit-breaker) · [Spring Cloud Gateway](https://docs.spring.io/spring-cloud-gateway/reference/)

## Two-Minute Drill

- Gateway принимает внешний запрос, выбирает маршрут и может применить фильтры до/после вызова downstream.
- Он не делает downstream отказоустойчивым без timeout, retry policy и ограничений нагрузки.

**Проверь себя.** Куда направится /api/users/42 после StripPrefix=1?

На /users/42 в выбранном downstream.
