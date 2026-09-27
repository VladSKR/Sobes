# Архитектура и распределённые системы

## За две минуты

- При сетевом разделении узлы могут перестать обмениваться данными: тогда доступность и единая актуальная картина вступают в конфликт.
- Saga координирует несколько локальных транзакций через состояния и компенсации, а не делает общий SQL rollback.
- Для внешних вызовов сначала определи timeout, idempotency и поведение при неизвестном результате.

---

#architecture #distributed #microservices

---

## CAP теорема

Если между двумя группами узлов разорвана сеть (partition), они не могут одновременно знать о новых записях друг друга. В этот момент системе приходится выбирать поведение для затронутых запросов:

- **Consistency (C)** в смысле CAP: чтение ведёт себя как единая актуальная копия (линеаризуемость).
- **Availability (A)**: каждый запрос к работающему узлу получает ответ, а не отказ из-за ожидания другой части кластера.
- **Partition tolerance (P)**: рассматриваем работу при разрыве связи между узлами.

Например, при разрыве связи узел может отказать в записи, чтобы не принять противоречащее изменение (**сохраняет C, теряет A для части запросов**), либо принять запись, но тогда разные части кластера временно дадут несовместимые ответы (**сохраняет A, теряет C**). Фраза «просто выбираем любые два из трёх» скрывает главное: компромисс проявляется **во время сетевого разделения**. Названия продуктов сами по себе не определяют режим всех их операций.

