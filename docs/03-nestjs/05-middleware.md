---
sidebar_position: 5
title: Middleware
---

# Middleware

> **TL;DR:** Функция в стиле Express, выполняется **до** роутинга и всего пайплайна NestJS. Не знает, какой хендлер сработает — поэтому не место для авторизации. Место для: raw body, cors, helmet, request-id, низкоуровневого логирования.

## Два вида

```ts
// Функциональный — когда нет зависимостей
export function loggerMiddleware(req: Request, res: Response, next: NextFunction) {
  console.log(`${req.method} ${req.url}`);
  next();
}

// Классовый — когда нужен DI
@Injectable()
export class LoggerMiddleware implements NestMiddleware {
  constructor(private logger: LoggerService) {}
  use(req: Request, res: Response, next: NextFunction) {
    this.logger.log(`${req.method} ${req.url}`);
    next();
  }
}
```

## Подключение

```ts
@Module({...})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(LoggerMiddleware)
      .exclude({ path: 'health', method: RequestMethod.GET })
      .forRoutes('*'); // или конкретный контроллер / { path, method }
  }
}
```

Глобально без DI: `app.use(loggerMiddleware)` в `main.ts`.

## Отличие от guard/interceptor — главное на собесе

- Выполняется **до резолва роута**: нет `ExecutionContext`, нет доступа к метаданным хендлера (`Reflector` бесполезен).
- Работает в терминах конкретной платформы (`req/res` Express или Fastify) — привязка к HTTP.
- `next()` обязателен, иначе запрос повиснет.

## Что спрашивают на собеседовании

1. **Middleware vs Guard — где делать аутентификацию?** — можно распарсить токен в middleware, но решение «пускать/не пускать» по ролям — только в guard (нужны метаданные хендлера).
2. **Когда middleware — единственный вариант?** — когда нужно тело запроса до его парсинга (raw body для webhook-подписей Stripe), или интеграция express-совместимых библиотек (helmet, compression).
3. **Выполнится ли middleware, если роут не существует?** — да (он до роутинга), а guards/pipes — нет, будет 404 из роутера.
4. **Как в классовый middleware попадают зависимости?** — обычный DI: он инстанцируется контейнером модуля, где сконфигурирован.

## Ссылки

- [Middleware — официальная документация](https://docs.nestjs.com/middleware)
- [Raw body](https://docs.nestjs.com/faq/raw-body) — типовой кейс для webhook-ов
- Исходники: [`packages/core/middleware/middleware-module.ts`](https://github.com/nestjs/nest/blob/master/packages/core/middleware/middleware-module.ts) — как middleware привязывается к роутам
