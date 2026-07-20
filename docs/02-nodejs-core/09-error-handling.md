---
sidebar_position: 9
title: Обработка ошибок процесса
---

# Обработка ошибок: uncaughtException и unhandledRejection

> **TL;DR:** `uncaughtException` = процесс в неопределённом состоянии: залогировать → graceful exit → рестарт оркестратором. «Проглотить и продолжить» нельзя. `unhandledRejection` с Node 15 по умолчанию **роняет процесс** так же, как uncaughtException. Domains deprecated именно потому, что обещали «продолжить безопасно», но не могли этого гарантировать.

## Почему нельзя продолжать после uncaughtException

Исключение вылетело из произвольного места: неизвестно, какие инварианты нарушены — залочен ли мьютекс, недописан ли файл, потерян ли ответ клиенту, жив ли коннект к БД. Продолжение = утечки ресурсов и коррупция данных с отложенными, неотлаживаемыми симптомами. Официальная документация прямо говорит: использовать только для синхронной очистки, затем завершаться.

```js
process.on('uncaughtException', (err, origin) => {
  logger.fatal({ err, origin }, 'uncaught exception');
  // отправить в Sentry, flush логов — и умереть
  process.exitCode = 1;
  server.close(() => process.exit(1));
  setTimeout(() => process.exit(1), 5000).unref(); // страховка
});

process.on('unhandledRejection', (reason) => {
  throw reason; // единообразно превращаем в uncaughtException
});
```

Рестарт — забота внешнего супервизора (k8s, systemd, PM2), не самого процесса.

## Карта ошибок: что где ловится

| Источник | Как ловить |
|---|---|
| sync throw в запросе | try/catch, в Nest — exception filter |
| rejected promise в запросе | await + try/catch / фильтры |
| колбэк-API | первый аргумент `(err, data)`; throw в колбэке = uncaughtException! |
| EventEmitter | событие `'error'`; **без слушателя — краш процесса** (частый факап со стримами) |
| фоновая задача / setInterval | свой try/catch внутри; иначе — uncaughtException |

## Почему domains — deprecated

`domain` обещал перехватывать все ошибки в «домене» запроса и продолжать работу. Проблемы: после ошибки состояние всё равно неконсистентно (та же причина, что выше), утечки контекста между запросами, неявная магия поверх event loop. Модуль заморожен как deprecated; современная замена для контекста — `AsyncLocalStorage`, для ошибок — честные try/catch + фатальные хендлеры.

## Что спрашивают на собеседовании

1. **Что делать в `uncaughtException` и почему нельзя игнорировать?** — лог + телеметрия + завершение; состояние неконсистентно; рестарт — оркестратор.
2. **Что происходит с необработанным rejection?** — с Node 15 — краш (`--unhandled-rejections=throw` по умолчанию); раньше — warning, что маскировало баги.
3. **Почему `emitter.emit('error')` без слушателя роняет процесс?** — спецповедение EventEmitter; поэтому у стримов всегда вешаем on('error') или используем pipeline.
4. **Почему domains убрали?** — не могли гарантировать консистентность после перехвата; сложная семантика; заменены ALS + обычной обработкой.
5. **Как не терять стек в async-коде?** — не отрывать промисы (`floating promise`), eslint `no-floating-promises`, `--async-stack-traces` включён по умолчанию в современных Node.

## Ссылки

- [`process` events: `uncaughtException`, `unhandledRejection`](https://nodejs.org/api/process.html#event-uncaughtexception) — включая официальное «Warning: not a safety net»
- [CLI `--unhandled-rejections`](https://nodejs.org/api/cli.html#--unhandled-rejectionsmode) и [PR nodejs/node#33021 — throw по умолчанию в Node 15](https://github.com/nodejs/node/pull/33021)
- [Domain — deprecation](https://nodejs.org/api/domain.html)
- [Error handling — гайд Joyent](https://web.archive.org/web/2023/https://www.joyent.com/node-js/production/design/errors) — классика про operational vs programmer errors, на неё до сих пор ссылаются
