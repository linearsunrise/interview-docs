---
sidebar_position: 2
title: Фазы event loop
---

# Фазы event loop

> **TL;DR:** Цикл: **timers → pending callbacks → poll → check → close callbacks**. `setTimeout` живёт в timers, `setImmediate` — в check. Микротаски (`process.nextTick`, затем promises) выполняются **после каждого колбэка**, а не между фазами.

## Схема

```
┌─> timers          setTimeout / setInterval
│   pending         отложенные системные колбэки (напр., ошибки TCP)
│   idle, prepare   (внутреннее)
│   poll            ← здесь loop проводит большую часть времени:
│                     I/O-колбэки; ждёт события, если нечего делать
│   check           setImmediate
└── close           socket.on('close')

между ЛЮБЫМИ двумя колбэками: nextTick queue → microtask queue (promises)
```

## Главные факты для собеса

**`setTimeout(fn, 0)` vs `setImmediate(fn)` в главном модуле — порядок не гарантирован.** `setTimeout(fn, 0)` это на самом деле `setTimeout(fn, 1)`; если процесс стартовал быстрее 1мс — timers-фаза пуста, setImmediate успеет первым, иначе наоборот.

**Внутри I/O-колбэка порядок гарантирован: `setImmediate` первым.** После poll-фазы (где выполнился I/O-колбэк) следующая фаза — check; до timers цикл дойдёт только на новой итерации.

```js
fs.readFile(__filename, () => {
  setTimeout(() => console.log('timeout'), 0);
  setImmediate(() => console.log('immediate')); // всегда раньше
});
```

**`process.nextTick` — не часть event loop.** Его очередь дренится сразу после текущей операции, раньше promise-микротасков. Рекурсивный `nextTick` **голодает** event loop (I/O никогда не наступит); рекурсивный `setImmediate` — нет (даёт циклу крутиться). Официальная позиция доков: в новом коде предпочитайте `setImmediate`.

**Порядок микротасков:** синхронный код → вся nextTick-очередь → вся promise-очередь → следующий колбэк/фаза.

```js
setTimeout(() => console.log('timeout'));
setImmediate(() => console.log('immediate'));
Promise.resolve().then(() => console.log('promise'));
process.nextTick(() => console.log('nextTick'));
// nextTick, promise, затем timeout/immediate (их порядок — лотерея в main)
```

## Как это связано с производительностью

«Лаг event loop» (сколько таймер опаздывает) — ключевая метрика здоровья Node-сервиса: [`perf_hooks.monitorEventLoopDelay()`](https://nodejs.org/api/perf_hooks.html#perf_hooksmonitoreventloopdelayoptions). Рост лага = кто-то блокирует поток; алертить на p99.

## Что спрашивают на собеседовании

1. **Перечислите фазы и что в каждой** — см. схему; poll — центральная: там I/O и там loop «спит».
2. **`setImmediate` vs `setTimeout(0)` — когда порядок гарантирован?** — внутри I/O-колбэка immediate всегда раньше; в главном модуле — недетерминирован.
3. **Чем опасен рекурсивный `process.nextTick`?** — starvation: очередь дренится до конца перед возвратом в цикл, I/O не выполняется.
4. **Где выполняются промис-колбэки?** — микротаск-очередь после каждого колбэка (с Node 11 — так и в браузере), не отдельная фаза.
5. **Как замерить, что event loop блокируется?** — `monitorEventLoopDelay`, метрика в Prometheus, профилирование `--cpu-prof`.

## Ссылки

- [The Node.js Event Loop, Timers, and `process.nextTick()`](https://nodejs.org/en/learn/asynchronous-work/event-loop-timers-and-nexttick) — канонический документ, с него списаны все статьи
- [libuv design overview](https://docs.libuv.org/en/v1.x/design.html)
- [Изменение порядка микротасков в Node 11](https://github.com/nodejs/node/pull/22842) — PR, выровнявший поведение с браузером
- Исходники: [libuv `core.c` — `uv_run()`](https://github.com/libuv/libuv/blob/v1.x/src/unix/core.c) — сам цикл с фазами, читается сверху вниз
