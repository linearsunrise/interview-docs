---
sidebar_position: 11
title: AsyncLocalStorage
---

# AsyncLocalStorage

> **TL;DR:** ALS — «thread-local для async»: контекст, который автоматически следует за цепочкой асинхронных вызовов. Решает задачу «request-id / текущий юзер / tenant в любом месте кода» без протаскивания параметра через все сигнатуры и без REQUEST scope в Nest.

## Как пользоваться

```js
import { AsyncLocalStorage } from 'node:async_hooks';
const als = new AsyncLocalStorage();

// middleware: оборачиваем обработку запроса в контекст
app.use((req, res, next) => {
  als.run({ requestId: req.headers['x-request-id'] ?? randomUUID() }, next);
});

// где угодно глубже — без передачи параметров:
function log(msg) {
  const store = als.getStore(); // undefined вне контекста!
  console.log(`[${store?.requestId}] ${msg}`);
}
```

Всё, что вызвано (в т.ч. через await, setTimeout, промисы) изнутри `als.run()`, видит этот store. Разные запросы — разные store, изоляция автоматическая.

## Как это работает и что по цене

Под капотом — механизм наследования контекста через async-цепочки (промисы несут ссылку на контекст создания). Современные версии Node сделали ALS дешёвым (единицы процентов); это несравнимо дешевле, чем REQUEST scope в Nest, где на каждый запрос пересоздаётся поддерево DI.

Ловушки:
- **Разрыв контекста** — колбэки из очередей, созданных вне запроса (например, пул коннектов зовёт колбэк из своего контекста), сторонние EventEmitter. Лечение: `AsyncResource.bind(fn)` / `als.enterWith()`.
- `getStore()` вне контекста → `undefined` — всегда обрабатывать.

## Применения

1. **request-id в логах** — главный кейс; pino/winston берут id из ALS в каждом log-вызове.
2. **Трейсинг** — OpenTelemetry context propagation в Node построен на ALS.
3. **Текущий пользователь/tenant** — вместо REQUEST-scoped провайдера (см. [Injection scopes](../03-nestjs/02-injection-scopes.md)).
4. **Транзакции БД** — положить транзакционный EntityManager в ALS, репозитории берут его оттуда (так работают @nestjs-cls/transactional, typeorm-transactional).

В NestJS — либо руками (middleware + `als.run`), либо [nestjs-cls](https://github.com/Papooch/nestjs-cls).

## Что спрашивают на собеседовании

1. **Как сделать request-id в каждой строке лога без передачи через параметры?** — ALS: middleware кладёт id, логгер читает `getStore()`.
2. **ALS vs REQUEST scope в Nest?** — ALS: сервисы остаются синглтонами, дёшево; REQUEST scope: пересоздание DI-поддерева на запрос.
3. **Когда контекст «теряется»?** — колбэк исполняется из чужой async-цепочки (пулы, внешние эмиттеры); чинится `AsyncResource.bind`.
4. **На чём построен ALS?** — на отслеживании async-контекста в V8/Node (async_hooks/внутренний механизм континуаций); знать глубоко не требуют, важно понимать «контекст наследуется по цепочке вызовов».

## Ссылки

- [AsyncLocalStorage — официальная документация](https://nodejs.org/api/async_context.html) (включая `AsyncResource.bind` и troubleshooting потери контекста)
- [Рецепт ALS в доках NestJS](https://docs.nestjs.com/recipes/async-local-storage)
- [nestjs-cls](https://github.com/Papooch/nestjs-cls) и [@nestjs-cls/transactional](https://papooch.github.io/nestjs-cls/plugins/available-plugins/transactional)
- [OpenTelemetry JS context manager](https://github.com/open-telemetry/opentelemetry-js/tree/main/packages/opentelemetry-context-async-hooks) — ALS в бою у трейсинга
