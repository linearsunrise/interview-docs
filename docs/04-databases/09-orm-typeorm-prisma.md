---
sidebar_position: 9
title: "ORM: N+1, транзакции, миграции (TypeORM/Prisma)"
---

# TypeORM/Prisma: N+1, транзакции, миграции, eager vs lazy

> **TL;DR:** Почти все "непонятные тормоза" на ORM в проде сводятся к N+1 — одному запросу за списком плюс по одному запросу за связанными данными на каждую строку. Обе экосистемы (TypeORM, Prisma) умеют джойнить или батчить связанные данные заранее — проблема почти всегда в том, что это не включено явно там, где нужно. Транзакции в ORM — тонкий момент: важно, что именно передаётся в колбэк/менеджер транзакции, а не просто обёртка `try { ... } catch { rollback }`.

## N+1 — как обнаружить и починить

```ts
// TypeORM — классический N+1
const orders = await orderRepo.find(); // 1 запрос
for (const order of orders) {
  order.user = await userRepo.findOneBy({ id: order.userId }); // N запросов, по одному на заказ
}
```

```ts
// Prisma — то же самое через отдельные вызовы в цикле
const orders = await prisma.order.findMany();
for (const order of orders) {
  const user = await prisma.user.findUnique({ where: { id: order.userId } }); // N+1
}
```

**Обнаружение:** включить логирование SQL (`logging: true` в TypeORM, `log: ['query']` в Prisma) на дев-стенде и посчитать число запросов на один HTTP-запрос — если растёт линейно с размером списка, это N+1. В проде — APM (New Relic/Datadog) с трейсингом запросов к БД на транзакцию, либо просто счётчик запросов на request через middleware.

**Починка — две стратегии:**

```ts
// TypeORM: join сразу — один SQL-запрос с JOIN
const orders = await orderRepo.find({ relations: { user: true } });

// Prisma: include — один SQL-запрос (Prisma сама решает: JOIN или отдельный батч-запрос)
const orders = await prisma.order.findMany({ include: { user: true } });

// Явный батч через IN, если join нежелателен (например, сильно "широкая" связанная таблица)
const userIds = [...new Set(orders.map(o => o.userId))];
const users = await userRepo.findBy({ id: In(userIds) }); // 1 запрос вместо N
const byId = new Map(users.map(u => [u.id, u]));
```

## Join vs отдельный запрос с IN — когда что

`JOIN` — один round-trip к базе, но если связанных строк много (например, `orders` → `order_items`, по 50 позиций на заказ), результат "размножается" по декартову произведению — при нескольких one-to-many связях одновременно объём передаваемых данных может взорваться (JOIN несколько one-to-many сразу почти всегда хуже, чем отдельные IN-запросы). Отдельный запрос с `IN` — минимум 2 round-trip'а, но каждый компактный и предсказуемый по объёму, легче кэшируется отдельно. Практическое правило: одна to-one связь — почти всегда `JOIN`/`include`; несколько to-many связей одновременно — батч через `IN`, ORM (особенно Prisma) часто сама выбирает эту стратегию под капотом при `include`.

## Eager vs lazy loading

```ts
// TypeORM — eager: связь подгружается ВСЕГДА при любом find(), даже если не нужна
@ManyToOne(() => User, { eager: true })
user: User;

// lazy — Promise, резолвится только при обращении (требует await user.user)
@ManyToOne(() => User, { lazy: true })
user: Promise<User>;
```

`eager: true` — соблазнительно удобно, но опасно: связь тянется в **каждом** запросе через этот репозиторий, включая места, где она не нужна, — незаметный побочный источник лишних JOIN'ов по всей кодовой базе. На практике предпочитают explicit loading (`relations: {...}` точечно на каждый запрос) — ORM не решает за разработчика, что тянуть, разработчик явно указывает под конкретный кейс использования.

## Транзакции в ORM

```ts
// TypeORM — весь колбэк выполняется в одной транзакции, коммит/rollback автоматический
await dataSource.transaction(async (manager) => {
  await manager.update(Account, { id: 1 }, { balance: () => 'balance - 10' });
  await manager.update(Account, { id: 2 }, { balance: () => 'balance + 10' });
  // исключение внутри колбэка -> автоматический ROLLBACK, наружу пробрасывается ошибка
});

// Prisma — интерактивная транзакция
await prisma.$transaction(async (tx) => {
  await tx.account.update({ where: { id: 1 }, data: { balance: { decrement: 10 } } });
  await tx.account.update({ where: { id: 2 }, data: { balance: { increment: 10 } } });
});
```

Частая ошибка — выполнять запрос через "внешний" репозиторий/клиент (`orderRepo`, `prisma` напрямую) внутри колбэка транзакции вместо переданного `manager`/`tx` — такой запрос выполнится **вне** транзакции, в отдельном соединении, и не увидит незакоммиченные изменения (или, хуже, создаст независимую гонку). Prisma отдельно даёт `$transaction([...])` — batch-вариант для независимых операций без интерактивной логики между ними, дешевле по накладным расходам, если ветвления по результатам не нужны.

## Миграции

Обе экосистемы поддерживают разный подход: TypeORM исторически даёт и auto-generate (`typeorm migration:generate` — сравнивает entities со схемой БД и генерирует diff), и `synchronize: true` (авто-применение схемы без миграций — **категорически** не для прода, риск потери данных при расхождении). Prisma строится вокруг декларативной schema.prisma + `prisma migrate dev` (генерирует SQL-файл миграции + применяет) — миграции всегда явные файлы в git, ревьюабельные до применения. Общее правило для собеседования: `synchronize`/аналоги — только для локальной разработки и прототипов, в проде — только явные, ревьюенные, версионированные миграции, применяемые отдельным шагом деплоя (не при старте приложения под нагрузкой).

## Что спрашивают на собеседовании

1. **Что такое N+1 и как его обнаружить?** — один запрос за списком плюс по одному за каждой связью на элемент; обнаруживается по логам SQL/APM — число запросов растёт линейно с размером выборки.
2. **Как починить N+1 в TypeORM/Prisma?** — `relations`/`include` для join'а связи в один запрос, либо явный батч через `IN` по собранным ключам, если джойн раздувает объём данных.
3. **Чем плох `eager: true` на связи?** — связь тянется в каждом запросе через эту сущность, даже там, где не нужна — незаметная деградация по всей кодовой базе, а не в одном конкретном месте.
4. **Почему `synchronize: true` / `db push` без ревью миграций опасны в проде?** — база сама решает, как привести схему к текущим entities — может выполнить деструктивную операцию (drop column/table) без явного подтверждения человеком.
5. **В чём частая ошибка при использовании транзакций в ORM?** — выполнение части запросов через клиент/репозиторий вне переданного в колбэк `manager`/`tx` — эти запросы окажутся вне транзакции.

## Ссылки

- [TypeORM — Relations (eager/lazy, relations option)](https://typeorm.io/relations)
- [TypeORM — Transactions](https://typeorm.io/transactions)
- [Prisma — Query optimization / solving N+1](https://www.prisma.io/docs/orm/prisma-client/queries/query-optimization-performance)
- [Prisma — Transactions](https://www.prisma.io/docs/orm/prisma-client/queries/transactions)
- [Prisma — Migrate (workflows)](https://www.prisma.io/docs/orm/prisma-migrate)
