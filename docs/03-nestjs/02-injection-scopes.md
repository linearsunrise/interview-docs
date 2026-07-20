---
sidebar_position: 2
title: Injection scopes
---

# Injection scopes

> **TL;DR:** DEFAULT — синглтон на всё приложение. REQUEST — новый инстанс на каждый запрос. TRANSIENT — новый инстанс на каждого потребителя. REQUEST «пузырится» вверх по графу зависимостей — это главный подводный камень.

## Три скоупа

```ts
@Injectable() // Scope.DEFAULT — синглтон, создаётся при старте
@Injectable({ scope: Scope.REQUEST })  // на каждый запрос
@Injectable({ scope: Scope.TRANSIENT }) // на каждое место инжекта
```

## Пузырение (scope bubbling)

Если `CatsService` (DEFAULT) зависит от `CatsRepository` (REQUEST) — `CatsService` **сам становится REQUEST-scoped**, как и контроллер, который его использует. Вся цепочка вверх пересоздаётся на каждый запрос → аллокации, нагрузка на GC, просадка p99. Исключение: TRANSIENT не пузырится — транзиентным остаётся только он сам.

REQUEST-scoped провайдеру доступен сам запрос: `@Inject(REQUEST) private req: Request`.

Ещё одно следствие: REQUEST-scoped провайдеры **лениво инстанцируются** — `onModuleInit` у них ведёт себя иначе, а достать их через `ModuleRef.get()` нельзя, только `ModuleRef.resolve(Token, contextId)`.

## Альтернатива: AsyncLocalStorage

В 90% случаев REQUEST scope нужен ради «прокинуть request-id / текущего юзера / tenant». Это дешевле делать через `AsyncLocalStorage` (нода) или обёртку [nestjs-cls](https://github.com/Papooch/nestjs-cls): все сервисы остаются синглтонами, а контекст запроса читается из ALS.

## Durable providers (мультитенантность)

Если REQUEST scope всё же нужен (например, пул коннектов на тенанта), с NestJS 9 есть **durable providers**: пишешь свою `ContextIdStrategy`, которая по заголовку (`x-tenant-id`) возвращает один и тот же `contextId` для одного тенанта — DI-поддерево создаётся не на каждый запрос, а один раз на тенанта.

## Что спрашивают на собеседовании

1. **Что будет, если заинжектить REQUEST-scoped в DEFAULT-scoped?** — принимающий провайдер сам станет REQUEST-scoped (пузырение), вплоть до контроллера.
2. **Чем это плохо?** — инстанцирование всей цепочки на каждый запрос: память, GC, латентность.
3. **Как получить request-контекст без REQUEST scope?** — `AsyncLocalStorage` / nestjs-cls, interceptor или middleware кладёт данные в store.
4. **TRANSIENT vs REQUEST?** — TRANSIENT: свой инстанс у каждого *потребителя* (классика — логгер с контекстом класса); REQUEST: свой инстанс на каждый *запрос*.
5. **Как реализовать мультитенантность с коннектом к БД по заголовку?** — durable providers + `ContextIdStrategy`, либо фабрика-синглтон, которая держит Map тенант→коннект и выбирает по данным из ALS.

## Ссылки

- [Injection scopes — официальная документация](https://docs.nestjs.com/fundamentals/injection-scopes) (внизу — раздел про durable providers)
- [Async Local Storage — рецепт в доках NestJS](https://docs.nestjs.com/recipes/async-local-storage)
- [`AsyncLocalStorage` в Node.js](https://nodejs.org/api/async_context.html)
- [nestjs-cls](https://github.com/Papooch/nestjs-cls) — CLS-обёртка, стандарт де-факто
- [Анонс NestJS 9 (durable providers)](https://trilon.io/blog/nestjs-9-is-now-available)
- Исходники: [`packages/core/injector/injector.ts`](https://github.com/nestjs/nest/blob/master/packages/core/injector/injector.ts) — поиск по `contextId` показывает, как резолвятся per-request инстансы
