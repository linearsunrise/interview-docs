---
sidebar_position: 3
title: Streams и backpressure
---

# Streams и backpressure

> **TL;DR:** 4 типа: Readable, Writable, Duplex, Transform. Backpressure: `write()` вернул `false` → буфер полон (`highWaterMark`) → жди `'drain'`. Игнорируешь — память растёт до OOM. `pipeline()` вместо `pipe()`: сам управляет backpressure, пробрасывает ошибки и уничтожает все стримы.

## Зачем стримы

Отдать файл 10 ГБ: `fs.readFile` загрузит его в память целиком (и упадёт — у Buffer есть лимит). Стрим держит в памяти только `highWaterMark` (по умолчанию 64 КБ для байтовых, 16 объектов в objectMode):

```js
import { pipeline } from 'node:stream/promises';
await pipeline(
  fs.createReadStream('huge.csv'),
  zlib.createGzip(),
  res, // http.ServerResponse — тоже Writable
);
```

## Backpressure руками

```js
function write(stream, chunk) {
  if (!stream.write(chunk)) {                       // буфер переполнен
    return new Promise(r => stream.once('drain', r)); // ждём слива
  }
}
```

`write()` не «отказывает» — он **всегда принимает** chunk, но возвращает `false`, сигналя «притормози». Продолжать писать можно, но буфер (память процесса) растёт неограниченно. Классический сценарий OOM: быстрый Readable (файл с диска) → медленный Writable (сеть/БД).

## `pipe` vs `pipeline`

| | `a.pipe(b)` | `pipeline(a, b, cb)` |
|---|---|---|
| Backpressure | да | да |
| Ошибка в источнике | b **не** закрывается → утечка дескрипторов | все стримы destroy |
| Ошибки | слушать на каждом стриме отдельно | один колбэк/promise |

Поэтому `pipe` в проде — красный флаг; `pipeline` (или `stream/promises`) — стандарт.

## Transform — рабочая лошадка

```js
const csvToJson = new Transform({
  objectMode: true,
  transform(chunk, enc, cb) {
    try { cb(null, JSON.stringify(parseRow(chunk)) + '\n'); }
    catch (e) { cb(e); }
  },
});
```

Ещё знать: `Readable.from(asyncIterable)`, итерирование `for await (const chunk of readable)`, `objectMode`, и что с Node 17+ есть Web Streams (`ReadableStream`) — другой API, интероп через `Readable.toWeb/fromWeb`.

## Что спрашивают на собеседовании

1. **Что такое backpressure и что будет при игнорировании?** — сигнал «потребитель не успевает»; игнор = внутренний буфер Writable растёт → OOM.
2. **Как отдать файл 10 ГБ?** — `createReadStream` + `pipeline` в response; памяти — на один highWaterMark.
3. **Чем `pipeline` лучше `pipe`?** — при ошибке уничтожает всю цепочку (нет утечки fd/памяти), единая обработка ошибок.
4. **Что такое `highWaterMark`?** — порог буферизации (не жёсткий лимит!), после которого `write()` возвращает false / Readable перестаёт читать источник.
5. **Duplex vs Transform?** — Duplex: независимые read/write стороны (TCP-сокет); Transform: выход — функция входа (gzip).

## Ссылки

- [Stream — официальная документация](https://nodejs.org/api/stream.html), особенно раздел [Backpressuring in Streams](https://nodejs.org/en/learn/modules/backpressuring-in-streams)
- [`stream/promises` pipeline](https://nodejs.org/api/stream.html#streampipelinesource-transforms-destination-options)
- Исходники: [`lib/internal/streams/pipeline.js`](https://github.com/nodejs/node/blob/main/lib/internal/streams/pipeline.js) — видно, как достигается destroy всей цепочки; [`lib/internal/streams/writable.js`](https://github.com/nodejs/node/blob/main/lib/internal/streams/writable.js) — механика highWaterMark
- [Web Streams API в Node](https://nodejs.org/api/webstreams.html)
