---
sidebar_position: 8
title: Interceptors и RxJS
---

# Interceptors и RxJS

> **TL;DR:** Interceptor оборачивает вызов хендлера: код **до** `next.handle()` — pre-фаза, операторы RxJS на возвращённом Observable — post-фаза. Типовые применения: логирование времени, единый envelope ответа, timeout, кэш, маппинг ошибок.

## Скелет

```ts
@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<unknown> {
    const start = Date.now();                       // PRE
    return next.handle().pipe(                      // вызов хендлера
      tap(() => console.log(`took ${Date.now() - start}ms`)), // POST
    );
  }
}
```

Важно: `next.handle()` возвращает **cold Observable** — если его не вернуть/не подписаться, хендлер вообще не выполнится. На этом строится кэш-interceptor: вернуть `of(cachedValue)` вместо `next.handle()`.

## Джентльменский набор операторов

```ts
next.handle().pipe(
  map(data => ({ data, success: true })),          // envelope ответа
  timeout(5000),                                    // таймаут
  catchError(err => throwError(() =>                // маппинг ошибок
    err instanceof TimeoutError ? new RequestTimeoutException() : err)),
  tap({ error: e => this.logger.error(e) }),       // side-эффект на ошибку
)
```

Ещё из коробки: `ClassSerializerInterceptor` (применяет `@Exclude()/@Expose()` из class-transformer к возвращаемым объектам), `CacheInterceptor` из `@nestjs/cache-manager`.

## Порядок («луковица»)

Pre-фаза: глобальные → контроллера → метода. Post-фаза: в обратном порядке. Ошибка из guard в interceptor **не попадает** (guards выполняются раньше); ошибка из хендлера — попадает в `catchError`.

## Что спрашивают на собеседовании

1. **Interceptor для замера времени + request-id** — см. скелет; request-id брать из ALS или заголовка.
2. **Как обернуть все ответы в `{ data, meta }`?** — глобальный interceptor с `map`; исключения он не поймает — их формат задаёт exception filter (согласовать оба).
3. **Почему `next.handle()` можно не вызывать и что произойдёт?** — хендлер не выполнится; так работает кэширование/short-circuit.
4. **Отличие от middleware?** — interceptor знает хендлер (ExecutionContext), работает вокруг него и видит результат; middleware — только до роутинга.
5. **Что будет с ошибкой, брошенной в pipe?** — она пройдёт через post-фазу interceptors (catchError её видит), потом в filters.

## Ссылки

- [Interceptors — официальная документация](https://docs.nestjs.com/interceptors)
- [Serialization (ClassSerializerInterceptor)](https://docs.nestjs.com/techniques/serialization)
- [Caching](https://docs.nestjs.com/techniques/caching)
- [RxJS: операторы map / tap / catchError / timeout](https://rxjs.dev/api)
- Исходники: [`packages/core/interceptors/interceptors-consumer.ts`](https://github.com/nestjs/nest/blob/master/packages/core/interceptors/interceptors-consumer.ts) — как цепочка interceptors собирается через `reduce` в «луковицу»
