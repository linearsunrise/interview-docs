---
sidebar_position: 12
title: HTTP изнутри
---

# `http` изнутри: keep-alive, агенты, таймауты

> **TL;DR:** Keep-alive экономит TCP/TLS-хендшейки, но рождает две классические проблемы: (1) исходящие запросы без агента с keep-alive медленные; (2) случайные 502 за балансировщиком, когда `keepAliveTimeout` сервера **меньше** idle timeout LB. Таймауты по умолчанию надо знать и почти всегда перенастраивать.

## Клиентская сторона: Agent

`http.Agent` управляет пулом сокетов: `keepAlive: true` переиспользует соединения, `maxSockets` ограничивает параллелизм на хост (по умолчанию Infinity), `maxFreeSockets` — сколько простаивающих держать. С Node 19 глобальный агент создаётся с `keepAlive: true` — раньше каждый `fetch`-подобный запрос без агента открывал новый TCP+TLS (десятки мс накладных).

```js
const agent = new http.Agent({ keepAlive: true, maxSockets: 50 });
http.request({ agent, ... });
```

Современная альтернатива — [undici](https://github.com/nodejs/undici) (на нём построен глобальный `fetch` в Node): свой пул, pipelining, заметно быстрее `http.request`.

## Серверная сторона: три таймаута

| Таймаут | Default | Что делает |
|---|---|---|
| `server.keepAliveTimeout` | 5 c | сколько держать idle keep-alive сокет |
| `server.headersTimeout` | 60 c | сколько ждать полные заголовки (защита от Slowloris) |
| `server.requestTimeout` | 300 c | максимум на весь запрос (с Node 18 включён) |

**Классика прода — 502 за ALB/nginx:** LB держит idle-соединение к Node (например, 60 c у ALB), Node закрывает его через свои 5 c; LB отправляет очередной запрос в уже умирающий сокет → 502/ECONNRESET. Лечение: `keepAliveTimeout` **больше** idle timeout балансировщика (и `headersTimeout` больше `keepAliveTimeout`).

```js
server.keepAliveTimeout = 65_000;
server.headersTimeout = 66_000;
```

## Что ещё знать

- **По умолчанию у исходящего `http.request` нет таймаута вообще** — зависший апстрим держит сокет вечно; всегда задавать (`AbortSignal.timeout(5000)` для fetch/undici).
- Заголовок `Connection: close` / `Connection: keep-alive` — кто управляет жизнью соединения; HTTP/1.1 keep-alive по умолчанию.
- `Transfer-Encoding: chunked` — стриминг ответа неизвестной длины; именно так работает `res.write()` до `end()`.
- Сокеты — это стримы: response — Writable с backpressure (связь с темой [streams](./03-streams.md)).
- HTTP/2 (`node:http2`): мультиплексирование стримов в одном соединении — половина проблем keep-alive исчезает, но в типовом сетапе h2 терминируется на LB, а до Node идёт h1.

## Что спрашивают на собеседовании

1. **Откуда случайные 502 за балансировщиком?** — `keepAliveTimeout` (5 c) меньше idle timeout LB; race на переиспользовании закрываемого сокета.
2. **Зачем нужен Agent?** — пул и переиспользование соединений, лимиты параллелизма; без keep-alive каждый запрос платит TCP+TLS handshake.
3. **Какие таймауты выставить у исходящих запросов?** — connect/response/общий; в Node руками (AbortSignal), дефолтов нет.
4. **Slowloris — что это и чем защищён Node?** — клиент шлёт заголовки по байту, держа сокеты; `headersTimeout`/`requestTimeout` + LB.
5. **Чем undici быстрее `http.request`?** — свой клиент без legacy-слоёв, pipelining, эффективный пул; это и есть `fetch` в Node.

## Ссылки

- [HTTP — официальная документация](https://nodejs.org/api/http.html): [`keepAliveTimeout`](https://nodejs.org/api/http.html#serverkeepalivetimeout), [`headersTimeout`](https://nodejs.org/api/http.html#serverheaderstimeout), [`requestTimeout`](https://nodejs.org/api/http.html#serverrequesttimeout), [Agent](https://nodejs.org/api/http.html#class-httpagent)
- [undici](https://github.com/nodejs/undici) — и [почему fetch в Node построен на нём](https://nodejs.org/en/learn/getting-started/fetch)
- [AWS: разбор idle timeout ALB и таргетов](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/application-load-balancers.html#connection-idle-timeout) — первоисточник по проблеме 502
- Исходники: [`lib/_http_agent.js`](https://github.com/nodejs/node/blob/main/lib/_http_agent.js) — пул сокетов и keep-alive логика
