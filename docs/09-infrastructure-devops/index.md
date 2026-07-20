---
sidebar_position: 9
title: Инфраструктура и Observability
---

# Инфраструктура, DevOps, Observability

## Подтемы

1. [Docker: слои и кеш, multi-stage build](./01-docker-multistage.md) — порядок COPY, `.dockerignore`, non-root user
2. [Образ: alpine/distroless, размер и безопасность](./02-docker-image-size-security.md) — `npm ci --omit=dev`, компромиссы базовых образов
3. [Сигналы в контейнере: PID 1, tini](./03-container-signals-pid1.md) — почему `npm start` глотает SIGTERM
4. [Kubernetes-базис](./04-kubernetes-basics.md) — pod, deployment, service, ingress, configmap/secret, HPA
5. [Probes: liveness vs readiness vs startup](./05-probes-liveness-readiness.md) — каскадные рестарты при БД в liveness
6. [Requests/limits, OOMKilled](./06-requests-limits-oomkilled.md) — throttling vs SIGKILL, `--max-old-space-size`
7. [Деплой: rolling, blue-green, canary; feature flags](./07-deployment-strategies.md) — trade-offs стратегий
8. [CI/CD: стадии, кеширование зависимостей](./08-cicd-pipeline.md) — fail fast, порядок по стоимости
9. [Логи: structured JSON, correlation id](./09-structured-logging.md) — pino, AsyncLocalStorage, что нельзя логировать
10. [Метрики: Prometheus, RED/USE](./10-metrics-prometheus.md) — pull-модель, гистограммы вместо среднего
11. [Трейсинг: OpenTelemetry, propagation](./11-distributed-tracing-otel.md) — traceparent, распространение через очереди
12. [Алертинг: SLO burn rate, error budget](./12-alerting-slo-error-budget.md) — симптомы vs причины, многоуровневые окна
13. [12-factor app](./13-twelve-factor-app.md) — своими словами, с осознанными отступлениями
14. [Миграции БД в деплое: expand-contract](./14-db-migrations-deploy.md) — совместимость версий при rolling update

## Чеклист знаний

- [ ] Docker: слои и кеш, multi-stage build для Node (deps → build → runtime), .dockerignore, non-root user
- [ ] Образ: alpine/distroless, `npm ci --omit=dev`, размер и безопасность
- [ ] Сигналы в контейнере: PID 1, почему `npm start` глотает SIGTERM (tini / прямой node)
- [ ] Kubernetes-базис: pod, deployment, service, ingress, configmap/secret, HPA
- [ ] Probes: liveness vs readiness vs startup — что в каждой проверять и типовые ошибки
- [ ] Requests/limits, OOMKilled и связь с `--max-old-space-size`
- [ ] Деплой: rolling, blue-green, canary; feature flags
- [ ] CI/CD: стадии (lint, test, build, scan), кеширование зависимостей
- [ ] Логи: structured JSON (pino), уровни, correlation/request id, что нельзя логировать
- [ ] Метрики: Prometheus, RED (rate, errors, duration) / USE, гистограммы для latency
- [ ] Трейсинг: OpenTelemetry, распространение контекста (traceparent) между сервисами и через очереди
- [ ] Алертинг: на симптомы (SLO burn rate), а не на причины; error budget
- [ ] 12-factor app — своими словами
- [ ] Миграции БД в деплое: expand-contract, совместимость версий

## Вопросы с собеседований

1. Напишите (устно) Dockerfile для Nest-приложения production-уровня — что и почему.
2. Liveness vs readiness — что будет, если в liveness проверять коннект к БД? (каскадные рестарты при падении БД)
3. Под убивается OOMKilled — как диагностировать? Как связаны limits и heap size Node?
4. Приложение в k8s не завершает соединения при деплое, клиенты ловят 502 — где искать? (SIGTERM → PID 1, preStop, graceful shutdown, readiness)
5. Как выкатить миграцию с переименованием колонки без даунтайма? (expand-contract: добавить → двойная запись → перенос → переключение чтения → удалить)
6. Canary vs blue-green — trade-offs, что нужно для canary (метрики, автооткат).
7. Как устроите логирование в микросервисах, чтобы по запросу пользователя собрать всю цепочку? (request id + трейсинг, propagation через заголовки и сообщения)
8. Какие метрики снимаете с Node-сервиса в первую очередь? (RED + event loop lag, heap, пул БД)
9. На что алертить, чтобы не тонуть в шуме?
10. Расскажите 12-factor и где вы отступали от него осознанно.

## Ключевые тезисы

**Multi-stage Dockerfile:** stage 1 — `npm ci` всех deps + build; stage 2 — только prod-deps + dist; итог: маленький образ без dev-зависимостей и исходников. Плюс non-root user и корректная обработка сигналов (node напрямую или tini).

**Probes:** readiness — «могу ли принимать трафик» (зависимости можно проверять) → под убирается из балансировки без рестарта. Liveness — «жив ли процесс» (только внутреннее состояние: deadlock, залипший event loop) → рестарт. Внешние зависимости в liveness = каскадный рестарт всего флота при мигании БД.

**Expand-contract:** каждая выкатка должна быть совместима и со старой, и с новой версией кода, потому что во время rolling-деплоя они работают одновременно.

## Красные флаги

- Один stage в Dockerfile, dev-зависимости в проде
- Проверка БД в liveness
- `console.log` вместо структурированных логов в проде
- Миграция «переименовал колонку и задеплоил»

## Практика

- Написать production-Dockerfile для своего пет-проекта, сравнить размер до/после multi-stage
- Добавить в Nest: pino с request-id (AsyncLocalStorage), /metrics с prom-client, health-эндпоинты (Terminus)
- Локально в minikube/kind: deployment с probes, убедиться в zero-downtime rolling update под нагрузкой
