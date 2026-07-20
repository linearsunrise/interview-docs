---
sidebar_position: 6
title: "Promise.all / allSettled / race / any"
---

# `Promise.all` / `allSettled` / `race` / `any`

> **TL;DR:** Все четыре принимают массив промисов и возвращают один промис — отличаются только тем, **когда** он settled и что содержит. `all` — fail-fast, но не отменяет остальные (они продолжают выполняться в фоне, результат просто игнорируется — источник "невидимых" запросов и лишней нагрузки).

## Сравнение

| Комбинатор | Резолвится когда | Реджектится когда | Результат |
|---|---|---|---|
| `Promise.all` | все fulfilled | первый reject (fail-fast) | массив значений, порядок как у входа |
| `Promise.allSettled` | всегда, когда все settled | никогда | массив `{status, value \| reason}` |
| `Promise.race` | первый settled (любой) | первый settled — если это reject | значение/причина победителя |
| `Promise.any` | первый fulfilled | все rejected | значение победителя / `AggregateError` |

## `Promise.all` — fail-fast, но без отмены

```js
const [user, orders] = await Promise.all([fetchUser(id), fetchOrders(id)]);
// если fetchOrders упадёт раньше — Promise.all реджектится немедленно,
// но fetchUser (если ещё не завершился) ВСЁ РАВНО долетит до конца — просто результат никто не заберёт
```

Для настоящей отмены нужен `AbortController`: сигнал прокидывается во все запросы, и при первом реджекте вручную вызывается `controller.abort()` для остальных.

## `allSettled` — когда нужны частичные результаты

```js
const results = await Promise.allSettled(ids.map(fetchUser));
const ok = results.filter((r) => r.status === 'fulfilled').map((r) => r.value);
const failed = results.filter((r) => r.status === 'rejected');
// один упавший запрос не рушит остальные — типично для батчевых операций / уведомлений
```

## `race` — паттерн таймаута

```js
function withTimeout(promise, ms) {
  const timeout = new Promise((_, reject) =>
    setTimeout(() => reject(new Error('timeout')), ms)
  );
  return Promise.race([promise, timeout]);
}
```

Важно: "проигравший" промис не отменяется — если это, например, `fetch`, запрос продолжит идти в фоне и съедать ресурсы, пока сам не завершится (или пока к нему не прикручен `AbortSignal`).

## `any` — первый успех, не первый ответ

Отличие от `race`: `any` игнорирует отклонённые промисы и ждёт первый **fulfilled**. Если отклонятся вообще все — реджект с `AggregateError`, содержащим массив всех причин в `.errors`. Практический кейс: запрос к нескольким зеркалам/репликам — берём первый, кто ответил успешно.

## Что спрашивают на собеседовании

1. **Чем `all` опасен, если один из промисов может упасть?** — реджектится сразу, теряя результаты остальных (хотя они всё равно выполнятся); нужен `allSettled`, если нужны частичные результаты.
2. **Останавливает ли `Promise.all` остальные промисы после первого reject?** — нет, JS не умеет отменять промисы сами по себе; нужен `AbortController`, промисы этого не делают "из коробки".
3. **Как реализовать таймаут для запроса?** — `Promise.race([запрос, promise-таймер])`; обсудить, что проигравший запрос не отменяется без `AbortSignal`.
4. **Чем `any` отличается от `race`?** — `race` берёт первый settled (fulfilled или rejected), `any` — именно первый fulfilled, игнорируя реджекты, пока не отвалятся все.
5. **Что за `AggregateError`?** — специальный тип ошибки с полем `.errors` — массив причин; кидается `Promise.any`, когда все входные промисы отклонены.

## Ссылки

- [MDN — Promise.all](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/all), [allSettled](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/allSettled), [race](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/race), [any](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/any)
- [MDN — AbortController](https://developer.mozilla.org/en-US/docs/Web/API/AbortController) — реальная отмена, промисы сами не умеют
- [TC39 — proposal-promise-any](https://github.com/tc39/proposal-promise-any), [proposal-promise-allSettled](https://github.com/tc39/proposal-promise-allSettled)
