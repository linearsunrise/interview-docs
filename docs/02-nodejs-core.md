---
sidebar_position: 2
title: Node.js Core
---

# Node.js Core

## Чеклист знаний

- [ ] Архитектура: V8 + libuv, что уходит в thread pool (fs, dns, crypto, zlib), а что в OS async (сеть)
- [ ] Фазы event loop: timers → pending → poll → check → close; где живут setImmediate и setTimeout
- [ ] Streams: 4 типа, backpressure, `highWaterMark`, `pipeline` vs `pipe`
- [ ] Buffer: аллокация, encoding, работа с бинарными протоколами
- [ ] Worker threads vs child_process vs cluster — когда что
- [ ] CommonJS vs ESM: различия резолва, interop, top-level await
- [ ] Память: структура heap V8, поколения GC, `--max-old-space-size`
- [ ] Поиск утечек: heap snapshot, `process.memoryUsage()`, clinic.js
- [ ] Обработка ошибок: `uncaughtException`, `unhandledRejection`, домены (почему deprecated)
- [ ] Graceful shutdown: SIGTERM, закрытие сервера, drain соединений, таймаут
- [ ] AsyncLocalStorage — контекст запроса без передачи через параметры
- [ ] `http` изнутри: keep-alive, агенты, таймауты

## Вопросы с собеседований

1. Node.js однопоточный — почему тогда он обрабатывает тысячи соединений?
2. Что попадает в libuv thread pool? Почему `crypto.pbkdf2` может «затормозить» приложение и как это чинить? (`UV_THREADPOOL_SIZE`, воркеры)
3. `setImmediate` vs `setTimeout(fn, 0)` — когда порядок гарантирован, а когда нет?
4. Что такое backpressure и что будет, если его игнорировать? Как `pipeline` решает проблему ошибок и утечек по сравнению с `pipe`?
5. Как отдать файл в 10 ГБ клиенту, не убив память?
6. Cluster vs worker_threads — чем отличаются по памяти и коммуникации? Когда cluster не нужен (k8s)?
7. Как найти утечку памяти в проде? Расскажите реальный кейс.
8. Что делать в обработчике `uncaughtException` — и почему нельзя просто «проглотить» и продолжить?
9. Как реализовать graceful shutdown: что закрывать и в каком порядке?
10. Зачем AsyncLocalStorage, как через него делают request-id в логах?

## Ключевые тезисы

**Однопоточность:** один поток — это JS-код. I/O делегируется в OS (epoll/kqueue для сети) или thread pool (fs, crypto). Блокирует всё только синхронный CPU-bound JS.

**Backpressure:** `writable.write()` возвращает `false`, когда буфер превысил `highWaterMark` — надо ждать `drain`. Игнорирование = рост памяти до OOM. `pipeline()` управляет этим сам + корректно уничтожает все стримы при ошибке.

**Graceful shutdown (порядок):** перестать принимать новые соединения (`server.close`) → дождаться активных запросов (с таймаутом) → закрыть коннекты к БД/брокерам → `process.exit`. В k8s помнить про lag до снятия пода из endpoints — нужен небольшой delay перед закрытием.

**uncaughtException:** состояние процесса уже неконсистентно — залогировать, отправить в Sentry, завершить процесс; рестарт — забота оркестратора.

## Красные флаги

- «Node многопоточный, потому что есть thread pool» (без понимания, что именно там)
- Читает файл через `fs.readFile` целиком для отдачи клиенту
- «unhandledRejection можно игнорировать»

## Практика

- Написать transform-стрим (например, CSV → JSON) и прогнать файл на пару ГБ через `pipeline`
- Воспроизвести утечку (глобальный массив/подписки) и найти её через два heap snapshot в Chrome DevTools
- Реализовать graceful shutdown в тестовом Nest-приложении и проверить под нагрузкой (autocannon + SIGTERM)