Источник: [объяснение CAP в AWS](https://docs.aws.amazon.com/whitepapers/latest/availability-and-beyond-improving-resilience/cap-theorem.html).

---

## SAGA

Паттерн для управления распределёнными транзакциями через цепочку локальных транзакций.

Каждая локальная транзакция:
1. Обновляет свою БД
2. Публикует событие/команду для следующей транзакции
3. При ошибке — может запускать **компенсирующие действия**. Это новые операции, а не откат общей DB-транзакции

### Хореография (Choreography)
Каждый сервис публикует события, другие подписываются и реагируют.

```
OrderService → [OrderCreated] → PaymentService
PaymentService → [PaymentCompleted] → InventoryService
InventoryService → [ItemReserved] → OrderService (approve)

При ошибке:
InventoryService → [OutOfStock] → PaymentService (refund)
PaymentService → [PaymentRefunded] → OrderService (cancel)
```

✅ Нет центрального координатора (но брокер/БД всё равно могут отказать)  
❌ Сложно отслеживать общее состояние

### Оркестровка (Orchestration)
Центральный оркестратор управляет последовательностью.

```
CreateOrderSaga (оркестратор):
1. → ReserveCredit → CustomerService
2. ← ответ от CustomerService
3. → ApproveOrder / RejectOrder → OrderService
```

✅ Легко отслеживать состояние  
❌ Нужна надёжность самого оркестратора и хранение его состояния

### Преимущества SAGA
- Координация локальных транзакций без общего ACID commit
- Каждый сервис имеет свою БД

### Недостатки SAGA
- Сложное программирование компенсирующих транзакций
- Нет изолированности между сагами (видны промежуточные состояния)

---

## Circuit Breaker

Паттерн для предотвращения каскадных отказов.

```
CLOSED → (ошибки > threshold) → OPEN → (timeout) → HALF-OPEN
   ↑                                                      |
   ←————————————— успешный запрос ←————————————————————————
```

| Состояние | Поведение |
|---|---|
| **CLOSED** | Запросы проходят нормально, считаются ошибки |
| **OPEN** | Запросы сразу отклоняются (fail-fast), ждём timeout |
| **HALF-OPEN** | Пропускаем пробный запрос — если успех → CLOSED, если нет → OPEN |

```java
// Resilience4j
@CircuitBreaker(name = "paymentService", fallbackMethod = "paymentFallback")
public String processPayment(Order order) {
    return paymentClient.process(order);
}

public String paymentFallback(Order order, Exception e) {
    return "Payment service unavailable";
}
```

---

## OSI Model

| Уровень | Название | Протоколы |
|---|---|---|
| 7 | Application | HTTP, FTP, SMTP, DNS |
| 6 | Presentation | SSL/TLS, JPEG, XML |
| 5 | Session | NetBIOS, RPC |
| 4 | Transport | TCP, UDP |
| 3 | Network | IP, ICMP |
| 2 | Data Link | Ethernet, WiFi |
| 1 | Physical | Кабели, сигналы |

---

## WebSocket

Двустороннее постоянное соединение между клиентом и сервером.

```
HTTP Handshake:
GET /chat HTTP/1.1
Upgrade: websocket
Connection: Upgrade

101 Switching Protocols
← двустороннее соединение установлено →
```

**vs HTTP:**
- HTTP: модель запрос-ответ; само TCP-соединение может переиспользоваться (keep-alive)
- WebSocket: постоянное соединение, обе стороны могут слать сообщения

**Когда использовать:** чат, real-time уведомления, игры, совместное редактирование.

---

## SSE (Server-Sent Events)

Однонаправленный поток событий: сервер → клиент.

```java
@GetMapping(value = "/events", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public SseEmitter streamEvents() {
    SseEmitter emitter = new SseEmitter();
    executor.execute(() -> {
        emitter.send(SseEmitter.event().data("message 1"));
        emitter.send(SseEmitter.event().data("message 2"));
        emitter.complete();
    });
    return emitter;
}
```

**vs WebSocket:** SSE — только сервер→клиент, WebSocket — двусторонний.

---

## Webhook

Клиент регистрирует URL → сервер вызывает этот URL при событии.

```
1. Клиент: POST /webhooks {"url": "https://my-app.com/callback"}
2. Событие в системе
3. Сервер: POST https://my-app.com/callback {"event": "payment.completed"}
```

---

## Callback

Функция, передаваемая как аргумент, вызывается по завершении операции.

```java
// Синхронный callback
service.process(data, result -> handleResult(result));

// Асynchronous callback через CompletableFuture
service.processAsync(data)
       .thenAccept(result -> handleResult(result));
```

---

## Backpressure

Механизм управления потоком данных — производитель не должен перегружать потребителя.

```java
// Reactor (Project Reactor)
Flux.range(1, 1000)
    .onBackpressureBuffer(100)    // ограниченный буфер, но нужно решить, что делать при его переполнении
    .subscribe(item -> process(item));

// или
.onBackpressureDrop(dropped -> log("dropped: " + dropped))
.onBackpressureLatest() // только последний элемент
```

---

## RSocket

Реактивный бинарный протокол с встроенной backpressure поддержкой.

**4 модели взаимодействия:**
1. **Request-Response** — как HTTP, но реактивный
2. **Request-Stream** — один запрос, много ответов (сервер стримит)
3. **Fire-and-Forget** — запрос без ответа
4. **Channel** — двусторонний стрим

```java
// Spring RSocket
@MessageMapping("greet")
public Mono<String> greet(String name) {
    return Mono.just("Hello, " + name);
}

@MessageMapping("stream")
public Flux<String> stream() {
    return Flux.interval(Duration.ofSeconds(1))
               .map(i -> "event " + i);
}
```

---

## Горизонтальное масштабирование

**Вертикальное (Scale Up):** больше CPU/RAM на одном сервере. Ограничено железом.

**Горизонтальное (Scale Out):** добавление новых серверов. Требует:
- Stateless сервисы (состояние во внешнем хранилище)
- Load Balancer перед сервисами
- Общий кэш (Redis)
- Общая БД или шардирование
- Distributed tracing (идентификаторы запросов)

---

## Профилирование запросов

**Перцентиль (Percentile):**
- p50 = медиана (50% запросов быстрее этого времени)
- p95 = 95% запросов быстрее (важнее среднего!)
- p99 = "долгий хвост" (самые медленные 1%)

**Почему p95/p99 важнее среднего:**  
Среднее скрывает outliers. Если 99% запросов за 10ms, а 1% за 5000ms — среднее будет 60ms, но реальный UX плохой.

```
Пример: 1000 запросов
- Average: 60ms   ← не показывает реальную картину
- p50: 10ms
- p95: 100ms
- p99: 5000ms     ← вот где проблема
```

---

## Симметричное vs Асимметричное шифрование

| | Симметричное | Асимметричное |
|---|---|---|
| Ключи | Один ключ (encrypt + decrypt) | Пара: публичный + приватный |
| Скорость | Быстрее | Медленнее |
| Алгоритмы | AES, DES | RSA, ECC |
| Проблема | Как передать ключ безопасно? | Нет проблемы передачи |
| Использование | Шифрование данных | Обмен ключами, подпись, TLS handshake |

**TLS использует оба:**
1. Асимметричное — для безопасного обмена сессионным ключом
2. Симметричное — для шифрования самого трафика (быстрее)
