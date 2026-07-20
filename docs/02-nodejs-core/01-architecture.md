---
sidebar_position: 1
title: Архитектура V8 + libuv
---

# Архитектура: V8 + libuv

> **TL;DR:** «Однопоточный» Node = один поток исполняет JS. Вся асинхронщина — у libuv: сеть идёт через механизмы ОС (epoll/kqueue/IOCP) и потоков не занимает, а fs/dns.lookup/crypto/zlib выполняются в thread pool (по умолчанию 4 потока). Блокирует приложение только синхронный CPU-bound JS — или забитый thread pool.

## Кто за что отвечает

- **V8** — исполняет JS, управляет памятью (heap, GC). Ничего не знает про I/O.
- **libuv** — event loop, кроссплатформенный асинхронный I/O, thread pool.
- **Bindings (C++)** — мост: `fs.readFile` из JS превращается в вызов libuv.

## Куда уходит какая операция

| Механизм | Операции |
|---|---|
| OS async (epoll/kqueue/IOCP) — **без потоков** | сеть: sockets, http, dns.resolve* (c-ares) |
| libuv thread pool (default 4) | `fs.*`, `dns.lookup` (!), `crypto.pbkdf2/scrypt/randomBytes`, `zlib.*` (async-версии) |
| Главный поток — **блокирует всё** | синхронный JS, `JSON.parse` гигантского объекта, `*.Sync`-функции, регэксп с катастрофическим бэктрекингом |

Классика собеса: `dns.lookup` (используется по умолчанию в `http.request` при коннекте по имени хоста) ходит в **thread pool**, а `dns.resolve` — в c-ares без пула. Забитый пул → «зависшие» DNS и fs.

## Почему тысячи соединений — не проблема

Соединение — это не поток, а файловый дескриптор + колбэки. 10к keep-alive сокетов почти ничего не стоят, пока по ним не идёт CPU-работа. Проблема Node — не много соединений, а **один тяжёлый запрос**, который блокирует event loop для всех.

## `UV_THREADPOOL_SIZE`

```bash
UV_THREADPOOL_SIZE=16 node app.js  # максимум 1024; выставлять до старта
```

Если приложение активно жмёт gzip/хеширует пароли/читает файлы — 4 потоков мало: задачи выстраиваются в очередь, латентность растёт скачками. Признак: медленные `fs`-операции при свободном CPU. Для системного CPU-bound — worker_threads, не раздувание пула.

## Что спрашивают на собеседовании

1. **Node однопоточный — как он держит тысячи соединений?** — сеть асинхронная на уровне ОС; поток один только для JS; соединение = дескриптор, не поток.
2. **Почему `crypto.pbkdf2` «тормозит» приложение?** — 4 задачи займут весь thread pool → следующие ждут; заодно встанут fs и dns.lookup. Лечение: `UV_THREADPOOL_SIZE`, вынос в worker_threads, rate limiting на login.
3. **`dns.lookup` vs `dns.resolve`?** — lookup → getaddrinfo → thread pool (и учитывает /etc/hosts); resolve → c-ares, без пула.
4. **Чем опасен большой `JSON.parse`?** — синхронный, в главном потоке; на десятках МБ — сотни мс блокировки всех запросов.

## Ссылки

- [Don't block the event loop — официальный гайд](https://nodejs.org/en/learn/asynchronous-work/dont-block-the-event-loop) — лучший текст по теме, стоит прочитать целиком
- [libuv design overview](https://docs.libuv.org/en/v1.x/design.html) — event loop и thread pool из первых рук
- [`UV_THREADPOOL_SIZE`](https://nodejs.org/api/cli.html#uv_threadpool_sizesize)
- [`dns.lookup` vs `dns.resolve` — implementation considerations](https://nodejs.org/api/dns.html#implementation-considerations)
- Исходники: [libuv `threadpool.c`](https://github.com/libuv/libuv/blob/v1.x/src/threadpool.c) — весь пул умещается в один файл
