# REST / API / Протоколы

#rest #api #http #soap

---

## REST — ограничения

1. **Client-Server** — разделение ответственности
2. **Stateless** — сервер не хранит состояние клиента
3. **Cacheable** — ответы могут кэшироваться
4. **Uniform Interface** — единый интерфейс (URI, HTTP методы)
5. **Layered System** — клиент не знает, с каким слоем общается
6. **Code on Demand** (опционально) — сервер может слать исполняемый код

---

## HTTP методы и идемпотентность

| Метод | Идемпотентный | Безопасный | Использование |
|---|---|---|---|
| GET | ✅ | ✅ | Получить ресурс |
| HEAD | ✅ | ✅ | Только заголовки |
| OPTIONS | ✅ | ✅ | Возможности сервера |
| PUT | ✅ | ❌ | Полная замена ресурса |
| DELETE | ✅ | ❌ | Удалить ресурс |
| POST | ❌ | ❌ | Создать ресурс / действие |
| PATCH | ❌ | ❌ | Частичное обновление |

**Идемпотентный** = повторный вызов даёт тот же результат  
**Безопасный** = не изменяет состояние сервера

---

## PATCH vs PUT

| | PUT | PATCH |
|---|---|---|
| Что передаётся | Полный объект | Только изменяемые поля |
| Семантика | Замена | Частичное обновление |
| Идемпотентность | ✅ | ❌ (зависит от реализации) |

```http
# PUT — передаём весь объект
PUT /users/1
{"name": "Alice", "email": "alice@example.com", "age": 30}

# PATCH — только изменяемое поле
PATCH /users/1
{"age": 31}
```

---

## HATEOAS

**Hypermedia As The Engine Of Application State** — высший уровень REST.

Ответ содержит ссылки на возможные действия:

```json
{
  "id": 1,
  "status": "pending",
  "_links": {
    "self": {"href": "/orders/1"},
    "approve": {"href": "/orders/1/approve", "method": "POST"},
    "cancel": {"href": "/orders/1/cancel", "method": "POST"}
  }
}
```

Клиент не хардкодит URL — использует ссылки из ответа.

---

## HTTP статус-коды

| Диапазон | Значение | Примеры |
|---|---|---|
| 2xx | Успех | 200 OK, 201 Created, 204 No Content |
| 3xx | Редирект | 301 Moved, 304 Not Modified |
| 4xx | Ошибка клиента | 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 409 Conflict, 422 Unprocessable |
| 5xx | Ошибка сервера | 500 Internal Error, 502 Bad Gateway, 503 Unavailable |

---

## SOAP

**Simple Object Access Protocol** — протокол для обмена сообщениями на основе XML.

### Структура сообщения
```xml
<soap:Envelope>
    <soap:Header>
        <!-- авторизация, маршрутизация -->
    </soap:Header>
    <soap:Body>
        <getProductDetails>
            <productID>12345</productID>
        </getProductDetails>
    </soap:Body>
    <!-- soap:Fault — при ошибке -->
</soap:Envelope>
```

### REST vs SOAP

| | REST | SOAP |
|---|---|---|
| Формат | JSON/XML/любой | Только XML |
| Протокол | HTTP | HTTP, SMTP, другие |
| Стандарт | Архитектурный стиль | Протокол |
| Контракт | OpenAPI (опционально) | WSDL (обязателен) |
| Сложность | Проще | Сложнее |
| Транзакции | Сложно | WS-AtomicTransaction |

---

## jRPC (JSON-RPC)

Лёгкий RPC протокол на основе JSON.

```json
// Запрос
{"jsonrpc": "2.0", "method": "subtract", "params": [42, 23], "id": 1}

// Ответ
{"jsonrpc": "2.0", "result": 19, "id": 1}
```

---
