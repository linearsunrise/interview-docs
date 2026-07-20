---
sidebar_position: 5
title: "RBAC vs ABAC, реализация в Nest (guards + CASL)"
---

# RBAC vs ABAC, реализация в Nest: guards + метаданные, CASL

> **TL;DR:** RBAC (Role-Based Access Control) отвечает на вопрос "какая у пользователя роль и что этой роли разрешено" — просто, но грубо: не различает "редактировать любой пост" от "редактировать **свой** пост". ABAC (Attribute-Based Access Control) проверяет правило на основе атрибутов **и пользователя, и конкретного ресурса** — "является ли `post.authorId === user.id`" — то, что нужно для реальных прав уровня "владелец ресурса", а не только роли. Путаница RBAC с полноценной авторизацией — источник IDOR-уязвимостей (см. чек-лист [Безопасность](./index.md)): роль проверена, владение ресурсом — нет.

## RBAC — роль решает всё

```ts
// декоратор + guard: НЕ знает про конкретный ресурс, только про роль вызывающего
export const Roles = (...roles: string[]) => SetMetadata('roles', roles);

@Injectable()
class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}
  canActivate(ctx: ExecutionContext): boolean {
    const required = this.reflector.get<string[]>('roles', ctx.getHandler());
    if (!required) return true;
    const user = ctx.switchToHttp().getRequest().user;
    return required.some((role) => user.roles.includes(role));
  }
}

@Roles('editor')
@Patch(':id')
updatePost(@Param('id') id: string, @Body() dto: UpdatePostDto) {
  return this.postsService.update(id, dto); // РОЛЬ проверена, но ЧЕЙ это пост — нет
}
```

Это ровно паттерн `@Roles()` + `Reflector`, разобранный в [Guards](../03-nestjs/06-guards.md) — метаданные декоратора читаются guard'ом через `ExecutionContext`. Проблема: guard знает только про роль пользователя и наличие декоратора на хендлере — он физически не видит, что `id` в параметрах маршрута указывает на пост **чужого** автора. Пользователь с ролью `editor` пройдёт этот guard для **любого** `id`, включая посты, которые не должен редактировать — классический IDOR (Insecure Direct Object Reference), если проверка владения не добавлена отдельно, внутри самого хендлера/сервиса.

## ABAC / resource-based — учитывает конкретный ресурс

```ts
async updatePost(userId: string, postId: string, dto: UpdatePostDto) {
  const post = await this.postsRepo.findById(postId);
  if (!post) throw new NotFoundException();
  if (post.authorId !== userId && !this.isAdmin(userId)) {
    throw new ForbiddenException(); // атрибут РЕСУРСА (authorId), не только роль пользователя
  }
  return this.postsRepo.update(postId, dto);
}
```

Правило "редактировать можно только свои посты" физически не может быть выражено через `@Roles()` на уровне guard'а до вызова хендлера — guard срабатывает **до** того, как известно, какой конкретно ресурс запрошен (для этого нужно сходить в БД за самим постом). Это структурная причина, по которой resource-based проверки обычно живут внутри сервисного слоя (после загрузки ресурса), а не в guard'е — хотя NestJS позволяет и guard'ам делать асинхронные проверки с походом в БД, если действительно нужно отклонить запрос до хендлера.

## CASL — декларативные ability-правила вместо разбросанных `if`

```ts
function defineAbilitiesFor(user: User) {
  const { can, cannot, build } = new AbilityBuilder(createMongoAbility);

  can('read', 'Post'); // все могут читать посты
  can('update', 'Post', { authorId: user.id }); // атрибут ресурса — только свой пост
  if (user.roles.includes('admin')) can('manage', 'all'); // роль — полный доступ поверх обычных правил
  cannot('delete', 'Post', { status: 'published' }).because('Cannot delete published posts');

  return build();
}

// в хендлере/guard'е:
const ability = defineAbilitiesFor(user);
if (ability.cannot('update', subject('Post', post))) throw new ForbiddenException();
```

CASL объединяет и RBAC (`if (user.roles.includes('admin'))`), и ABAC (условия по атрибутам ресурса `{ authorId: user.id }`) в единый декларативный набор правил, который описывается **в одном месте** (обычно фабрика `defineAbilitiesFor`), а не размазан по `if`-проверкам в каждом сервисном методе отдельно. Практическая польза — правила становятся тестируемыми независимо от контроллеров/сервисов (юнит-тест на "может ли обычный пользователь редактировать чужой пост" не требует поднятия HTTP-слоя) и централизованно аудируемыми ("покажите мне все правила доступа к `Post`" — один файл, а не грёп по всей кодовой базе).

## RBAC vs ABAC — когда чего достаточно

| | RBAC | ABAC / resource-based |
|---|---|---|
| Простота | максимальная — просто список ролей | требует загрузки ресурса перед проверкой |
| Гранулярность | уровня "тип действия для роли" | уровня "это конкретное действие с этим конкретным ресурсом для этого пользователя" |
| Типичный кейс | админ-панель с чёткими ролями (admin/moderator/user) без владения ресурсами | пользовательский контент (свои посты/заказы/документы), multi-tenant системы |
| Риск при недостаточности | IDOR — роль проверена, владение — нет | — (сама модель это и решает) |

Практика — не "выбрать одно навсегда", а комбинировать: RBAC как первый грубый фильтр (может ли эта роль вообще выполнять такое действие в принципе), ABAC/resource-based — как более тонкая проверка после того, как известен конкретный ресурс. CASL (и похожие библиотеки в других экосистемах) как раз и построен вокруг этой комбинации, а не замены одного другим.

## Что спрашивают на собеседовании

1. **Как реализовать permissions уровня "редактировать можно только свои посты"?** — RBAC недостаточен (не видит конкретный ресурс) — нужна resource-based проверка после загрузки ресурса: сравнение атрибута ресурса (`authorId`) с ID текущего пользователя, обычно в сервисном слое или через CASL-подобную библиотеку.
2. **Почему проверка одной только роли — риск IDOR?** — роль подтверждает "тип действия разрешён этому пользователю в принципе", но не то, что конкретный запрошенный ресурс (по ID из параметров) принадлежит именно этому пользователю — нужна отдельная явная проверка владения.
3. **Почему resource-based проверки обычно не делаются в guard'е до хендлера?** — guard срабатывает до того, как известен конкретный запрошенный ресурс — для проверки атрибута ресурса (владелец) нужно сначала сходить в БД за самим ресурсом, что обычно естественнее в сервисном слое.
4. **Что даёт CASL по сравнению с разбросанными `if`-проверками?** — единое декларативное место описания всех правил доступа (и по ролям, и по атрибутам ресурса), тестируемое независимо от контроллеров и централизованно аудируемое.
5. **RBAC и ABAC — взаимоисключающие подходы?** — нет, комбинируются: RBAC как грубый первый фильтр по роли, ABAC/resource-based — как более точная проверка на уровне конкретного ресурса после того, как он известен.

## Ссылки

- [NIST — Role-Based Access Control (RBAC)](https://csrc.nist.gov/projects/role-based-access-control)
- [NIST SP 800-162 — Guide to Attribute Based Access Control (ABAC)](https://csrc.nist.gov/pubs/sp/800/162/final)
- [CASL — официальная документация](https://casl.js.org/v6/en/guide/intro)
- [OWASP — Insecure Direct Object References (IDOR)](https://owasp.org/www-community/attacks/Insecure_Direct_Object_Reference)
- [Guards в NestJS: ExecutionContext, Reflector, RolesGuard](../03-nestjs/06-guards.md)
