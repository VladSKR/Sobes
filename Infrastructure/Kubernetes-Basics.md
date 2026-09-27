# Kubernetes: что должен различать backend-разработчик

`Pod` — единица запуска контейнеров, часто эфемерная. `Deployment` управляет желаемым числом replicas и rolling update через ReplicaSet. `Service` даёт стабильную точку доступа к набору pod; `Ingress` и controller направляют внешний HTTP-трафик к Service, см. [[Infrastructure/Kubernetes-Ingress]].

| Проверка | Вопрос |
|---|---|
| `startupProbe` | Успело ли приложение запуститься? |
| `readinessProbe` | Можно ли посылать этому pod новый трафик? |
| `livenessProbe` | Надо ли перезапустить контейнер? |

При ошибке readiness pod перестаёт получать **новый** трафик от Service, но не обязан перезапускаться. Ошибка liveness может инициировать restart. `requests` помогают планировщику выделить ресурсы, `limits` ограничивают их использование; слишком низкий memory limit приводит к OOMKilled, а CPU limit может вызывать throttling. При rollout учитывай graceful shutdown, in-flight запросы и миграции БД.

Для ответа на вопрос «если пришло 2 млн запросов»: `replicas` сами по себе не создают бесконечные БД-соединения и не защищают downstream; нужны capacity, HPA/автомасштабирование с метриками, bounded work и отказ при перегрузке. По тесту отличия Pod/Deployment и probes стоило повторить; остальная инфраструктура не была столь подробно проверена.

Источник: [Kubernetes concepts](https://kubernetes.io/docs/concepts/).
