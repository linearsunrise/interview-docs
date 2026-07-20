---
sidebar_position: 6
title: Guards
---

# Guards

> **TL;DR:** Guard отвечает на один вопрос: пропустить запрос к хендлеру или нет. Возвращает `boolean` (или Promise/Observable). Его суперсила — `ExecutionContext` + `Reflector`: он знает, какой хендлер сработает, и читает его метаданные.

## Канонический пример: RolesGuard

```ts
// декоратор
export const Roles = Reflector.createDecorator<string[]>();
// использование: @Roles(['admin'])

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    // getAllAndOverride: метаданные метода перекрывают метаданные класса
    const roles = this.reflector.getAllAndOverride(Roles, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (!roles) return true; // роут без @Roles — публичный

    const { user } = context.switchToHttp().getRequest();
    return roles.some((role) => user?.roles?.includes(role));
  }
}
```

`Reflector.createDecorator` — типизированная замена паре `SetMetadata` + строковый ключ (старый вариант `@SetMetadata('roles', [...])` тоже надо знать).

## Ключевые детали

- **ExecutionContext** расширяет `ArgumentsHost`: `getHandler()` / `getClass()` — для метаданных, `switchToHttp() / switchToRpc() / switchToWs()` — универсальность между транспортами.
- **Порядок:** глобальные → контроллера → метода. Первый `false`/исключение — дальше не идём, клиент получает 403 (`ForbiddenException` по умолчанию). Кинуть свой `UnauthorizedException` — можно и нужно.
- **Глобальный guard с DI** — через `{ provide: APP_GUARD, useClass: AuthGuard }`.
- Паттерн **`@Public()`**: глобальный auth-guard + декоратор-исключение, guard проверяет метаданные `isPublic`.

## Что спрашивают на собеседовании

1. **Guard vs Middleware для авторизации?** — guard знает хендлер и его метаданные; middleware выполняется до роутинга и не знает ничего.
2. **`getAllAndOverride` vs `getAllAndMerge`?** — override: метод перекрывает класс (роли); merge: объединить (теги).
3. **Как сделать все роуты закрытыми по умолчанию?** — глобальный `APP_GUARD` + `@Public()` декоратор для исключений.
4. **Что вернётся клиенту при `return false` vs `throw new UnauthorizedException()`?** — 403 Forbidden по умолчанию vs ваш 401.
5. **Как guard работает в микросервисах/WebSocket?** — тот же интерфейс, контекст через `switchToRpc()` / `switchToWs()`.

## Ссылки

- [Guards — официальная документация](https://docs.nestjs.com/guards)
- [Execution context + Reflector](https://docs.nestjs.com/fundamentals/execution-context) — `createDecorator`, `getAllAndOverride/Merge`
- [Authentication (полный рецепт с JWT и @Public)](https://docs.nestjs.com/security/authentication)
- Исходники: [`packages/core/guards/guards-consumer.ts`](https://github.com/nestjs/nest/blob/master/packages/core/guards/guards-consumer.ts) — крошечный файл, видно как `tryActivate` обходит guards
