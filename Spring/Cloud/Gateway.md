# Spring Cloud Gateway

## Spring Cloud Gateway

API Gateway для микросервисов на реактивном стеке (WebFlux).

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
