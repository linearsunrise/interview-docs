---
sidebar_position: 9
title: Exception filters
---

# Exception filters

> **TL;DR:** Последний рубеж: превращают исключение в HTTP-ответ. По умолчанию встроенный глобальный фильтр отдаёт JSON `{ statusCode, message }` для `HttpException` и `500 Internal server error` для всего остального.

## База

```ts
throw new NotFoundException('Cat not found');
// → 404 { "statusCode": 404, "message": "Cat not found", "error": "Not Found" }
```

Вся иерархия (`BadRequestException`, `ConflictException`, ...) наследует `HttpException`. Неизвестное исключение (например, `TypeError`) → 500 без деталей (и это правильно: не светить внутренности).

## Свой фильтр

```ts
@Catch(HttpException)               // без аргументов = ловит всё
export class HttpExceptionFilter implements ExceptionFilter {
  catch(exception: HttpException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const res = ctx.getResponse<Response>();
    const req = ctx.getRequest<Request>();
    res.status(exception.getStatus()).json({
      statusCode: exception.getStatus(),
      path: req.url,
      timestamp: new Date().toISOString(),
      message: exception.getResponse(),
    });
  }
}
```

Подключение: `@UseFilters()` на метод/контроллер, `{ provide: APP_FILTER, useClass: ... }` глобально.

**`BaseExceptionFilter`**: если нужно только дополнить поведение (залогировать в Sentry), наследуйтесь от него и в конце зовите `super.catch(exception, host)` — не придётся переизобретать маппинг статусов.

## Порядок и границы

- Срабатывает **один** самый специфичный фильтр: метода → контроллера → глобальный.
- Ловит исключения из guards, pipes, interceptors и хендлера. Не ловит: ошибки в middleware (это зона платформы) и ошибки вне request-контекста (фоновые задачи, необработанные promise).
- В микросервисах — `RpcException` и фильтр с `host.switchToRpc()`; в WS — `WsException`.

## Что спрашивают на собеседовании

1. **Как сделать единый формат ошибок на всё приложение?** — глобальный catch-all фильтр (`@Catch()` без аргументов) + согласовать формат с envelope-interceptor'ом успешных ответов.
2. **Как логировать 500-е в Sentry, не меняя формат ответов?** — унаследоваться от `BaseExceptionFilter`, залогировать, вызвать `super.catch`.
3. **Поймает ли фильтр ошибку из guard?** — да. А из middleware — нет.
4. **Как замапить ошибки БД (например, unique violation) в 409?** — фильтр `@Catch(QueryFailedError)` (TypeORM) или `@Catch(Prisma.PrismaClientKnownRequestError)` с проверкой кода.
5. **Что с ошибками в async-коде вне запроса?** — фильтры не помогут; нужны `unhandledRejection`-хендлеры и логика фоновых задач.

## Ссылки

- [Exception filters — официальная документация](https://docs.nestjs.com/exception-filters)
- Исходники: [`packages/core/exceptions/base-exception-filter.ts`](https://github.com/nestjs/nest/blob/master/packages/core/exceptions/base-exception-filter.ts) — встроенный маппинг; [`packages/common/exceptions`](https://github.com/nestjs/nest/tree/master/packages/common/exceptions) — вся иерархия HttpException
- [Microservices exception filters](https://docs.nestjs.com/microservices/exception-filters)
