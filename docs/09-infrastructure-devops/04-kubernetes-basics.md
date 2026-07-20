---
sidebar_position: 4
title: "Kubernetes-базис: pod, deployment, service, ingress, configmap/secret, HPA"
---

# Kubernetes-базис: pod, deployment, service, ingress, configmap/secret, HPA

> **TL;DR:** Каждый объект решает свою отдельную задачу в цепочке "как код превращается в работающий, доступный извне, самовосстанавливающийся сервис": Pod — единица запуска, Deployment — управление жизненным циклом пода (сколько реплик, как обновлять), Service — стабильный сетевой адрес поверх меняющихся подов, Ingress — маршрутизация HTTP снаружи кластера, ConfigMap/Secret — конфигурация отдельно от образа, HPA — автоматическое изменение числа реплик под нагрузкой.

## Pod — минимальная единица развёртывания

```yaml
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: api
      image: myapp:1.2.3
      ports: [{ containerPort: 3000 }]
```

Под — один или несколько контейнеров, гарантированно запланированных на **одном** узле, разделяющих сетевое пространство (один IP на под, контейнеры внутри общаются через `localhost`) и опционально volume'ы. Поды сами по себе **эфемерны** — Kubernetes не пытается "вылечить" упавший под, он просто создаёт новый с новым IP взамен; напрямую поды почти никогда не создают вручную в проде — ими управляет Deployment.

## Deployment — управление жизненным циклом реплик

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  replicas: 3
  selector: { matchLabels: { app: myapp } }
  template:
    metadata: { labels: { app: myapp } }
    spec:
      containers: [{ name: api, image: myapp:1.2.3 }]
  strategy:
    type: RollingUpdate
    rollingUpdate: { maxSurge: 1, maxUnavailable: 0 }
```

Deployment описывает **желаемое состояние** (3 реплики образа `myapp:1.2.3`) и постоянно приводит фактическое состояние кластера к этому желаемому через контроллер: если под упал — создаёт новый взамен; если изменить `image` в манифесте — выполняет rolling update согласно `strategy` (подробнее о стратегиях деплоя — в [07-deployment-strategies.md](./07-deployment-strategies.md)). `maxUnavailable: 0` гарантирует, что при обновлении число доступных подов никогда не падает ниже желаемого — новый под должен стать `Ready` (см. [05-probes-liveness-readiness.md](./05-probes-liveness-readiness.md)), прежде чем старый будет остановлен.

## Service — стабильный адрес поверх меняющихся подов

```yaml
apiVersion: v1
kind: Service
spec:
  selector: { app: myapp }   # выбирает поды по label, НЕ по конкретным именам/IP
  ports: [{ port: 80, targetPort: 3000 }]
  type: ClusterIP
```

Поды создаются, уничтожаются и пересоздаются с новыми IP постоянно (деплои, рестарты, автоскейлинг) — Service решает задачу "как достучаться до группы подов, не зная и не полагаясь на их текущие конкретные IP". Он выбирает актуальный набор подов через `selector` по label (не хардкодит список адресов) и даёт единый стабильный виртуальный IP/DNS-имя, автоматически обновляя список реальных endpoint'ов (это и есть server-side service discovery на практике, см. [06-architecture-patterns/12-api-gateway-bff.md](../06-architecture-patterns/12-api-gateway-bff.md)). Типы: `ClusterIP` (доступен только внутри кластера — стандарт для внутренней межсервисной коммуникации), `NodePort` (открывает порт на каждом узле — простой, но редко используемый напрямую в проде способ внешнего доступа), `LoadBalancer` (провижионит внешний облачный балансировщик — типично для входной точки кластера в облаке).

## Ingress — HTTP-маршрутизация снаружи

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
spec:
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /orders
            pathType: Prefix
            backend: { service: { name: orders-service, port: { number: 80 } } }
          - path: /users
            pathType: Prefix
            backend: { service: { name: users-service, port: { number: 80 } } }
```

Один внешний вход (обычно один облачный `LoadBalancer`) распределяет входящий HTTP-трафик по разным внутренним `Service` на основе хоста/пути — то же назначение, что у API Gateway (см. [06-architecture-patterns/12-api-gateway-bff.md](../06-architecture-patterns/12-api-gateway-bff.md)), реализованное как декларативный Kubernetes-объект, обрабатываемый Ingress-контроллером (nginx-ingress, Traefik и т.п. — сам Ingress-манифест — только правила, контроллер — то, что их реально исполняет). Экономит облачные балансировщики (не нужен отдельный `LoadBalancer` на каждый сервис) и даёт единую точку для TLS-терминации, path-based/host-based роутинга.

