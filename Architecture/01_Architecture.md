# Архитектура и распределённые системы

#architecture #distributed #microservices

---

## CAP теорема

В распределённой системе можно выбрать только **2 из 3**:

| Свойство | Описание |
|---|---|
| **C** Consistency | Каждое чтение возвращает самую актуальную запись |
| **A** Availability | Каждый живой узел всегда отвечает на запросы |
| **P** Partition tolerance | Узлы продолжают работать при сетевых разрывах |

**Практически:** P всегда нужен в реальных системах → выбор между CA:
- **CP**: согласованность + устойчивость (HBase, ZooKeeper, MongoDB в режиме strong)
- **AP**: доступность + устойчивость (Cassandra, CouchDB, DNS)

---

## SAGA

Паттерн для управления распределёнными транзакциями через цепочку локальных транзакций.

Каждая локальная транзакция:
1. Обновляет свою БД
2. Публикует событие/команду для следующей транзакции
3. При ошибке — запускает **компенсирующие транзакции** (rollback через отмену)

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

✅ Нет единой точки отказа  
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
❌ Единая точка отказа, связанность

### Преимущества SAGA
- Согласованность без распределённых транзакций (2PC)
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
- HTTP: запрос-ответ, соединение закрывается
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
    .onBackpressureBuffer(100)    // буфер 100 элементов
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
