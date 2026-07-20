---
sidebar_position: 3
title: NestJS
---

# NestJS

## Чеклист знаний

- [ ] DI-контейнер: providers, tokens, `useClass/useValue/useFactory/useExisting`
- [ ] Injection scopes: DEFAULT, REQUEST, TRANSIENT — и цена REQUEST-scope (пузырение)
- [ ] Circular dependencies: `forwardRef`, почему это запах и как рефакторить
- [ ] Request lifecycle: middleware → guards → interceptors (pre) → pipes → handler → interceptors (post) → exception filters
- [ ] Отличия и зоны ответственности: Middleware / Guard / Interceptor / Pipe / Filter
- [ ] Кастомные декораторы (`createParamDecorator`, композиция через `applyDecorators`)
- [ ] Dynamic modules: `forRoot/forRootAsync/register` — в чём разница по смыслу
- [ ] ConfigModule, валидация env (Joi/zod), typed config
- [ ] Lifecycle hooks: `onModuleInit`, `onApplicationBootstrap`, `onModuleDestroy`, `enableShutdownHooks`
- [ ] Микросервисы: транспорты, `@MessagePattern` vs `@EventPattern`, hybrid app
- [ ] Тестирование: `Test.createTestingModule`, overrideProvider, e2e через supertest
- [ ] Fastify vs Express адаптеры
- [ ] Interceptors + RxJS: map, catchError, timeout — типовые применения

## Вопросы с собеседований

1. Расскажите полный lifecycle запроса в NestJS. В каком порядке отработают глобальный, контроллерный и метод-левел guard?
2. Guard vs Middleware — почему авторизацию делают в guard, а не в middleware? (доступ к ExecutionContext, метаданным хендлера)
3. Как работает DI под капотом? Что произойдёт, если инжектить REQUEST-scoped провайдер в DEFAULT-scoped?
4. Как разрулить циклическую зависимость между сервисами? Почему `forwardRef` — плохое долгосрочное решение?
5. Чем `forRoot` отличается от `forRootAsync`? Когда нужен `forFeature`?
6. Как реализовать мультитенантность (например, коннект к БД на основе заголовка)?
7. Interceptor для логирования времени ответа + для трансформации ответа в единый envelope — как?
8. `@MessagePattern` vs `@EventPattern` — request/response vs fire-and-forget.
9. Как тестировать сервис с зависимостями? Как подменить guard в e2e?
10. Как организовать структуру большого проекта: модули по фичам, shared, барельные экспорты — и какие проблемы с ними?

## Ключевые тезисы

**REQUEST scope «пузырится» вверх:** всё, что зависит от REQUEST-scoped провайдера, само становится REQUEST-scoped — инстансы создаются на каждый запрос, это удар по перфу. Альтернатива — AsyncLocalStorage (nestjs-cls).

**Порядок для глобальных/контроллерных/метод-левел:** guards и pipes идут сверху вниз (global → controller → method), interceptors — pre сверху вниз, post — снизу вверх (как луковица).

**forRoot vs forRootAsync:** async нужен, когда конфиг модуля зависит от других провайдеров (ConfigService) — доступен inject + useFactory.

**Циклические зависимости:** обычно признак того, что нужен третий модуль/сервис с общей логикой или события вместо прямого вызова.

## Красные флаги

- Путает порядок: «сначала pipes, потом guards»
- «Всё кладу в middleware, зачем guards»
- Не знает про цену REQUEST scope
- Никогда не писал тесты с TestingModule

## Практика

- Написать: LoggingInterceptor (время + request-id), RolesGuard с кастомным декоратором `@Roles()`, ValidationPipe с zod
- Собрать dynamic module с `forRootAsync` (например, обёртка над Redis-клиентом)
- Поднять два Nest-приложения через RabbitMQ/Redis transport: одно шлёт event, другое обрабатывает