## ConfigMap и Secret — конфигурация отдельно от образа

```yaml
apiVersion: v1
kind: ConfigMap
data: { LOG_LEVEL: "info", FEATURE_FLAG_X: "true" }
---
apiVersion: v1
kind: Secret
type: Opaque
data: { DATABASE_PASSWORD: cGFzc3dvcmQ= }   # base64, НЕ шифрование
```

Реализация принципа 12-factor "конфигурация отдельно от кода" (см. [13-twelve-factor-app.md](./13-twelve-factor-app.md)) на уровне Kubernetes — один и тот же образ переиспользуется между окружениями (dev/staging/prod), конфигурация подставляется через переменные окружения/volume-маунты из ConfigMap/Secret, не пересобирая образ под каждое окружение. Важная деталь для собеседования — `Secret` **не шифрован** сам по себе, значение просто в base64 (тривиально декодируется) — реальная защита секретов через встроенный механизм требует включения encryption at rest на уровне etcd (хранилища состояния кластера) или использования внешнего vault-решения (см. [07-security/14-secrets-management.md](../07-security/14-secrets-management.md)) — сам факт, что это `Secret`, а не `ConfigMap`, только про то, что объект **обрабатывается** немного иначе (не логируется по умолчанию, монтируется в tmpfs), а не про шифрование содержимого.

## HPA (Horizontal Pod Autoscaler)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: myapp }
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource: { name: cpu, target: { type: Utilization, averageUtilization: 70 } }
```

Автоматически меняет число реплик Deployment в заданных границах на основе метрик (CPU-утилизация по умолчанию, но можно и по памяти или произвольным кастомным метрикам через adapter — например, длина очереди BullMQ, см. [08-performance-reliability/09-queues-bullmq.md](../08-performance-reliability/09-queues-bullmq.md), или event loop lag, см. [08-performance-reliability/06-cpu-bound-event-loop-lag.md](../08-performance-reliability/06-cpu-bound-event-loop-lag.md)). Работает только если приложение **stateless** (см. [08-performance-reliability/04-scaling-load-balancing.md](../08-performance-reliability/04-scaling-load-balancing.md)) — новые реплики, создаваемые HPA, должны быть полностью взаимозаменяемы с существующими без какой-либо координации/переноса состояния.

## Что спрашивают на собеседовании

1. **В чём разница между Pod и Deployment?** — Pod — единица запуска (один или несколько контейнеров на одном узле), сам по себе эфемерен и не самовосстанавливается; Deployment управляет желаемым числом реплик подов и стратегией их обновления, автоматически пересоздавая упавшие.
2. **Зачем нужен Service, если можно обращаться к подам напрямую по их IP?** — IP подов меняются при каждом пересоздании (деплой, рестарт, автоскейлинг) — Service даёт стабильный адрес поверх текущего набора подов, автоматически обновляемого по label-селектору.
3. **Чем Ingress отличается от Service типа LoadBalancer?** — Service/LoadBalancer — один облачный балансировщик на один сервис; Ingress — единая точка входа с маршрутизацией по хосту/пути на множество внутренних сервисов, экономит облачные балансировщики и централизует TLS-терминацию.
4. **Secret в Kubernetes зашифрован?** — нет по умолчанию, значение просто в base64 — тривиально декодируется; реальная защита требует encryption at rest на уровне etcd или внешнего vault-решения; `Secret` vs `ConfigMap` — это про обработку объекта (не логируется, монтируется в tmpfs), не про шифрование содержимого.
5. **От чего зависит, сможет ли HPA эффективно масштабировать приложение?** — приложение должно быть stateless — новые реплики, поднятые HPA, обязаны быть полностью взаимозаменяемы с уже существующими без переноса состояния между ними.

## Ссылки

- [Kubernetes — Pods](https://kubernetes.io/docs/concepts/workloads/pods/)
- [Kubernetes — Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Kubernetes — Service](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Kubernetes — Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [Kubernetes — Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- [Kubernetes — Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
