# Жизненный цикл Spring bean: ответы на уточнения

## За две минуты

- Сначала регистрируются BeanDefinition, затем создаются экземпляры с DI, callback-ами и BeanPostProcessor.
- `@PostConstruct` — часть инициализации экземпляра; `@PreDestroy` относится к управляемому уничтожению.
- Prototype внутри singleton не даёт новый экземпляр при каждом обращении без provider/lookup.

---

`SpringApplication.run` поднимает `ApplicationContext`. Во время `refresh()` сначала регистрируются BeanDefinition (описания), затем запускаются BeanFactoryPostProcessor: в частности, обработка конфигураций, сканирование `@Component`, методов `@Bean` и автонастроек. После этого регистрируются BeanPostProcessor, затем обычно создаются singleton bean.

Для одного bean: создание экземпляра → внедрение зависимостей → Aware callbacks → `postProcessBeforeInitialization` → `@PostConstruct` / `afterPropertiesSet` / init-method → `postProcessAfterInitialization`, где, например, возможна AOP-прокси. При закрытии контекста для управляемых singleton вызываются destroy callbacks, включая `@PreDestroy`. Точный порядок специальных процессоров и lazy beans требует уточнений; это схема ответа, а не обещание, что каждый bean создаётся в один момент.

| Вопрос | Краткий ответ |
|---|---|
| `@Component` vs `@Bean`? | Класс для component scan vs factory method в конфигурации; `@Bean` удобно для чужого класса или явной фабрики. |
| BeanFactory vs ApplicationContext? | Базовый контейнер и расширенный контекст с событиями, ресурсами и другими интеграциями. |
| Prototype уничтожается автоматически? | Контейнер создаёт и инициализирует; обычный полный destroy lifecycle после передачи клиенту не ведёт. |
| Prototype внутри singleton? | Внедрение при создании singleton даёт один конкретный экземпляр; нужен provider/lookup для новых. |
| Self-invocation и AOP? | Внутренний вызов проходит мимо внешнего прокси. |

В прошлых ответах цепочка часто обрывалась на «Spring создаёт и инжектит». Запоминай три этапа: **BeanDefinition → BeanFactoryPostProcessor/BeanPostProcessor → экземпляр и callback-и**. Подробнее об исходном `refresh()` — [[Spring/Boot/Internals]], про транзакционный прокси — [[Spring/Core/Transactions-and-JPA]].

Источник: [Spring Bean lifecycle](https://docs.spring.io/spring-framework/reference/core/beans/factory-nature.html).
