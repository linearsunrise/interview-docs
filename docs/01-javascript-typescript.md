---
sidebar_position: 1
title: JavaScript / TypeScript
---

# JavaScript / TypeScript

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
