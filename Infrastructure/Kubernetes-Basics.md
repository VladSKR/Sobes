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

## Two-Minute Drill

- Pod запускает контейнеры, Deployment поддерживает желаемое число pod, Service даёт стабильный адрес для них.
- Readiness исключает pod из новых запросов; liveness инициирует перезапуск при сбое.
- Масштабирование pod не увеличивает автоматически ёмкость БД и downstream.

**Проверь себя.** Что проверять, если Pod запущен, а Service не посылает ему запросы?

Совпадение selector и меток, readiness и наличие адреса среди EndpointSlices.

Источник: [Kubernetes concepts](https://kubernetes.io/docs/concepts/).
