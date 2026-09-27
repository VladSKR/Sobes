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

Источник: [Spring MVC annotated controllers](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-controller.html).
