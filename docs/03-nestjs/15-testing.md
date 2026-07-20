---
sidebar_position: 15
title: Тестирование
---

# Тестирование

> **TL;DR:** Юнит: `Test.createTestingModule` + `overrideProvider` с моками. E2E: `createNestApplication` + supertest, guards подменяются через `overrideGuard`. Ключевая идея — DI делает всё подменяемым.

## Юнит-тест сервиса

```ts
const module = await Test.createTestingModule({
  providers: [
    CatsService,
    { provide: CatsRepository, useValue: { find: jest.fn().mockResolvedValue([cat]) } },
  ],
}).compile();

const service = module.get(CatsService);
```

Альтернатива ручным моками — **авто-мокинг**:

```ts
Test.createTestingModule({ providers: [CatsService] })
  .useMocker(createMock)   // @golevelup/ts-jest: все зависимости — авто-моки
  .compile();
```

Для чистой логики можно вообще без TestingModule: `new CatsService(mockRepo)` — быстрее и проще; TestingModule нужен, когда важна сборка графа.

## E2E

```ts
const module = await Test.createTestingModule({ imports: [AppModule] })
  .overrideProvider(PaymentGateway).useValue(fakeGateway)
  .overrideGuard(JwtAuthGuard).useValue({ canActivate: (ctx) => {
    ctx.switchToHttp().getRequest().user = testUser;
    return true;
  }})
  .compile();

app = module.createNestApplication();
app.useGlobalPipes(new ValidationPipe({ whitelist: true })); // как в main.ts!
await app.init();

await request(app.getHttpServer()).get('/cats').expect(200);
```

Грабли: всё, что вы настраиваете в `main.ts` (глобальные pipes, prefix, versioning), **не применяется автоматически** в e2e — надо повторить (или вынести настройку в общую функцию `setupApp(app)` и звать из обоих мест).

## Что ещё знать

- `overrideGuard/overrideInterceptor/overridePipe/overrideFilter` — подмена enhancer-ов, повешенных декораторами (`@UseGuards` и т.п.).
- Guard, зарегистрированный глобально через `APP_GUARD`, — это обычный провайдер: либо мокайте его зависимости (например, `JwtService`), либо шлите в тестах настоящий тестовый токен — заодно проверите auth-цепочку.
- REQUEST-scoped провайдер в тесте: `module.resolve(Token)` (не `get`), можно с `ContextIdFactory.create()`.
- БД в e2e: testcontainers (реальный Postgres в докере) > sqlite-подмена (другой диалект — другие баги).

## Что спрашивают на собеседовании

1. **Как тестировать сервис с зависимостями?** — мокать зависимости через `overrideProvider`/`useValue` или конструктором напрямую; тестируем поведение, не реализацию.
2. **Как подменить guard в e2e?** — `overrideGuard(JwtAuthGuard).useValue({ canActivate: ... })`, заодно подложить `req.user`.
3. **Чем unit отличается от e2e в Nest-терминах?** — unit: кусок графа с моками, без HTTP; e2e: полный граф + реальный HTTP через supertest.
4. **Почему e2e проходит, а прод падает на валидации?** — глобальный pipe из `main.ts` не был добавлен в тестовое приложение.
5. **Как достать REQUEST-scoped провайдер в тесте?** — `module.resolve()` с contextId.

## Ссылки

- [Testing — официальная документация](https://docs.nestjs.com/fundamentals/testing) — включая overrideProvider, auto-mocking, request-scoped
- [supertest](https://github.com/ladjs/supertest)
- [@golevelup/ts-jest createMock](https://github.com/golevelup/nestjs/tree/master/packages/testing)
- [Testcontainers for Node.js](https://node.testcontainers.org/)
- Исходники: [`packages/testing/testing-module.builder.ts`](https://github.com/nestjs/nest/blob/master/packages/testing/testing-module.builder.ts) — как работают override-методы
