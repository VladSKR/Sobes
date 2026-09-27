# Параллельные джобы между инстансами — Quartz Scheduler

## 2. Параллельные джобы между инстансами — Quartz Scheduler

### Проблема с @Scheduled + Shedlock

```
Shedlock — гарантирует что только ОДИН инстанс выполняет джобу
Это нужно чтобы не было дублирования
Но это означает — нет горизонтального масштабирования джоб
```

### Quartz Scheduler — решение

Quartz хранит джобы в БД и сам координирует между инстансами.

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-quartz</artifactId>
</dependency>
```

```yaml
spring:
  quartz:
    job-store-type: jdbc          # хранить джобы в БД
    jdbc:
      initialize-schema: always   # создать таблицы автоматически
    properties:
      org.quartz:
        jobStore:
          isClustered: true       # кластерный режим — главное для распределения
          clusterCheckinInterval: 5000
        scheduler:
          instanceId: AUTO        # уникальный ID каждого инстанса
```

```java
// Создание джобы
@Component
public class SmsJob implements Job {
    @Autowired
    private SmsService smsService;

    @Override
    public void execute(JobExecutionContext context) {
        smsService.sendPending();
    }
}

// Регистрация
@Configuration
public class QuartzConfig {
    @Bean
    public JobDetail smsJobDetail() {
        return JobBuilder.newJob(SmsJob.class)
            .withIdentity("smsJob")
            .storeDurably()
            .build();
    }

    @Bean
    public Trigger smsJobTrigger() {
        return TriggerBuilder.newTrigger()
            .forJob(smsJobDetail())
            .withSchedule(CronScheduleBuilder.cronSchedule("0/30 * * * * ?"))
            .build();
    }
}
```

### Quartz vs @Scheduled + Shedlock

| | @Scheduled + Shedlock | Quartz |
|---|---|---|
| Хранение джоб | в памяти | в БД |
| Кластеризация | ❌ один инстанс | ✅ распределение между инстансами |
| Retry при падении | ❌ | ✅ |
| История выполнения | ❌ | ✅ |
| Сложность | низкая | высокая |
| Когда использовать | простые джобы | критичные распределённые джобы |

### Как Quartz распределяет джобы

```
Инстанс 1, 2, 3 — все подключены к одной БД с таблицами Quartz
Каждый инстанс регулярно делает check-in в БД (heartbeat)
Когда наступает время джобы — один инстанс захватывает её через lock в БД
Остальные видят что джоба захвачена — не трогают
Если инстанс упал (нет heartbeat) — другой инстанс подхватывает его джобы
```

---
