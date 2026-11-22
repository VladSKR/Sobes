# Материалы Questions.docx в учебнике

В этой карте можно найти тему по названию раздела Word или по имени извлечённого изображения. Изображения прочитаны и перенесены в главы в виде объяснений, примеров, схем и Markdown-таблиц. Номер изображения соответствует имени `imageN` внутри DOCX. Таблицы из скриншотов доступны как текст.

## Разделы документа

| Темы в Questions.docx | Главы |
|---|---|
| Вопросы перед оглавлением: Ingress, запуск Java, JIT, связывание, рефлексия, дженерики, коллекции, исключения, Spring Gateway, способы DI, протоколы, backpressure, SQL-профилирование и масштабирование | [Быстрые вопросы](Quick-Questions.md), [оглавление](../00_INDEX.md) |
| JDBC: драйвер, URL, Statement, ResultSet, исключения, транзакции и batch | [JDBC](../Spring/Data/JDBC.md) |
| Базы данных: NoSQL, индексы, WAL, ACID, изоляция, блокировки, репликация, deadlock, EXPLAIN, партиционирование и DELETE/TRUNCATE | [Базы данных](../Databases/Overview.md), [SQL и JOIN](../Databases/PostgreSQL-SQL-Fundamentals.md) |
| Saga, Event streaming и CAP | [Распределённые системы](../Architecture/Distributed-Systems.md), [Kafka](../Messaging/Kafka.md) |
| Exceptions, ООП, PECS, Object, equals, коллекции | [Исключения](../Java%20Core/Language/Exceptions.md), [ООП и Object](../Java%20Core/Language/OOP.md), [дженерики](../Java%20Core/Language/Generics.md), [коллекции](../Java%20Core/Collections/Overview.md) |
| Стек, куча и Garbage Collectors | [Память JVM](../Java%20Core/JVM/Memory-and-GC-Roots.md), [сборка мусора](../Java%20Core/JVM/Garbage-Collectors.md) |
| Архитектура: OSI, WebSocket, Circuit Breaker и SOLID | [Сеть и потоки данных](../API/Network-and-Streaming.md), [распределённые системы](../Architecture/Distributed-Systems.md), [принципы проектирования](../Architecture/Design-Principles.md) |
| SOAP, RESTful, HATEOAS, идемпотентность, PUT/PATCH и RSocket | [REST и протоколы](../API/REST-and-Protocols.md), [сеть и потоки данных](../API/Network-and-Streaming.md) |
| Пирамида тестирования, JDBC/JPA/Hibernate, Maven/Gradle | [Уровни тестирования](../Testing/Testing-Pyramid.md), [JPA и Hibernate](../Spring/Data/Overview.md), [управление зависимостями](../Tooling/Dependency-Management.md) |
| Spring: скоупы, lifecycle, циклы, Boot, стартер и прокси | [Контейнер Spring](../Spring/Core/Overview.md), [Spring Boot](../Spring/Boot/Overview.md) |
| Kafka, семантика доставки и аннотации | [Kafka](../Messaging/Kafka.md), [аннотации Java](../Java%20Core/Language/Annotations.md) |
| Многопоточность, volatile, Runnable/Callable, Future и CompletableFuture | [Потоки](../Java%20Core/Concurrency/00_Overview.md), [JMM](../Java%20Core/Concurrency/JMM-Happens-Before.md), [Executor и Future](../Java%20Core/Concurrency/Executors-CompletableFuture.md) |

