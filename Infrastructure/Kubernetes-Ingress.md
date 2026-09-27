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
