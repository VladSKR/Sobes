# Чеклист Senior-подготовки

### Многопоточность — минимум знать:
- [ ] volatile: видимость, не атомарность
- [ ] synchronized: монитор, happens-before
- [ ] ReentrantLock: tryLock, таймаут, fairness
- [ ] ReadWriteLock: когда использовать
- [ ] AtomicInteger/AtomicReference
- [ ] ConcurrentHashMap и его атомарные операции
- [ ] ThreadLocal — что это и где используется (например в Spring транзакциях)
- [ ] ExecutorService, CountDownLatch, Semaphore

### Kotlin корутины — минимум знать:
- [ ] Dispatchers.IO vs Default vs Unconfined
- [ ] structured concurrency — отмена дочерних при падении родителя
- [ ] CoroutineScope и его жизненный цикл
- [ ] SupervisorJob — когда дочерние не отменяют друг друга
- [ ] Flow vs Channel

### Hibernate — минимум знать:
- [ ] batch_size настройка
- [ ] IDENTITY vs SEQUENCE — почему IDENTITY ломает батчинг
- [ ] N+1 проблема и решения (JOIN FETCH, EntityGraph, batch fetching)
- [ ] LazyInitializationException — почему возникает

### GC — минимум знать:
- [ ] G1: регионы, паузы, когда использовать
- [ ] ZGC: конкурентная работа, цели коротких пауз и проверка GC logs без жёсткой гарантии в миллисекундах
- [ ] Как читать GC логи
- [ ] OutOfMemoryError vs StackOverflowError — разница

### Race condition — рефлекс:
- [ ] Видеть check-then-act паттерн
- [ ] Знать атомарные альтернативы
- [ ] В финансовых системах — думать об этом в первую очередь
