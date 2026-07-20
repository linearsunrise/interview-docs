---
sidebar_position: 7
title: "Generators, iterators, for await...of"
---

# Generators, iterators, `for await...of`

> **TL;DR:** Итератор — любой объект с методом `.next()`, возвращающим `{value, done}`. Итерируемый объект — тот, у кого есть `[Symbol.iterator]`, возвращающий итератор (это то, что понимает `for...of`, spread, деструктуризация). Генератор — функция-фабрика итераторов: `yield` приостанавливает выполнение, сохраняя весь стек вызова, до следующего `.next()`. Асинхронный вариант — `[Symbol.asyncIterator]` + `for await...of`, на этом построены Node-стримы.

## Итератор и итерируемый объект вручную

```js
const range = {
  [Symbol.iterator]() {
    let i = 0;
    return {
      next: () => (i < 3 ? { value: i++, done: false } : { value: undefined, done: true }),
    };
  },
};
[...range]; // [0, 1, 2] — spread просто дёргает Symbol.iterator
for (const x of range) console.log(x); // то же самое под капотом
```

## Генератор — тот же протокол, но без ручного `next` каждый раз

```js
function* range(start, end) {
  for (let i = start; i < end; i++) yield i;
}
[...range(0, 3)]; // [0, 1, 2] — generator function автоматически реализует Symbol.iterator
```

`yield` — не `return`: функция запоминает, где остановилась, и продолжает с этого места при следующем `.next()`. `.next(arg)` может передать значение **внутрь** генератора — оно становится результатом выражения `yield`:

```js
function* echo() {
  const x = yield 'ready'; // получит значение из next(x)
  console.log('got', x);
}
const g = echo();
g.next();        // {value: 'ready', done: false}
g.next('hello'); // печатает "got hello"
```

`yield*` делегирует другому итерируемому — удобно для композиции генераторов без ручного цикла.

## Ленивые бесконечные последовательности

```js
function* naturals() {
  let n = 1;
  while (true) yield n++;
}
function take(iterable, n) {
  const result = [];
  for (const x of iterable) { if (result.length >= n) break; result.push(x); }
  return result;
}
take(naturals(), 5); // [1,2,3,4,5] — считает ровно столько, сколько взяли
```

## Асинхронные генераторы и `for await...of`

```js
async function* paginate(fetchPage) {
  let cursor = null;
  do {
    const { items, nextCursor } = await fetchPage(cursor);
    yield* items;
    cursor = nextCursor;
  } while (cursor);
}

for await (const item of paginate(fetchPage)) {
  process(item); // страницы подгружаются лениво, по одной, без загрузки всего в память
}
```

Это ровно тот протокол, на котором в Node с 10-й версии построены `Readable`-стримы — их можно напрямую обходить через `for await...of` без событий `data`/`end` (см. [02-nodejs-core/03-streams.md](../02-nodejs-core/03-streams.md)).

## Что спрашивают на собеседовании

1. **Чем итератор отличается от итерируемого объекта?** — итерируемый отдаёт итератор через `Symbol.iterator`; итератор — конкретный объект с `.next()`. Генератор реализует оба протокола автоматически.
2. **Как `for...of` работает "под капотом"?** — вызывает `obj[Symbol.iterator]()`, затем крутит `.next()` пока `done !== true`.
3. **Чем генератор отличается от обычной функции?** — приостанавливаемое выполнение: `yield` сохраняет весь контекст вызова (аналог coroutine), обычная функция всегда отрабатывает от начала до конца за один заход.
4. **Как сделать бесконечную ленивую последовательность?** — генератор с `while(true)` внутри — работает, пока кто-то не перестанет вызывать `.next()`.
5. **Как читать асинхронный источник данных постранично без загрузки всего в память?** — асинхронный генератор + `for await...of`, `yield*` для отдачи элементов страницы.

## Ссылки

- [MDN — Iteration protocols](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Iteration_protocols)
- [MDN — function*](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/function*) и [`for await...of`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/for-await...of)
- [ECMA-262 — Generator Objects](https://tc39.es/ecma262/#sec-generator-objects)
- [Streams и backpressure в Node](../02-nodejs-core/03-streams.md) — Readable как async iterable на практике
