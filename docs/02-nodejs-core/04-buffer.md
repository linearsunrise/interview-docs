---
sidebar_position: 4
title: Buffer и бинарные данные
---

# Buffer и бинарные данные

> **TL;DR:** Buffer — это `Uint8Array` поверх памяти вне V8-heap (считается в `external`). Главное на собесе: `alloc` (обнулённый) vs `allocUnsafe` (быстрый, но с мусором — риск утечки чужих данных), кодировки и чтение бинарных протоколов через `read*BE/LE`.

## Аллокация

```js
Buffer.alloc(1024);        // обнулён — безопасный дефолт
Buffer.allocUnsafe(1024);  // не обнулён: может содержать СТАРЫЕ данные процесса
Buffer.from('привет', 'utf8');
Buffer.from(arrayBuffer, offset, len); // view без копирования!
```

`allocUnsafe` быстрее (память из внутреннего пула 8 КБ, без zero-fill), но отдать такой буфер наружу не заполнив целиком = отдать случайные куски памяти процесса (пароли, ключи). Незаполненный `allocUnsafe`, ушедший в ответ, — известный класс уязвимостей.

## Кодировки

```js
buf.toString('utf8' | 'base64' | 'base64url' | 'hex' | 'latin1');
Buffer.byteLength('привет', 'utf8'); // 12, а .length строки — 6!
```

Ловушка: chunk'и стрима могут резать многобайтовый UTF-8 символ посередине — `chunk.toString()` даст «кракозябры». Решение: `string_decoder` или задать encoding у стрима.

## Бинарные протоколы

```js
// парсим заголовок: [4 байта length BE][1 байт type][payload]
const length = buf.readUInt32BE(0);
const type = buf.readUInt8(4);
const payload = buf.subarray(5, 5 + length); // view, НЕ копия
```

`subarray` возвращает представление той же памяти (изменения видны в обоих), копия — `Buffer.copyBytesFrom` / `Buffer.from(buf)`. Держать `subarray` от большого буфера = держать в памяти весь большой буфер (та же ловушка, что со `slice` строк).

## Что спрашивают на собеседовании

1. **`alloc` vs `allocUnsafe`?** — zero-fill vs мусор из пула; unsafe только когда тут же перезаписываешь целиком.
2. **Где живёт память Buffer?** — вне V8-heap (`external` в `process.memoryUsage()`); поэтому «heap в норме, а RSS растёт» часто указывает на буферы.
3. **Почему `str.length !== Buffer.byteLength(str)`?** — length считает UTF-16 code units, byteLength — байты в UTF-8.
4. **Как распарсить бинарный протокол по TCP?** — аккумулировать chunks (сообщение может прийти кусками или склеенным!), читать length-prefix, `read*BE/LE`.
5. **`subarray` копирует данные?** — нет, это view; и в этом и сила (zero-copy), и источник утечек.

## Ссылки

- [Buffer — официальная документация](https://nodejs.org/api/buffer.html), особенно [раздел про `allocUnsafe` и пул](https://nodejs.org/api/buffer.html#static-method-bufferallocunsafesize)
- [`string_decoder`](https://nodejs.org/api/string_decoder.html) — корректная сборка UTF-8 из чанков
- [DEP0005: почему конструктор `new Buffer()` deprecated](https://nodejs.org/api/deprecations.html#DEP0005) — уязвимость раскрытия памяти, из-за которой появились alloc/allocUnsafe
- Исходники: [`lib/buffer.js`](https://github.com/nodejs/node/blob/main/lib/buffer.js) — пул и аллокация
