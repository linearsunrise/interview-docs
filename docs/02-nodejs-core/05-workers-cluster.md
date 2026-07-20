---
sidebar_position: 5
title: worker_threads / child_process / cluster
---

# worker_threads vs child_process vs cluster

> **TL;DR:** worker_threads — потоки в одном процессе (общий SharedArrayBuffer, дёшево) для CPU-bound задач. child_process — отдельный процесс (изоляция, свой V8) для запуска чужих программ или полной изоляции. cluster — обёртка над child_process для масштабирования HTTP на ядра; в k8s обычно не нужен — там масштабируют репликами подов.

## Сравнение

| | worker_threads | child_process | cluster |
|---|---|---|---|
| Что это | поток + свой V8 isolate | отдельный процесс | процессы + шаринг порта |
| Память | общая через `SharedArrayBuffer`; остальное — structured clone | полностью раздельная | раздельная |
| Коммуникация | `postMessage` (+ transfer без копий), `MessageChannel` | IPC (сериализация JSON/advanced), stdio | IPC + балансировка входящих соединений |
| Стоимость | средняя (~МБ на isolate) | высокая (целый процесс) | высокая |
| Кейс | CPU-bound: парсинг, сжатие, крипто, ML | exec внешних утилит (ffmpeg), sandbox | все ядра под HTTP без k8s |

## worker_threads: минимум для собеса

```js
// main.js
const worker = new Worker('./heavy.js', { workerData: input });
worker.on('message', (result) => ...);
worker.on('error', ...);

// heavy.js
const { parentPort, workerData } = require('node:worker_threads');
parentPort.postMessage(heavyCompute(workerData));
```

Нюансы: создание воркера дорогое → **пул** (piscina); `postMessage` **копирует** данные (structured clone) — большие объёмы передавать через `transferList` (ArrayBuffer перемещается без копии) или `SharedArrayBuffer` + `Atomics`.

## cluster и почему он «умер» в k8s

`cluster.fork()` создаёт N процессов; primary принимает соединения и раздаёт воркерам round-robin (по умолчанию на Linux). Это решает «Node использует одно ядро» на голой VM. В Kubernetes проще и правильнее: 1 процесс = 1 под, CPU limit ~1, масштабирование репликами — оркестратор даёт рестарты, health checks и балансировку сам. PM2 cluster mode — то же самое для VM-мира.

## Что спрашивают на собеседовании

1. **CPU-bound задача в API — что делать?** — вынести в worker pool (piscina), либо в отдельный сервис/очередь; главный поток не должен считать.
2. **worker_threads vs child_process по памяти и коммуникации?** — см. таблицу; ключ: SharedArrayBuffer возможен только у потоков.
3. **Когда cluster не нужен?** — в k8s/оркестраторе: масштабирование репликами, изоляция и рестарты уже есть.
4. **Что копируется при `postMessage`?** — structured clone всего сообщения; исключения — transferable (ArrayBuffer, MessagePort) и SharedArrayBuffer.
5. **Почему не создавать воркер на каждый запрос?** — стоимость создания isolate; нужен пул с очередью задач.

## Ссылки

- [worker_threads — официальная документация](https://nodejs.org/api/worker_threads.html)
- [child_process](https://nodejs.org/api/child_process.html), [cluster](https://nodejs.org/api/cluster.html) (в начале — описание round-robin балансировки)
- [piscina](https://github.com/piscinajs/piscina) — стандарт де-факто для worker pool
- [Don't block the event loop — раздел про offloading](https://nodejs.org/en/learn/asynchronous-work/dont-block-the-event-loop)