В списке вопросов до оглавления также был [видеоматериал](https://youtu.be/_0c9Fd9FacU).

## Изображения по темам

| № изображения | Что на нём изображено | Где читать |
|---|---|---|
| 13, 15, 16, 24, 31, 40, 43, 48 | Запрос к Derby, структура JDBC URL, JAR и service provider драйвера, ошибка при отсутствии подходящего драйвера | [Драйвер и соединение](../Spring/Data/JDBC.md#драйвер-и-соединение) |
| 28, 32, 46, 47, 65 | Таблицы методов execute, чтение результата по строкам, обработка смешанных результатов и чтение двоичных данных. Закрытие JDBC-объектов в обратном порядке | [Выполнение SQL](../Spring/Data/JDBC.md#выполнение-sql-и-чтение-результата) |
| 9, 30, 45, 50, 54, 55 | Типы и режимы ResultSet, навигация, таблица JDBC-типов даты и времени | [Режимы и навигация ResultSet](../Spring/Data/JDBC.md#режимы-и-навигация-resultset) |
| 1, 11, 33, 57 | Частичный успех вставок, пакет SQL, разбор SQLException и rollback | [Ошибки, пакет и транзакция](../Spring/Data/JDBC.md#ошибки-пакет-и-граница-транзакции) |
| 17 | Четыре модели NoSQL | [Модели данных](../Databases/Overview.md#реляционные-модели-и-nosql) |
| 7, 36 | Упорядоченные ключи B-tree и перекрывающиеся пространственные прямоугольники R-tree | [От дерева к диапазону](../Databases/Overview.md#от-дерева-к-диапазону) |
| 38 | INNER, LEFT, RIGHT, FULL JOIN и выбор только несовпавших строк | [Как JOIN собирает строки](../Databases/PostgreSQL-SQL-Fundamentals.md#как-join-собирает-строки) |
| 60, 70 | Резерв кредита для заказа через хореографию и через CreateOrderSaga | [Saga](../Architecture/Distributed-Systems.md#saga) |
| 5, 20, 27, 29, 35, 37, 49 | Иерархия исключений, checked/unchecked, multicatch, вложенный try, распространение по стеку и ошибка при закрытии ресурса | [Исключения Java](../Java%20Core/Language/Exceptions.md) |
| 18, 66 | Таблица уровней OSI и сравнение обмена по HTTP с WebSocket | [Сеть и потоки данных](../API/Network-and-Streaming.md) |
| 52, 67 | Обнаружение сбоя, временное отключение и восстановление по аналогии с электрическим автоматом | [Circuit Breaker](../Architecture/Distributed-Systems.md#circuit-breaker) |
| 22, 34 | Методы Object, контракт equals | [ООП и Object](../Java%20Core/Language/OOP.md) |
| 56, 69 | Иерархия Collection и Map. HashSet, LinkedHashSet и TreeSet | [Интерфейсы коллекций](../Java%20Core/Collections/Overview.md) |
| 42 | Пять принципов SOLID | [Принципы проектирования](../Architecture/Design-Principles.md) |
| 2, 4, 8, 26, 44, 59, 61, 62, 64, 68 | Ограничения REST и uniform interface, гипермедийные действия в ответе по счёту и смысл идемпотентности | [REST и протоколы](../API/REST-and-Protocols.md) |
| 14 | Unit, service/system и E2E/GUI, их скорость и стоимость поддержки | [Уровни тестирования](../Testing/Testing-Pyramid.md) |
| 6, 19, 21, 23, 51, 53 | Модели RSocket, мультиплексирование, кадры, backpressure и возобновление | [RSocket](../API/Network-and-Streaming.md#rsocket) |
| 10, 25 | Пример volatile-флага и атомарность чтения или записи 64-битного volatile-значения | [Java Memory Model](../Java%20Core/Concurrency/JMM-Happens-Before.md) |
| 39, 41 | Maven dependencies и dependencyManagement, наследование версии и управление транзитивными зависимостями | [Управление зависимостями](../Tooling/Dependency-Management.md) |
| 63 | Heap, стеки потоков, Metaspace и Code Cache | [Память JVM](../Java%20Core/JVM/Memory-and-GC-Roots.md) |
| 3, 58 | Mark, sweep, compaction и схема Eden/S0/S1/Old | [Сборка мусора](../Java%20Core/JVM/Garbage-Collectors.md) |
| 12 | Жизненный цикл Spring-бина, aware-интерфейсы, post-processors и callbacks | [Жизненный цикл бина](../Spring/Core/Overview.md#жизненный-цикл-бина) |

## Таблицы Word

| Исходная таблица | Перенос |
|---|---|
| Операторы сравнения для B-tree | [Таблица типов индексов PostgreSQL](../Databases/Overview.md#типы-индексов-postgresql) включает все пять операторов и поясняет, как связаны диапазоны и выбор индекса |
| Serial, Parallel, G1 и ZGC | [Сравнение сборщиков и фаз](../Java%20Core/JVM/Garbage-Collectors.md#сравнение-сборщиков-и-фаз) включает все четыре варианта, флаги, паузы и конкурентные фазы. Shenandoah и CMS добавлены для сопоставления |
