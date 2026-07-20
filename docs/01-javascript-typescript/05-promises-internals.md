---
sidebar_position: 5
title: Promise изнутри
---

# Promise изнутри

> **TL;DR:** Promise — конечный автомат с тремя состояниями (`pending → fulfilled` или `pending → rejected`, необратимо и один раз). `executor` в конструкторе выполняется **синхронно**. `.then()` всегда возвращает **новый** промис — отсюда цепочки. Если резолвить промис значением с `.then` (thenable), движок гоняется за ним через "thenable resolution procedure" — это добавляет лишние тики микротасков, частый источник неожиданного порядка выполнения.

## Состояния и неизменность

```js
const p = new Promise((resolve, reject) => {
  console.log('executor'); // синхронно, сразу при создании
  resolve(1);
  resolve(2); // no-op — состояние уже settled, повторные вызовы игнорируются
});
```

Как только промис перешёл в `fulfilled`/`rejected` — это навсегда, значение/причина зафиксированы. Именно поэтому Promise — надёжный примитив для кэширования результата асинхронной операции: можно повесить `.then()` хоть через час после резолва, он выполнится немедленно (в следующем микротаске) с уже готовым значением.

## Почему `.then()` возвращает новый промис

```js
const p1 = Promise.resolve(1);
const p2 = p1.then((v) => v + 1);
p1 === p2; // false
```

Это то, что делает `.then().then().catch()` осмысленной цепочкой: каждый `.then` — самостоятельный узел с собственным состоянием, которое резолвится результатом колбэка (или пробрасывает исключение как rejection следующего звена). Частая ошибка — забыть `return` внутри `.then()`: следующее звено получит `undefined` вместо ожидаемого значения.

## Thenable resolution — почему лишний тик

```js
Promise.resolve({ then(resolve) { resolve(42); } })
  .then((v) => console.log(v)); // 42, но не за один тик — движок сначала распознаёт thenable
```

Если `resolve(x)` или возврат из `.then()` — объект с методом `.then`, спецификация обязывает движок сначала "подписаться" на этот thenable (это отдельная микротаска), и только потом продолжить цепочку. На практике это значит: `await` промиса, обёрнутого во внешний thenable (частый случай — библиотеки со своей promise-like реализацией), может занять на один тик больше, чем `await` нативного промиса.

## unhandled rejection — как это отслеживается

Движок помечает промис "unhandled", если к моменту завершения текущего чекпоинта микротасков к нему не был привязан rejection-обработчик. Если обработчик добавить позже (но всё ещё в том же тике) — событие может не сработать; если после — сработает `unhandledRejection` (Node) / `unhandledrejection` (браузер). Что делать с ним в Node и почему нельзя просто проигнорировать — в [02-nodejs-core/09-error-handling.md](../02-nodejs-core/09-error-handling.md).

```js
async function risky() { throw new Error('boom'); }
risky(); // необработанный reject — async-функция всегда возвращает Promise
```

## Что спрашивают на собеседовании

1. **Когда выполняется executor — синхронно или асинхронно?** — синхронно, сразу в момент `new Promise(...)`; асинхронна только доставка результата через `.then`.
2. **Почему `.then()` создаёт новый промис, а не переиспользует исходный?** — чтобы цепочки были композируемыми: у каждого звена своё состояние, ошибка в колбэке становится rejection именно этого звена.
3. **Что будет, если резолвить промис другим промисом/thenable?** — состояние "унаследуется" от вложенного, но через дополнительный тик микротасков — thenable resolution procedure.
4. **Как определяется unhandled rejection?** — если на момент окончания текущего чекпоинта микротасков у отклонённого промиса нет обработчика — событие `unhandledRejection`.
5. **Что возвращает `async function`, если внутри нет `return`?** — всё равно Promise, резолвленный `undefined`; исключение внутри — reject, даже без явного `throw` наружу.

## Ссылки

- [MDN — Promise](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise)
- [Jake Archibald — Tasks, microtasks, queues and schedules](https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/)
- [ECMA-262 — Promise Objects](https://tc39.es/ecma262/#sec-promise-objects) и [`PerformPromiseThen`](https://tc39.es/ecma262/#sec-performpromisethen)
- [Обработка ошибок в Node: `unhandledRejection`](../02-nodejs-core/09-error-handling.md)
