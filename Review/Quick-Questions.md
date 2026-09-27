# Быстрые вопросы к повторению

#questions #review

---

## Вопросы по разным разделам

Полные материалы находятся по ссылкам в ответах; новые вопросы добавляй в заметку соответствующей темы.


### 1. Как запустить программу через командную строку

```bash
# Компиляция одного файла
javac Main.java

# Компиляция с classpath
javac -cp ./libs/some.jar Main.java

# Запуск
java Main
java -cp ./classes:./libs/some.jar com.example.Main

# Запуск JAR
java -jar app.jar

# Запуск с параметрами JVM
java -Xms256m -Xmx1g -XX:+UseG1GC -jar app.jar

# Параметры приложения (Spring Boot)
java -jar app.jar --server.port=8080 --spring.profiles.active=prod
```

---

### 2. JIT — Just-In-Time Compiler

JVM начинает с интерпретации байткода. JIT компилирует "горячий" (часто вызываемый) код в нативный машинный код и кэширует в CodeCache.

**Уровни (Tiered Compilation):**
- 0 — интерпретатор
- 1–3 — C1 (быстрая компиляция, базовые оптимизации)
- 4 — C2 (агрессивные оптимизации, медленнее компилирует, быстрее выполняет)

**Оптимизации C2:** inlining методов, escape analysis, loop unrolling, devirtualization.

→ Подробнее: [[Java Core/JVM/Overview]]

---

### 3. Динамическое и статическое связывание

- **Статическое (compile-time):** `static`, `private`, `final` методы, перегрузка (overloading)
- **Динамическое (runtime):** переопределение (`@Override`), полиморфизм, через vtable

