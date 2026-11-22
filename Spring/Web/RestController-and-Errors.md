# @RestController и HTTP-ответ

`@RestController` сочетает `@Controller` и `@ResponseBody`: результат обработчика записывается в тело ответа через подходящий `HttpMessageConverter` (для JSON обычно Jackson). `@RequestMapping`, `@GetMapping` и другие связывают маршрут, HTTP-метод и обработчик; `@RequestBody` читает тело запроса, `@PathVariable` — часть пути.

`ResponseEntity<T>` позволяет явно задать status, headers и body. Для ошибок обычно применяют `@ExceptionHandler` / `@ControllerAdvice` и согласованный формат ответа, а не ловят все исключения внутри каждого контроллера. Валидация `@Valid` относится к аргументам при настроенном валидаторе; ошибка валидации — ожидаемый клиентский ответ, а не 500.

```java
@PostMapping("/users")
public ResponseEntity<UserDto> create(@Valid @RequestBody CreateUser body) {
    UserDto user = service.create(body);
    return ResponseEntity.created(URI.create("/users/" + user.id()))
                         .body(user);
}
```

На интервью расширяй ответ после «возвращает JSON»: маршрутизация, десериализация/валидация, контроллер → сервис, сериализация, status/headers, обработка ошибок. Контракт HTTP — [[API/REST-and-Protocols]].

## Три исхода одного HTTP-запроса

При `POST /users` Spring сначала выбирает handler по HTTP-методу и пути. Конвертер читает JSON в DTO, затем валидатор проверяет поля. Неверный JSON и невалидный DTO обычно приводят к 400 ещё до вызова сервиса. Если данные корректны, сервис создаёт пользователя и контроллер возвращает 201 с заголовком `Location`.

Если email уже используется, доменная операция может выбросить `EmailAlreadyUsed`. Маппинг можно централизовать в `@RestControllerAdvice`.

```java
@ExceptionHandler(EmailAlreadyUsed.class)
ProblemDetail duplicateEmail(EmailAlreadyUsed error) {
    return ProblemDetail.forStatusAndDetail(
        HttpStatus.CONFLICT, "Email is already registered");
}
```

API получает стабильный 409. Для неожиданного сбоя возвращай обобщённый ответ 500, а подробности диагностики оставляй в журнале с идентификатором запроса. Проверь статусы, заголовки и JSON тестом MVC. [Контракт HTTP](../../API/REST-and-Protocols.md)

## Two-Minute Drill

- `@RestController` связывает HTTP маршрут с методом и сериализует результат в тело ответа.
- `ResponseEntity` позволяет выбрать status/headers/body, а `@ControllerAdvice` централизует обработку ошибок.
- Контроллеру стоит передавать бизнес-операцию сервису, а не держать все правила в HTTP-слое.

**Проверь себя.** Дойдёт ли тело с невалидным JSON до сервиса?

Обычно нет. Десериализация запроса завершается ошибкой до вызова метода контроллера.

Источник: [Spring MVC annotated controllers](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-controller.html).
