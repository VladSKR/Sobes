# Kubernetes: что должен различать backend-разработчик

`Pod` — единица запуска контейнеров, часто эфемерная. `Deployment` управляет желаемым числом replicas и rolling update через ReplicaSet. `Service` даёт стабильную точку доступа к набору pod; `Ingress` и controller направляют внешний HTTP-трафик к Service, см. [[Infrastructure/Kubernetes-Ingress]].

| Проверка | Вопрос |
|---|---|
| `startupProbe` | Успело ли приложение запуститься? |
| `readinessProbe` | Можно ли посылать этому pod новый трафик? |
| `livenessProbe` | Надо ли перезапустить контейнер? |

При ошибке readiness pod перестаёт получать **новый** трафик от Service, но не обязан перезапускаться. Ошибка liveness может инициировать restart. `requests` помогают планировщику выделить ресурсы, `limits` ограничивают их использование; слишком низкий memory limit приводит к OOMKilled, а CPU limit может вызывать throttling. При rollout учитывай graceful shutdown, in-flight запросы и миграции БД.

Для ответа на вопрос «если пришло 2 млн запросов»: `replicas` сами по себе не создают бесконечные БД-соединения и не защищают downstream; нужны capacity, HPA/автомасштабирование с метриками, bounded work и отказ при перегрузке. По тесту отличия Pod/Deployment и probes стоило повторить; остальная инфраструктура не была столь подробно проверена.

## Один релиз backend-сервиса

`Deployment` описывает желаемый шаблон Pod и количество реплик. Контроллер создаёт ReplicaSet, который поддерживает нужное число Pod. Планировщик выбирает узлы по требуемым ресурсам и ограничениям, kubelet запускает контейнеры. Когда Pod удаляется, ReplicaSet может создать другой с новым IP. `Service` выбирает Pod по меткам и даёт клиентам устойчивое имя.

При обновлении Deployment новый ReplicaSet постепенно заменяет старый. `readinessProbe` определяет готовность принимать трафик, `livenessProbe` может привести к перезапуску контейнера, а `startupProbe` защищает медленный старт от преждевременной проверки liveness. `requests` участвуют в размещении Pod. При достижении CPU limit возможен throttling, при превышении memory limit — завершение процесса. HPA меняет желаемое количество реплик по метрикам, а на готовность новых Pod влияет всё тот же цикл запуска.

[Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) · [Пробы](https://kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/) · [Путь внешнего запроса](Kubernetes-Ingress.md)

## Разбор интервью: requests, limits, throttling, eviction

```yaml
resources:
  requests:
    cpu: 250m
    memory: 512Mi
  limits:
    cpu: 500m
    memory: 1024Mi
```

**Requests** — ресурсы, которые Scheduler учитывает при размещении Pod на Node по её **allocatable**: 250m = четверть логического CPU, 512Mi — требуемая для планирования память. Это **не физически выделенные 512Mi RAM**, не жёсткий предел потребления и не резервирование отдельного CPU. Если на Node по сумме requests недостаточно 512Mi, Pod туда не назначат даже при низкой фактической нагрузке.

**Limits** применяются Linux cgroups к работающему контейнеру. CPU limit 500m — квота CPU-времени (примерно 50 мс CPU на 100 мс периода при стандартной настройке); при попытке 900m происходит **throttling**, а **не restart**. Memory limit 1024Mi — верхняя граница учтённой памяти; при невозможности удовлетворить выделение памяти ядро может завершить процесс (контейнер с причиной `OOMKilled`). Превышение memory **request** 512Mi само по себе не ошибка: контейнер может занимать, например, 800Mi.

### Почему Pod может быть Evicted до достижения limit

Node имеет конечную память. Например, десять Pod с request 512Mi и limit 2Gi могут быть запланированы на 8Gi Node, если хватает прочих ресурсов, но когда фактический расход нескольких Pod растёт, возникает **MemoryPressure**. Kubelet пытается освободить ресурсы и при необходимости **вытесняет** (Evicts) Pod. При выборе важны превышение requests и приоритеты. Это не живой перенос памяти процесса между Node.

Если Pod управляется Deployment, ReplicaSet создаёт **новый Pod**, Scheduler ищет Node. Если не хватает allocatable ресурсов ни на одной Node, новый Pod **Pending** с `FailedScheduling: Insufficient memory`; прежний Evicted не оживает. При наличии корректно настроенного Cluster Autoscaler может добавиться Node. Отличай от **OOMKilled** при превышении cgroup limit: часто перезапускается контейнер **в том же Pod**, а не создаётся отдельный Pod.

`memory: 1024Mi` ограничивает контейнер, а JVM `-Xmx` — только Java heap. Кроме heap процесс использует direct memory (например, Netty), metaspace, thread stacks, JIT code cache и native allocations. Поэтому `-Xmx=900m` при limit 1024Mi может **ухудшить**, а не исправить OOMKilled. [Диагностика JVM](../Java%20Core/JVM/Memory-and-GC-Roots.md).

**Проверь себя.** Если Node почти свободна по фактическому CPU, но сумма requests уже близка к allocatable, назначит ли Scheduler Pod с большим CPU request? Что будет при достижении CPU limit во время работы?

Scheduler может отказать в размещении из-за недостатка заявленного CPU; уже работающий контейнер ограничивается CPU throttling, а не перезапускается.

Источники: [Kubernetes resource management](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/) · [Node pressure eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/).

## Two-Minute Drill

- **Собеседование:** прежде чем отвечать, фиксируй исходные условия (порядок, лимит, протокол или состояние) и последствия.

- Pod запускает контейнеры, Deployment поддерживает желаемое число pod, Service даёт стабильный адрес для них.
- Readiness исключает pod из новых запросов; liveness инициирует перезапуск при сбое.
- Масштабирование pod не увеличивает автоматически ёмкость БД и downstream.

**Проверь себя.** Что проверять, если Pod запущен, а Service не посылает ему запросы?

Совпадение selector и меток, readiness и наличие адреса среди EndpointSlices.

Источник: [Kubernetes concepts](https://kubernetes.io/docs/concepts/).
