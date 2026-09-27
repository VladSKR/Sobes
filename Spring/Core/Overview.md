# Spring Framework

#spring #java #di #ioc

---

## IoC и Dependency Injection

### Inversion of Control (IoC)
Управление созданием и жизненным циклом объектов передаётся контейнеру (Spring).

**Без IoC:**
```java
class OrderService {
    PaymentService payment = new PaymentService(); // сам создаёт зависимость
}
```

**С IoC:**
```java
@Service
class OrderService {
    private final PaymentService payment; // контейнер создаёт и внедряет
    
    public OrderService(PaymentService payment) {
        this.payment = payment;
    }
}
```

---

## Способы Dependency Injection

### 1. Constructor Injection (рекомендуется)
```java
@Service
public class OrderService {
    private final PaymentService payment;
    private final NotificationService notification;

    @Autowired // с одним конструктором — необязательна
    public OrderService(PaymentService payment, NotificationService notification) {
        this.payment = payment;
        this.notification = notification;
    }
}
```
✅ Поля final, гарантированная инициализация, легко тестировать

### 2. Setter Injection
```java
@Service
public class OrderService {
    private PaymentService payment;

    @Autowired
    public void setPayment(PaymentService payment) {
        this.payment = payment;
    }
}
```
✅ Опциональные зависимости, решение циклических зависимостей

### 3. Field Injection (не рекомендуется)
```java
@Service
public class OrderService {
    @Autowired
    private PaymentService payment; // через рефлексию
}
```
❌ Нельзя сделать final, сложнее тестировать

---

## Скоупы бинов

| Скоуп | Создание | Когда |
|---|---|---|
| **singleton** (default) | Один раз на контейнер | Большинство сервисов |
| **prototype** | Новый при каждом getBean() | Stateful объекты |
| **request** | На каждый HTTP-запрос | Web-слой |
| **session** | На каждую HTTP-сессию | Пользовательские данные |
| **application** | На всё приложение | Один бин на ServletContext |
| **websocket** | На каждое WS-соединение | WebSocket хендлеры |

---

## Жизненный цикл бина
#Современный вариант

1. SpringApplication.run(...)  
2. Создаётся ApplicationContext  
3. Внутри context есть BeanFactory / BeanDefinitionRegistry  
4. Spring регистрирует первичные BeanDefinition из @SpringBootApplication class  
5. ConfigurationClassPostProcessor разбирает @Configuration, @ComponentScan, @Bean, @Import, @Conditional, auto-configuration  
6. Scanner находит @Component / @Service / @Repository / @Controller  
7. В BeanFactory оказывается набор BeanDefinition  
8. BeanFactoryPostProcessor-ы могут изменить BeanDefinition до создания объектов  
9. Регистрируются BeanPostProcessor-ы  
10. BeanFactory создаёт singleton beans  
11. Каждый bean проходит lifecycle: instantiate -> populate dependencies -> aware callbacks -> before init -> init -> after init  
12. Готовый singleton кладётся в singleton cache

```
1. Инстанцирование (конструктор)
2. Установка свойств (DI)
3. Aware-интерфейсы (BeanNameAware, BeanFactoryAware...)
4. BeanPostProcessor.postProcessBeforeInitialization()
5. Инициализация:
   a) @PostConstruct
   b) InitializingBean.afterPropertiesSet()
   c) init-method
6. BeanPostProcessor.postProcessAfterInitialization()
--- бин готов к использованию ---
7. Деинициализация:
   a) @PreDestroy
   b) DisposableBean.destroy()
   c) destroy-method
```

---

## Циклическая зависимость

**Проблема:**
```
BeanA → BeanB → BeanA (цикл)
```

Spring Boot 2.6+ — бросает `BeanCurrentlyInCreationException` по умолчанию.

**Решения:**

1. **Пересмотреть архитектуру** (лучший вариант)

2. **@Lazy** — отложить инициализацию одного из бинов:
```java
@Autowired
public BeanA(@Lazy BeanB beanB) {
    this.beanB = beanB;
}
```

3. **Setter injection** вместо конструктора:
```java
@Autowired
public void setBeanB(BeanB beanB) {
    this.beanB = beanB;
}
```

4. **Флаг** (не рекомендуется):
```yaml
spring:
  main:
    allow-circular-references: true
```

---

## Способы проксирования

| Метод                 | Механизм                   | Когда                   |
| --------------------- | -------------------------- | ----------------------- |
| **JDK Dynamic Proxy** | Реализует те же интерфейсы | Бин реализует интерфейс |
| **CGLib**             | Наследование от класса     | Бин без интерфейса      |

**CGLib ограничения:**
- Нельзя проксировать `final` классы
- Нельзя переопределить `final` методы
- Чуть хуже производительность

```java
// AOP / @Transactional работает через прокси
// Вызов метода внутри одного класса (self-invocation) — прокси НЕ работает
```

---








---

### Дополнение к созданию ApplicationContext

## Шаг 1 — Создание ApplicationContext

java

```java
ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);
```

`AnnotationConfigApplicationContext` внутри себя создаёт:

- `DefaultListableBeanFactory` — реестр всех BeanDefinition и сам IoC-контейнер
- `AnnotatedBeanDefinitionReader` — читает классы с аннотациями
- `ClassPathBeanDefinitionScanner` — сканирует пакеты
