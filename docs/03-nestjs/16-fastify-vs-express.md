---
sidebar_position: 16
title: Fastify vs Express
---

# Fastify vs Express

> **TL;DR:** NestJS не привязан к HTTP-фреймворку — работает через адаптеры (`AbstractHttpAdapter`). Express — дефолт и максимум совместимости; Fastify — заметно быстрее на «тонких» эндпоинтах (JSON in/out) за счёт schema-based сериализации и оптимизированного роутинга.

## Подключение

```ts
const app = await NestFactory.create<NestFastifyApplication>(
  AppModule,
  new FastifyAdapter({ logger: true }),
);
await app.listen(3000, '0.0.0.0'); // у Fastify дефолт — localhost, в докере нужен 0.0.0.0!
```

## Что меняется на практике

| Аспект | Express | Fastify |
|---|---|---|
| Производительность | базовая | ~в 2 раза больше req/s на hello-world (см. официальные бенчмарки) |
| Экосистема middleware | огромная, всё «просто работает» | нужны `@fastify/*`-аналоги (helmet, cookie, multipart) |
| `@Res()` типы | `express.Response` | `FastifyReply` — другой API (`.send()` вместо `.json()`) |
| Валидация | в Nest — через pipes | есть своя JSON-schema (в Nest обычно не используется) |

Библиотеки, жёстко завязанные на Express (некоторые passport-стратегии, старые middleware), — главная причина оставаться на Express.

## Честный senior-ответ про производительность

Выигрыш Fastify — в сериализации JSON и роутинге. Если запрос ходит в БД за 50мс, разница между 0.1мс и 0.05мс фреймворка не видна: **боттлнек почти всегда I/O, а не HTTP-слой**. Fastify оправдан на высоконагруженных «тонких» сервисах (прокси, агрегаторы, high-RPS API); переезжать ради переезда — нет. Такой ответ (с оговоркой) ценится больше, чем «Fastify быстрее, надо брать».

## Что спрашивают на собеседовании

1. **Как Nest абстрагируется от HTTP-фреймворка?** — паттерн «адаптер»: `AbstractHttpAdapter`, реализации `ExpressAdapter`/`FastifyAdapter`; платформо-специфика — только в `@Req/@Res` и middleware.
2. **Что сломается при переезде Express → Fastify?** — middleware-экосистема, типы `@Res()`, поведение по умолчанию (bind на localhost), file upload (`multer` → `@fastify/multipart`).
3. **Когда выигрыш Fastify реален?** — CPU-профиль запроса тонкий (мало I/O), высокий RPS; иначе боттлнек в БД/сети.
4. **Что такое `@Res({ passthrough: true })`?** — доступ к response (куки, заголовки) без отключения стандартного механизма ответа Nest — работает в обоих адаптерах.

## Ссылки

- [Performance (Fastify) — официальная документация](https://docs.nestjs.com/techniques/performance)
- [Официальные бенчмарки Fastify](https://fastify.dev/benchmarks/)
- Исходники: [`packages/platform-fastify`](https://github.com/nestjs/nest/tree/master/packages/platform-fastify) и [`packages/platform-express`](https://github.com/nestjs/nest/tree/master/packages/platform-express) — сравнить два адаптера; [`packages/core/adapters/http-adapter.ts`](https://github.com/nestjs/nest/blob/master/packages/core/adapters/http-adapter.ts) — контракт `AbstractHttpAdapter`
- [find-my-way](https://github.com/delvedor/find-my-way) — radix-tree роутер, за счёт которого Fastify быстрый
