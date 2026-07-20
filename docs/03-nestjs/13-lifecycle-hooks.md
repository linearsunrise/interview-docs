---
sidebar_position: 13
title: Lifecycle hooks
---

# Lifecycle hooks

> **TL;DR:** Хуки жизненного цикла приложения (не запроса!). Порядок старта: `onModuleInit` → `onApplicationBootstrap` → listen. Порядок остановки: `onModuleDestroy` → `beforeApplicationShutdown` → `onApplicationShutdown`. Shutdown-хуки **не работают** без `app.enableShutdownHooks()`.

## Таблица

| Хук | Когда | Типовое применение |
|---|---|---|
| `onModuleInit` | зависимости модуля отрезолвлены | коннект к БД, прогрев кэша |
| `onApplicationBootstrap` | все модули инициализированы | запуск планировщиков, подписки |
| `onModuleDestroy` | получен сигнал завершения | отписки |
| `beforeApplicationShutdown` | после `onModuleDestroy`, до закрытия коннектов; получает сигнал (`'SIGTERM'`) | дождаться in-flight операций |
| `onApplicationShutdown` | последний; получает сигнал | закрыть коннекты БД/брокеров |

Все хуки могут быть `async` — Nest дождётся promise. Порядок в пределах одного хука: сначала провайдеры-зависимости, потом зависящие от них (граф DI).

## Graceful shutdown (главный практический кейс)

```ts
const app = await NestFactory.create(AppModule);
app.enableShutdownHooks(); // подписка на SIGTERM/SIGINT — без этого хуки остановки не вызовутся
```

Сценарий для Kubernetes: под получает SIGTERM → нужно перестать принимать новые запросы, дождаться текущих, закрыть коннекты, выйти. Без graceful shutdown при каждом деплое часть запросов обрывается 502-ми.

Нюансы:
- `enableShutdownHooks` не включён по умолчанию, потому что подписка на сигналы стоит памяти (актуально при многих инстансах в одном процессе, например в тестах).
- health checks (`@nestjs/terminus`) + readiness probe должны начать отдавать fail до завершения, чтобы балансер вывел под из ротации.
- REQUEST-scoped провайдеры хуков не имеют (точнее, их вызов не гарантирован) — они живут вне управляемого жизненного цикла приложения.

## Что спрашивают на собеседовании

1. **`onModuleInit` vs `onApplicationBootstrap`?** — первый на уровне модуля (его зависимости готовы), второй — когда готово всё приложение; кросс-модульные вещи (планировщики) — во второй.
2. **Почему shutdown-хуки не сработали?** — забыт `enableShutdownHooks()`; либо процесс убит SIGKILL (его перехватить нельзя).
3. **Как сделать graceful shutdown в k8s?** — enableShutdownHooks + закрытие коннектов в `onApplicationShutdown` + readiness probe + `terminationGracePeriodSeconds` больше максимального времени запроса.
4. **Где открывать коннект к БД — в конструкторе или в `onModuleInit`?** — в хуке: конструктор должен быть дешёвым и синхронным, DI не ждёт async-конструкторов.

## Ссылки

- [Lifecycle events — официальная документация](https://docs.nestjs.com/fundamentals/lifecycle-events) — там же полная диаграмма последовательности
- [Health checks (Terminus)](https://docs.nestjs.com/recipes/terminus)
- [Kubernetes: Pod termination](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination) — что происходит при SIGTERM
- Исходники: [`packages/core/hooks`](https://github.com/nestjs/nest/tree/master/packages/core/hooks) — по файлу на каждый хук, видно порядок вызова
