# Метрики, перцентили и разбор инцидента

p50 — медиана задержки, p95/p99 показывают хвост. Средняя latency скрывает редкие многосекундные запросы; сравнивай per-endpoint, status, downstream и нагрузку. У метрики нужны понятные labels, но user ID в label создаёт неконтролируемую кардинальность.

При инциденте: начни с пользовательского симптома и окна времени → RPS/errors/latency/saturation → traces с correlation ID → медленные SQL и EXPLAIN → pool и очереди → recent deploy/config. Сопоставляй метрики с бизнес-инвариантами: число stuck pending orders, outbox lag, возраст reservations, расхождение ledger/balance. Разделяй обнаружение, временное ограничение ущерба и постоянное исправление.

Краткий ответ по перцентилям уже есть в [[Review/Quick-Questions]]; здесь — применение к System Design переводов и заказов: [[Architecture/Money-Transfer-and-Order-Design]].
