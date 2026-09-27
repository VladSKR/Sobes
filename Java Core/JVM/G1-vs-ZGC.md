# ZGC vs G1

## 4. ZGC vs G1

### G1 — Garbage First

**Архитектура:**
- Heap делится на регионы одинакового размера (1MB - 32MB)
- Регионы динамически назначаются как Eden, Survivor, Old, Humongous
- GC приоритизирует регионы с наибольшим количеством мусора (отсюда название Garbage First)

**Stop-the-World паузы в G1:**
```
Minor GC (Young):  10-200ms  — зависит от размера Young generation
Mixed GC:          50-500ms  — очищает Young + часть Old
Full GC:           секунды   — если concurrent marking не успевает
```

**Когда G1 — правильный выбор:**
- Heap 4GB - 32GB
- Паузы до 200ms допустимы
- Серверные приложения без жёстких latency требований

---

### ZGC — Z Garbage Collector

**Ключевое отличие:** почти вся работа выполняется **параллельно с приложением** (concurrent).

**Stop-the-World паузы в ZGC:**
```
Всегда < 1-10ms независимо от размера heap
Даже на heap 1TB — пауза те же миллисекунды
```

**Как ZGC это делает:**

Использует **цветные указатели** (colored pointers) — в самих битах указателя хранит метаданные о состоянии объекта. Это позволяет делать маркировку и перемещение объектов без остановки приложения.

```
Обычный указатель:  [адрес объекта]
ZGC указатель:      [метаданные 4 бита][адрес объекта]
                     ↑ finalizable, remapped, marked0, marked1
```

**Фазы ZGC:**
```
STW: Pause Mark Start     (~1ms)  — корневые объекты
Concurrent: Mark          (параллельно с приложением)
STW: Pause Mark End       (~1ms)  — финализация маркировки
Concurrent: Prepare Reloc (параллельно)
STW: Pause Relocate Start (~1ms)  — начало перемещения
Concurrent: Relocate      (параллельно)
```

---

### Сравнение

| | G1 | ZGC |
|---|---|---|
| Паузы | 10ms - 500ms | < 10ms всегда |
| Зависимость паузы от heap | Да, растёт | Нет |
| Throughput | Выше | Немного ниже (~15%) |
| Подходящий heap | 4-32GB | 8MB - 16TB |
| Java версия | Java 9+ (default) | Java 15+ (production ready) |

---

### Когда переходить на ZGC

- Heap > 32GB
- Latency критична: trading, real-time системы, крипто-биржи
- Паузы G1 вызывают таймауты в downstream сервисах
- SLA требует p99 latency < 100ms

**Включение:**
```bash
java -XX:+UseZGC -Xmx16g -jar app.jar

# ZGC с generational режимом (Java 21+, лучший throughput)
java -XX:+UseZGC -XX:+ZGenerational -Xmx16g -jar app.jar
```

---

### Для крипто-биржи

G1 с паузой 200ms = 200ms без обработки ордеров. На высоколиквидном рынке это потери.

ZGC — правильный выбор для такой системы. Стоит поднять вопрос о переходе.

---
