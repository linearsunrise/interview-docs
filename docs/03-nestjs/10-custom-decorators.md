---
sidebar_position: 10
title: Кастомные декораторы
---

# Кастомные декораторы

> **TL;DR:** Три инструмента: `createParamDecorator` (достать что-то из запроса в параметр хендлера), `SetMetadata`/`Reflector.createDecorator` (повесить метаданные для guards/interceptors), `applyDecorators` (склеить несколько декораторов в один).

## createParamDecorator

```ts
export const CurrentUser = createParamDecorator(
  (data: keyof User | undefined, ctx: ExecutionContext) => {
    const user = ctx.switchToHttp().getRequest().user;
    return data ? user?.[data] : user; // @CurrentUser('email') → user.email
  },
);

@Get('me')
me(@CurrentUser() user: User) {}
```

Бонус: к кастомному параметр-декоратору можно применять pipes — `@CurrentUser(new ValidationPipe())`.

## Метаданные

```ts
// старый способ:
export const Roles = (...roles: string[]) => SetMetadata('roles', roles);
// новый типизированный:
export const Roles = Reflector.createDecorator<string[]>();
// чтение в guard:
this.reflector.getAllAndOverride(Roles, [ctx.getHandler(), ctx.getClass()]);
```

## Композиция: applyDecorators

```ts
export function Auth(...roles: string[]) {
  return applyDecorators(
    Roles(roles),
    UseGuards(JwtAuthGuard, RolesGuard),
    ApiBearerAuth(),                       // swagger
    ApiUnauthorizedResponse({ description: 'Unauthorized' }),
  );
}

@Auth('admin')  // один декоратор вместо четырёх на каждом роуте
@Get()
findAll() {}
```

Это любимый вопрос про «как убрать дублирование декораторов на 50 контроллерах».

## Что спрашивают на собеседовании

1. **Как сделать `@CurrentUser()`?** — `createParamDecorator` + чтение `req.user` (его туда кладёт auth guard/passport).
2. **Как guard узнаёт о `@Roles()` на хендлере?** — метаданные через `SetMetadata`/`createDecorator`, чтение через `Reflector` с `getHandler()/getClass()`.
3. **`applyDecorators` — зачем?** — композиция: auth + swagger + роли одним декоратором, единая точка изменения.
4. **На чём это всё работает?** — `reflect-metadata`: декораторы пишут метаданные на класс/метод, Reflector их читает в рантайме.

## Ссылки

- [Custom decorators — официальная документация](https://docs.nestjs.com/custom-decorators)
- [Execution context — Reflector, createDecorator](https://docs.nestjs.com/fundamentals/execution-context)
- Исходники: [`packages/common/decorators`](https://github.com/nestjs/nest/tree/master/packages/common/decorators) — все встроенные декораторы это 3–10 строк поверх `SetMetadata`, полезно полистать для демистификации
