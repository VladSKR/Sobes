# Kubernetes Ingress

## Ingress (Ингресс)

В контексте Kubernetes — **Ingress** это объект, управляющий внешним HTTP/HTTPS доступом к сервисам кластера.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
```

- **Ingress Controller** — реализация (nginx, traefik, istio)
- Ingress без контроллера не работает
- Может терминировать TLS

## Проверяем маршрут по слоям

Пусть `api.example.org/orders` должен вести в Service `orders` на порт 80, а контейнер слушает 8080. В Ingress указывают имя и порт Service, в Service — `targetPort: 8080`. Ingress Controller выбирает ресурс по `ingressClassName` и конфигурирует маршрутизацию. Контроллер может направлять запросы непосредственно на endpoints Service. Точное поведение зависит от его реализации.

Если запрос не доходит, проверь DNS и адрес контроллера, затем совпадение Host, path, pathType и ingress class. Дальше проверь Service selector и EndpointSlices: пустой список часто указывает на несовпавшие метки или неготовые Pod. TLS-секрет обслуживает сертификат для внешнего имени. Переписывание пути обычно задаётся особенностями контроллера и требует явной проверки.

[Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) · [Kubernetes Service](https://kubernetes.io/docs/concepts/services-networking/service/)

## Two-Minute Drill

- Ingress задаёт правила внешнего HTTP/HTTPS маршрута к Service внутри кластера.
- Нужен работающий Ingress Controller: один YAML-объект сам по себе трафик не направляет.

**Проверь себя.** DNS и TLS работают, но маршрут по префиксу не найден. Где искать причину?

В выбранном ingress class, совпадении Host и path и поведении pathType или rewrite.