→ Подробнее: [[Java Core/Language/OOP#Динамическое vs Статическое связывание]]

---

### 4. Композиция, агрегация, ассоциация

- **Ассоциация** — объекты знают друг о друге, живут независимо
- **Агрегация** — "has-a", часть может жить без целого (Department → Employee)
- **Композиция** — "owns-a", часть не живёт без целого (House → Room)

→ Подробнее: [[Java Core/Language/OOP#Композиция, Агрегация, Ассоциация]]

---

### 5. getClass() — рефлексия

```java
Object obj = "hello";
Class<?> c = obj.getClass(); // java.lang.String (реальный тип, не Object)

// Отличие от instanceof
boolean isString = obj instanceof String; // учитывает наследование
boolean exact = obj.getClass() == String.class; // только точный класс
```

→ Подробнее: [[Java Core/Language/OOP#Рефлексия]]

---

### 6. Рефлексия в Spring

Spring использует рефлексию для:
1. Создания бинов (`Constructor.newInstance()`)
2. `@Autowired` полей (`field.setAccessible(true); field.set(bean, value)`)
3. Чтения аннотаций (`@Component`, `@Transactional`, `@Value`)
4. AOP — проксирование через CGLib/JDK Proxy
5. Jackson — сериализация/десериализация через поля

→ Подробнее: [[Spring/Core/Reflection#Рефлексия в Spring]]

---

### 7. Почему в Kotlin нет checked exceptions

Kotlin не заставляет объявлять `throws` или ловить checked exception на месте вызова. Java-метод всё равно может бросить `IOException` во время выполнения; его нужно обработать там, где есть стратегия восстановления. Не объясняй выбор языка «нарушением LSP»: это неверная причинно-следственная связь.

→ Подробнее: [[Kotlin/Exceptions#Почему нет обязательной проверки checked exceptions]]

---

### 8. Почему может быть видно только последнее исключение

При незловленном исключении JVM ищет `UncaughtExceptionHandler` в порядке:
1. Обработчик потока (`thread.getUncaughtExceptionHandler()`)
2. Обработчик ThreadGroup
3. Default handler (`Thread.getDefaultUncaughtExceptionHandler()`)

Если тело `try` бросило одно исключение, а `finally` бросил другое, наружу уйдёт исключение из `finally`: первое потеряется, если его специально не сохранить. В `try-with-resources` ошибка закрытия доступна через `getSuppressed()`. `UncaughtExceptionHandler` уже обрабатывает то исключение, которое дошло до границы потока; он не выбирает «последнее» из истории.

→ Подробнее: [[Java Core/Language/Exceptions#UncaughtExceptionHandler]]

---

### 9. Отслеживание исключений в потоках

Если в дочернем потоке брошено исключение — **main поток не упадёт**.

```java
Thread t = new Thread(() -> { throw new RuntimeException("в потоке"); });
t.setUncaughtExceptionHandler((thread, ex) -> log(ex));
t.start();
// main продолжает работу

// Через ExecutorService — исключение оборачивается в ExecutionException
Future<?> f = executor.submit(() -> { throw new RuntimeException(); });
f.get(); // только здесь узнаём об исключении
```

→ Подробнее: [[Java Core/Concurrency/Overview#Отслеживание исключений в потоках]]

---

### 10. История дженериков — в рантайме нет типов (Type Erasure)

До Java 5 коллекции были нетипизированы:
```java
List list = new ArrayList(); // хранит Object
list.add("string");
String s = (String) list.get(0); // ручной каст, ClassCastException в рантайме
```

Java 5 (2004) — добавили дженерики. Для обратной совместимости реализованы через **type erasure**: типы стираются компилятором, в байткоде остаётся `Object` + автоматические касты.

```java
List<String> list = new ArrayList<>();
// После erasure в байткоде: List list = new ArrayList()
// get(0) возвращает Object, компилятор добавляет (String) каст
```

→ Подробнее: [[Java Core/Language/Generics#Type Erasure (Стирание типов)]]

---

### 11. Метод в интерфейсе Iterable — Iterator iterator()

```java
public interface Iterable<T> {
    Iterator<T> iterator(); // единственный обязательный метод
}

public interface Iterator<T> {
    boolean hasNext();
    T next();
    default void remove() { throw new UnsupportedOperationException(); }
}
```

For-each работает с любым `Iterable`:
```java
for (String s : collection) { } 
// компилятор разворачивает в:
Iterator<String> it = collection.iterator();
while (it.hasNext()) { String s = it.next(); }
```

---

### 12. Как удалять объекты при итерации

```java
// ❌ ConcurrentModificationException
for (String s : list) {
    if (condition) list.remove(s);
}

// ✅ Через Iterator
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    if (condition(it.next())) it.remove();
}

// ✅ removeIf (Java 8+)
list.removeIf(s -> condition(s));

// ✅ Для Map
map.entrySet().removeIf(entry -> condition(entry.getValue()));
```

→ Подробнее: [[Java Core/Collections/Overview#Удаление элементов при итерации]]

---

### 13. Работа хэша в HashMap и HashSet

1. Вычисляется `hashCode()` ключа
2. Применяется дополнительное перемешивание: `hash = key.hashCode() ^ (key.hashCode() >>> 16)`
3. Индекс бакета: `index = hash & (capacity - 1)`
4. В бакете — цепочка или дерево, если выполнены пороги по длине и ёмкости
5. При совпавшем hash ключ сравнивают через `equals()`

HashSet = HashMap где значения = константа `PRESENT`.

→ Подробнее: [[Java Core/Collections/Overview#Как работает HashMap]]

---

### 14. IdentityHashMap, EnumMap и EnumSet

**IdentityHashMap:**
- Сравнивает ключи через `==` (ссылки), а не `equals()`
- Намеренно нарушает контракт Map
- Используется: сериализация, обход графа объектов без зацикливания

```java
IdentityHashMap<String, Integer> map = new IdentityHashMap<>();
String a = new String("key");
String b = new String("key");
map.put(a, 1);
map.put(b, 2);
map.size(); // 2 — разные ссылки!
```

**EnumMap:**
- Ключи — только элементы одного enum
- Внутри — массив, индекс = ordinal() enum-константы
- Очень быстрый и компактный

**EnumSet:**
- Элементы — только enum-константы
- Внутри — битовый вектор (long или long[])
- Все операции O(1), минимум памяти

---

### 15. SOLID и выбор интерфейса коллекции

Обычно метод принимает `List<T>` или `Map<K,V>`, если ему не нужны особенности конкретной реализации. Это пример зависимости от абстракции; остальные буквы SOLID не надо механически доказывать типами коллекций. См. [[Architecture/Design-Principles]].

---

### 16. YAGNI

**You Aren't Gonna Need It** — не добавляй то, что "может пригодиться".

```java
// ❌ YAGNI нарушение
class UserService {
    void exportToPdf() {} // "вдруг понадобится"
    void exportToExcel() {} // не нужно сейчас
}

// ✅ YAGNI соблюдение
class UserService {
    List<User> findAll() {} // только то, что нужно сейчас
}
```

---

### 17. Способы injection в Spring: constructor, setter, поле

→ [[Spring/Core/Overview#Способы Dependency Injection]]

---

### 18. Ингрессы (Ingress в Kubernetes)

Объект Kubernetes для управления входящим HTTP/HTTPS трафиком в кластер.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /users
        pathType: Prefix
        backend:
          service:
            name: user-service
            port:
              number: 8080
```

Требует **Ingress Controller** (nginx, traefik, istio) — без него не работает.

→ Подробнее: [[Infrastructure/Kubernetes-Ingress#Ingress (Ингресс)]]

---

### 19. Реестр схем данных (Schema Registry)

Централизованное хранилище схем сообщений (Avro, Protobuf, JSON Schema).

**Зачем:** при изменении схемы продюсером — консьюмер может сломаться. Schema Registry:
1. Хранит версии схем
2. Валидирует совместимость при регистрации новой версии
3. Продюсер/консьюмер договариваются через ID схемы

```
Producer → [schema_id + data] → Kafka
Consumer → schema_id → Schema Registry → десериализует
```

Популярная реализация: **Confluent Schema Registry**.

---

### 20. jRPC (JSON-RPC)

Лёгкий RPC протокол. Запрос и ответ — JSON объекты.

```json
// Request
{"jsonrpc": "2.0", "method": "getUser", "params": {"id": 1}, "id": 1}

// Response
{"jsonrpc": "2.0", "result": {"name": "Alice"}, "id": 1}

// Error
{"jsonrpc": "2.0", "error": {"code": -32600, "message": "Invalid Request"}, "id": 1}
```

**vs REST:** JSON-RPC описывает вызов метода и формат сообщения независимо от конкретного транспорта; через HTTP его часто передают POST, но это не требование «всегда только POST». HTTP-статус может отражать транспортный результат, а поле `error` — ошибку RPC.

---

### 21. Профилирование запросов и перцентили

**Перцентиль (p50, p95, p99):**
- p50 = медиана: половина запросов быстрее этого времени
- p95 = 95% запросов быстрее
- p99 = "долгий хвост": только 1% медленнее

**Почему не среднее:** среднее скрывает выбросы.

```
1000 запросов: 980 по 10ms + 20 по 5000ms
Average = (980*10 + 20*5000) / 1000 = 109.8ms
p99 ≈ 5000ms — показывает хвост; точный расчёт зависит от определения перцентиля
```

**Инструменты профилирования:** Micrometer + Prometheus + Grafana, JProfiler, async-profiler.

---

### 22. Горизонтальное масштабирование

→ [[Architecture/Distributed-Systems#Горизонтальное масштабирование]]

---

### 23. Симметричное / асимметричное шифрование

→ [[Architecture/Distributed-Systems#Симметричное vs Асимметричное шифрование]]
