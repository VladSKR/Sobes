# JVM — Внутреннее устройство

#java #jvm #gc #core

---

## Структура памяти JVM

```
Native Memory (вся системная память)
├── Heap (куча) [-Xms / -Xmx]
│   ├── Young Generation
│   │   ├── Eden
│   │   ├── Survivor S0
│   │   └── Survivor S1
│   └── Old (Tenured) Generation
├── Stack (стек) — свой на каждый поток
├── Metaspace [-XX:MaxMetaspaceSize] — метаданные классов, static
└── CodeCache — JIT-скомпилированный нативный код
```

**PermGen** — был до Java 7. С Java 8 заменён на Metaspace (в native memory).

---

## JIT Компилятор

**JIT (Just-In-Time)** — компилирует часто исполняемый байткод в нативный машинный код.

### Как работает
1. JVM начинает интерпретировать байткод
2. Профилировщик отслеживает "горячие" методы (Hot Spots)
3. JIT компилирует горячие методы в нативный код и кэширует в CodeCache
4. Последующие вызовы — выполняются нативно (быстро)

### Уровни компиляции (Tiered Compilation)
| Уровень | Компилятор | Оптимизация |
|---|---|---|
| 0 | Интерпретатор | Нет |
| 1-3 | C1 (client) | Базовая |
| 4 | C2 (server) | Агрессивная |

---

## Garbage Collection

### Алгоритм Mark & Sweep
1. **Mark** — сканирует все объекты, помечает живые (Stop-The-World)
2. **Sweep** — освобождает память мёртвых объектов
3. **Compact** — уплотнение: перемещает живые объекты в единый блок

### Поколенческая сборка мусора

```
1. Новые объекты → Eden
2. Eden заполнен → Minor GC
3. Живые из Eden → S0 (возраст +1)
4. Следующий Minor GC: Eden + S0 → S1 (возраст +1)
5. Следующий Minor GC: Eden + S1 → S0 (возраст +1)
6. Возраст > threshold → Old Generation
7. Old заполнен → Major GC (медленно — много живых объектов)
```

**В каждый момент один из S0/S1 всегда пуст.**

---

## Сборщики мусора

| GC           | Потоки            | Паузы                   | Когда использовать                        | Флаг                 |
| ------------ | ----------------- | ----------------------- | ----------------------------------------- | -------------------- |
| **Serial**   | 1                 | Stop-The-World          | Однопоточные, малые данные                | `-XX:+UseSerialGC`   |
| **Parallel** | Несколько         | Stop-The-World          | Пиковая производительность, паузы > 1с OK | `-XX:+UseParallelGC` |
| **G1**       | Параллельно + STW | Короткие, предсказуемые | Паузы < 1с, большие кучи                  | `-XX:+UseG1GC`       |
| **ZGC**      | Параллельно       | < 10ms                  | Минимальная латентность                   | `-XX:+UseZGC`        |

### G1 Garbage Collector

**Основная идея:** делит кучу на множество небольших **регионов** (обычно 1-32 MB).

Регионы могут быть: Eden, Survivor, Old, Humongous (для крупных объектов).

**Фазы работы G1:**
1. **Initial Mark** (STW) — пометить корневые объекты
2. **Concurrent Mark** — обойти граф объектов параллельно с приложением
3. **Final Mark** (STW) — завершить маркировку, обработать изменения
4. **Cleanup / Evacuation** (STW) — собрать мусор из самых "мусорных" регионов

**Garbage-First** = сначала собирает регионы с наибольшим количеством мусора.

---

## Запуск программы из командной строки

```bash
# Компиляция
javac Main.java

# Запуск
java Main
java -cp ./classes Main
java -jar app.jar

# Параметры JVM
java -Xms256m -Xmx1g -XX:+UseG1GC -jar app.jar

# Параметры приложения
java -jar app.jar --server.port=8080

# Просмотр флагов JVM
java -XX:+PrintFlagsFinal -version
```

---

## Аннотации — Retention Policy

```java
@Retention(RetentionPolicy.SOURCE)   // удаляется компилятором (@Override)
@Retention(RetentionPolicy.CLASS)    // в байткоде, недоступна через рефлексию (default)
@Retention(RetentionPolicy.RUNTIME)  // доступна через рефлексию (@Transactional, @Autowired)
```

---

## Ingress (Ингресс)

В контексте Kubernetes — **Ingress** это объект, управляющий внешним HTTP/HTTPS доступом к сервисам кластера.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
```

- **Ingress Controller** — реализация (nginx, traefik, istio)
- Ingress без контроллера не работает
- Может терминировать TLS
