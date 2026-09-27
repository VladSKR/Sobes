## BeanPostProcessor + Рефлексия

**Задача:** выполнить логику над полями бина во время его инициализации.

```java
@Component
public class LogAnnotationBeanPostProcessor implements BeanPostProcessor {

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        for (Field field : bean.getClass().getDeclaredFields()) {
            LogAnnotation annotation = field.getAnnotation(LogAnnotation.class);
            if (annotation != null) {
                field.setAccessible(true); // только после проверки аннотации!
                try {
                    System.out.println(field.getName() + " = " + field.get(bean));
                } catch (IllegalAccessException e) {
                    throw new RuntimeException(e);
                }
            }
        }
        return bean;
    }
}
```

**Почему `setAccessible` только после проверки аннотации:** BeanPostProcessor обрабатывает **все** бины, включая системные Spring/JMX. Вызов `setAccessible` на полях системных классов (например `com.sun.jmx.*`) бросает `InaccessibleObjectException` из-за Java Module System.

**Почему `postProcessAfterInitialization`, а не Before:** В `Before` бин ещё не инициализирован — поля, устанавливаемые через `@PostConstruct` или `InitializingBean`, могут быть null.

---

## Аннотация — хорошие практики

```java
@Target(ElementType.FIELD)   // ограничивает применение только полями
@Retention(RetentionPolicy.RUNTIME)
public @interface LogAnnotation {
    boolean masked() default false; // маскировка чувствительных данных
    String name() default "";       // кастомное имя в логе
}
```

**Зачем `@Target`:** без него аннотацию можно поставить на метод/класс — компилятор не ругнётся, но BeanPostProcessor её не найдёт (`getDeclaredFields()` не видит методы). `@Target` делает контракт явным.

---

## Как Spring Boot собирает BeanDefinition

Замена XML-ридера — несколько механизмов, все регистрируют `BeanDefinition` в `DefaultListableBeanFactory`.

### 1. ComponentScan → ClassPathBeanDefinitionScanner

Сканирует classpath, читает байткод через ASM (без загрузки классов в JVM), ищет `@Component` и производные (`@Service`, `@Repository`, `@Controller`). Создаёт `ScannedGenericBeanDefinition`.

### 2. ConfigurationClassParser → @Bean методы

Парсит `@Configuration`-классы. Каждый `@Bean`-метод → `ConfigurationClassBeanDefinition`. Также обрабатывает `@Import`, `@ImportResource`, `@PropertySource`.

### 3. AutoConfigurationImportSelector

Читает файл:

```
META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
```

Каждая строка — `@AutoConfiguration`-класс с условиями:

```java
@AutoConfiguration
@ConditionalOnClass(DataSource.class)
@ConditionalOnMissingBean(DataSource.class)
public class DataSourceAutoConfiguration { ... }
```

Условия позволяют автоконфигурации не вмешиваться, если бин определён вручную.

---

## AbstractApplicationContext.refresh()

Главный метод жизненного цикла. Выполняется один раз при старте.

|Шаг|Метод|Что происходит|
|---|---|---|
|1|`prepareRefresh()`|Флаг active, время старта, валидация properties|
|2|`obtainFreshBeanFactory()`|Возвращает `DefaultListableBeanFactory`|
|3|`prepareBeanFactory()`|ClassLoader, Aware-процессоры, встроенные бины (`environment` и др.)|
|4|`postProcessBeanFactory()`|Хук для подклассов (сервлетный контекст)|
|5|`invokeBeanFactoryPostProcessors()`|**Запуск `ConfigurationClassPostProcessor`** → все BeanDefinition зарегистрированы|
|6|`registerBeanPostProcessors()`|BeanPostProcessor-бины создаются раньше остальных|
|7|`initMessageSource()`|i18n, заглушка если не определён|
|8|`initApplicationEventMulticaster()`|Механизм событий (`@EventListener`)|
|9|`onRefresh()`|Старт embedded сервера (Tomcat/Jetty)|
|10|`registerListeners()`|Подписка `ApplicationListener`-бинов|
|11|`finishBeanFactoryInitialization()`|**Создание всех singleton-бинов**, вызов BeanPostProcessor|
|12|`finishRefresh()`|`ContextRefreshedEvent`, старт `Lifecycle`-бинов|

**Цепочка создания бина на шаге 11:**

```
BeanDefinition → instantiate → inject dependencies
→ BeanPostProcessor.before → init (@PostConstruct)
→ BeanPostProcessor.after → готовый бин
```

---

## Ручное обновление контекста

### context.refresh() повторно — опасно

Пересоздаёт все singleton-бины. Старые ссылки становятся невалидными. В Spring Boot попытается поднять сервер повторно → `InstanceAlreadyExistsException`.

### @RefreshScope + Spring Cloud — правильный способ

```java
@RefreshScope
@Component
public class MyService {
    @Value("${some.property}")
    private String property;
}
```

```bash
POST /actuator/refresh
```

Пересоздаются только `@RefreshScope`-бины.

### Программная регистрация нового бина

```java
DefaultListableBeanFactory factory =
    (DefaultListableBeanFactory) context.getBeanFactory();

GenericBeanDefinition bd = new GenericBeanDefinition();
bd.setBeanClass(MyService.class);
factory.registerBeanDefinition("myService", bd);
```

### Регистрация готового экземпляра

```java
factory.registerSingleton("myService", new MyService());
```

---

## Типичные ошибки

|Ошибка|Причина|
|---|---|
|`InaccessibleObjectException`|`setAccessible(true)` на полях системных классов без проверки аннотации|
|`InstanceAlreadyExistsException`|Двойной вызов `SpringApplication.run()` в main|
|Аннотация не работает на методе|Отсутствует `@Target(ElementType.FIELD)`|
