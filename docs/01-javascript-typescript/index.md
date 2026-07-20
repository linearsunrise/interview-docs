---
sidebar_position: 1
title: JavaScript / TypeScript
---

# JavaScript / TypeScript

## Подтемы

1. [Event loop и очереди задач](./01-event-loop-microtasks.md) — микро/макротаски, порядок выполнения, `async/await` как сахар над промисами
2. [Замыкания и лексическое окружение](./02-closures-scope.md) — `[[Environment]]`, `var` vs `let` в цикле, приватное состояние
3. [Прототипы и class](./03-prototypes-and-classes.md) — `[[Prototype]]`-цепочка, `class` как сахар, приватные поля `#x`
4. [`this` и способы привязки](./04-this-and-binding.md) — 4 правила, приоритет, потеря контекста в колбэках
5. [Promise изнутри](./05-promises-internals.md) — состояния, thenable resolution, unhandled rejection
6. [Promise.all / allSettled / race / any](./06-promise-combinators.md) — отличия, таймаут-паттерн, отмена через AbortController
7. [Generators, iterators, for await...of](./07-generators-iterators.md) — протоколы итерации, ленивые последовательности, async-генераторы
8. [WeakMap / WeakSet / WeakRef](./08-weakmap-weakset-weakref.md) — слабые ссылки, кэши без утечек, зачем не итерируются
9. [TS: generics и utility types](./09-ts-generics-utility-types.md) — constraints, как устроены Partial/Pick/Omit/Record
10. [TS: conditional, infer, mapped, template literal types](./10-ts-conditional-mapped-types.md) — distributive conditional types, DeepPartial, парсинг строк на уровне типов
11. [TS: type guards, discriminated unions, satisfies](./11-ts-type-guards-unions.md) — сужение типов, exhaustiveness checking, `satisfies` vs аннотация
12. [TS: декораторы и reflect-metadata](./12-ts-decorators.md) — legacy vs standard-декораторы, откуда DI берёт типы параметров
13. [Structural typing, unknown vs any](./13-structural-typing.md) — duck typing, excess property check, branding, почему `any` заразен

## Чеклист знаний

- [ ] Event loop: фазы, микро/макротаски, порядок выполнения
- [ ] Замыкания и лексическое окружение
- [ ] Прототипное наследование, `class` как сахар
- [ ] `this`: 4 правила привязки, arrow functions
- [ ] Promise изнутри: состояния, chaining, unhandled rejection
- [ ] `Promise.all` / `allSettled` / `race` / `any` — отличия и кейсы применения
- [ ] Generators, итераторы, `for await...of`
- [ ] WeakMap/WeakSet, WeakRef — зачем нужны
- [ ] TS: generics с constraints, utility types (Pick, Omit, Partial, Record, ReturnType)
- [ ] TS: conditional types, `infer`, mapped types, template literal types
- [ ] TS: type guards, discriminated unions, `satisfies`
- [ ] TS: декораторы (важно — фундамент NestJS), `reflect-metadata`
- [ ] structural vs nominal typing, `unknown` vs `any`

## Вопросы с собеседований

1. Что выведет код с `setTimeout`, `Promise.resolve().then`, `queueMicrotask`, синхронным кодом? (дают сниппет)
2. Чем `process.nextTick` отличается от `queueMicrotask`?
3. Как работает замыкание? Классическая задача с `var` в цикле и `setTimeout`.
4. Чем `Promise.all` опасен и когда брать `allSettled`?
5. Как отменить Promise? (AbortController, паттерны отмены)
6. Разница `interface` vs `type`? Когда что?
7. Напишите utility type: сделать все поля объекта опциональными рекурсивно (DeepPartial).
8. Что такое discriminated union и зачем он в обработке ошибок/событий?
9. Как работают декораторы и `reflect-metadata`? Как NestJS через них получает типы для DI?
10. `unknown` vs `any` — почему `any` ломает типобезопасность транзитивно?

## Ключевые тезисы

**Event loop:** микротаски (Promise, nextTick) выполняются полностью после каждой макротаски и *до* рендера/следующей фазы. `nextTick` — отдельная очередь с приоритетом выше Promise-микротасок. Злоупотребление nextTick может «заморить голодом» (starve) I/O.

**Promise.all:** fail-fast — падает на первом reject, но остальные промисы *продолжают выполняться* (их нельзя отменить), просто результат игнорируется. Для батчей внешних вызовов чаще нужен `allSettled` + агрегация ошибок.

**Декораторы в Nest:** `emitDecoratorMetadata` заставляет TS записывать типы параметров конструктора в metadata (`design:paramtypes`) — именно оттуда DI-контейнер узнаёт, что инжектить.

## Красные флаги

- «Микротаски и макротаски — это одно и то же, просто очередь»
- Не может объяснить, зачем в проекте strict mode в TS
- Знает utility types, но не может написать простой mapped type сам

## Практика

- Решить 10 задач на порядок вывода event loop (сниппеты с `async/await` + таймеры)
- Написать DeepPartial, DeepReadonly, свой `Awaited` через `infer`
- Написать простейший декоратор метода с логированием через `reflect-metadata`
