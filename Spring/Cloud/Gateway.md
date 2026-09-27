# Spring Cloud Gateway

## За две минуты

- Gateway принимает внешний запрос, выбирает маршрут и может применить фильтры до/после вызова downstream.
- Он не делает downstream отказоустойчивым без timeout, retry policy и ограничений нагрузки.

---

## Как работает маршрут

Downstream — сервис, к которому Gateway перенаправляет запрос. Например, клиент просит `/api/users/42`, Gateway выбирает маршрут по Path и отправляет запрос сервису пользователей. Фильтр может убрать `/api` или добавить заголовок, но бизнес-операцию выполняет уже целевой сервис.

Spring Cloud Gateway для реактивного стека использует WebFlux.

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
            - AddRequestHeader=X-Request-Id, {requestId}
```

**Возможности:**
- Routing по path, header, method
- Фильтры (rate limiting, auth, logging)
- Load balancing через Spring Cloud LoadBalancer
- Circuit Breaker интеграция
