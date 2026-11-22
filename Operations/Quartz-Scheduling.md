# Параллельные джобы между инстансами — Quartz Scheduler

## Когда достаточно @Scheduled с ShedLock

Два экземпляра сервиса могут запустить один и тот же метод `@Scheduled`. ShedLock использует общую блокировку на имя задачи. Если одна задача выполняется дольше `lockAtMostFor`, другой экземпляр может получить освободившуюся блокировку и тоже начать выполнение. Для частых независимых запусков с персистентным расписанием и координацией trigger подходит Quartz. Большую бизнес-операцию можно декомпозировать на независимые записи в обоих вариантах.

## Quartz и общий JobStore

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
      initialize-schema: never    # схему для production лучше вести миграцией
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

## Сравнение Quartz и ShedLock

| | @Scheduled + Shedlock | Quartz |
|---|---|---|
| Хранение джоб | в памяти | в БД |
| Кластеризация | Один запуск конкретной задачи с общим lock | Триггеры координируются между инстансами |
| Recovery после падения | Зависит от реализации задачи | Настраивается; не означает автоматический безопасный retry побочных эффектов |
| История выполнения | Добавляется отдельно | Нужен отдельный мониторинг; job store не полная история бизнес-результатов |
| Сложность | низкая | высокая |
| Когда использовать | простые джобы | критичные распределённые джобы |

## Как Quartz распределяет запуски

```
Инстанс 1, 2, 3 — все подключены к одной БД с таблицами Quartz
Каждый инстанс регулярно делает check-in в БД (heartbeat)
Когда наступает время джобы — один инстанс захватывает её через lock в БД
Остальные видят что джоба захвачена — не трогают
Если инстанс упал, кластер может восстановить выполнение согласно настройкам job recovery; внешние побочные эффекты требуют идемпотентности
```

---

## Опоздание, пересечение запусков и падение

`JobDetail` определяет исполняемую работу и её идентичность. `Trigger` задаёт время запуска. Если scheduler был недоступен в нужное время или очередь была занята, Quartz может считать trigger просроченным (misfire). Для cron отдельно выбери, запускать ли один раз сейчас или перейти к следующему запланированному времени. Recovery с `requestRecovery()` относится к уже исполнявшейся задаче при падении узла.

Когда обработка SMS длится 40 секунд при расписании раз в 30, `@DisallowConcurrentExecution` запрещает пересечение для того же JobKey. Два разных JobKey одного класса могут работать одновременно. Разделение пачки сообщений между работниками проектируется отдельно, например коротким атомарным захватом записей и сохранением ключа отправки.

[Quartz: jobs и recovery](https://www.quartz-scheduler.org/documentation/quartz-2.4.x/tutorials/tutorial-lesson-03.html)

## Two-Minute Drill

- Quartz с общим JDBC job store позволяет нескольким экземплярам согласованно брать trigger.
- Один запланированный запуск исполняет один узел; это не автоматическое параллельное деление одной большой джобы на части.
- После падения recovery настраивается, побочные эффекты всё равно должны быть идемпотентны.

**Проверь себя.** Два JobDetail одного класса с разными ключами могут пересекаться?

Да. DisallowConcurrentExecution действует на JobKey.
