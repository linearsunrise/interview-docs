---
sidebar_position: 13
title: "Structural typing, unknown vs any"
---

# Structural vs nominal typing, `unknown` vs `any`

> **TL;DR:** TypeScript сравнивает типы **по форме** (structural typing / duck typing) — совпадают поля и их типы, значит типы совместимы, независимо от названия и места объявления. Это отличается от nominal typing в Java/C#, где совместимость решает явное объявление (`implements`/`extends`). `unknown` — безопасный `any`: тип принимает что угодно, но использовать значение нельзя без предварительного сужения. `any` выключает проверки — причём делает это **заразно**, распространяясь на всё, что через него проходит.

## Структурная типизация — тип определяется формой, не именем

```ts
interface Point { x: number; y: number; }
class Vector { constructor(public x: number, public y: number) {} }

function log(p: Point) { console.log(p.x, p.y); }
log(new Vector(1, 2)); // ok — Vector структурно совпадает с Point, хотя нигде не написано "implements Point"
log({ x: 1, y: 2, z: 3 }); // ok — лишнее поле не мешает, если объект пришёл ЧЕРЕЗ переменную
```

## Excess property check — исключение только для литералов

```ts
log({ x: 1, y: 2, z: 3 }); // ok, через переменную/промежуточный тип лишнее поле не мешает
log({ x: 1, y: 2, z: 3 } as Point); // тоже ok — явное приведение
// НО:
function log2(p: Point) {}
log2({ x: 1, y: 2, z: 3 }); // ошибка компиляции: Object literal may only specify known properties
```

Это единственное место, где TS ведёт себя не чисто структурно: объектный **литерал**, переданный напрямую в позицию с ожидаемым типом, проверяется на лишние поля (чаще всего это опечатка в имени поля, которую иначе никак не поймать при чисто структурной проверке). Через промежуточную переменную эта проверка не срабатывает — типичный вопрос "а почему тут ошибка, а тут нет".

## Эмуляция nominal typing — branding

```ts
type UserId = string & { readonly __brand: 'UserId' };
type OrderId = string & { readonly __brand: 'OrderId' };

function getUser(id: UserId) {}
declare const orderId: OrderId;
getUser(orderId); // ошибка — хотя оба "просто string" по значению, TS различает их по бренду

function toUserId(raw: string): UserId { return raw as UserId; } // единственная законная точка создания
```

Приём нужен, когда структурная типизация вредит: два разных ID-типа, обе строки — без бренда TS позволит перепутать местами `orderId` и `userId` в вызове, потому что "форма" (string) одинаковая.

## `unknown` vs `any`

```ts
function handle(x: unknown) {
  x.toUpperCase(); // ошибка компиляции — сначала докажи, что это строка
  if (typeof x === 'string') x.toUpperCase(); // ok, сужено
}

function handleAny(x: any) {
  x.toUpperCase(); // компилируется, упадёт в рантайме, если x — не строка
  const n: number = x; // any присваивается КУДА УГОДНО без проверки — заражает n
}
```

`any` не просто "выключает проверку для этой переменной" — любое значение, полученное из `any` (свойство, результат вызова метода, присваивание в другую переменную), само становится `any`, и проверки отключаются по всей цепочке дальше. `unknown` — тип верхней границы (top type): принимает всё, но отдаёт обратно только после явного сужения (`typeof`, `instanceof`, кастомный guard, `zod`-подобная схема).

## Практический кейс — `JSON.parse`

```ts
function parseConfig(raw: string): unknown { return JSON.parse(raw); }
const config = parseConfig(input);
// config.timeout — ошибка компиляции, пока не сузили тип
if (isConfig(config)) config.timeout; // ok, после guard'а
```

`JSON.parse` в стандартных тайпингах возвращает `any` — сознательный компромисс совместимости; в новом коде эту границу принято перекрывать явной обёрткой, возвращающей `unknown`, и дальше валидировать через type guard/схему (см. [TS: type guards, discriminated unions, satisfies](./11-ts-type-guards-unions.md)).

## Что спрашивают на собеседовании

1. **Чем структурная типизация отличается от номинативной?** — совместимость по форме (поля+типы), а не по явному объявлению наследования/реализации интерфейса, как в Java/C#.
2. **Почему для объектного литерала ошибка на лишнее поле, а для переменной той же формы — нет?** — excess property check срабатывает только на прямых литералах в позиции ожидаемого типа — эвристика против опечаток, не универсальное правило структурной типизации.
3. **Как эмулировать nominal typing в TS?** — branding: пересечение базового типа с уникальным "тегом" (`& { __brand: '...' }`), различающиеся типы перестают быть взаимозаменяемыми, даже совпадая по базовой форме.
4. **Почему `any` опаснее, чем кажется?** — он не локален: заражает все переменные и цепочки вызовов, куда попадает значение типа `any`, отключая проверки транзитивно, а не только в точке объявления.
5. **Как безопасно типизировать результат `JSON.parse`/внешнего API?** — обернуть в функцию, возвращающую `unknown`, и требовать явного сужения (guard/схема) перед использованием полей.

## Ссылки

- [TS Handbook — Everyday Types (`unknown`)](https://www.typescriptlang.org/docs/handbook/2/everyday-types.html#unknown)
- [TS FAQ — "TypeScript uses structural typing"](https://github.com/microsoft/TypeScript/wiki/FAQ#type-system-behavior)
- [TS Handbook — Object Types (excess property checks)](https://www.typescriptlang.org/docs/handbook/2/objects.html#excess-property-checks)
- [Nominal typing in TypeScript — branding pattern (Michal Zalecki)](https://michalzalecki.com/nominal-typing-in-typescript/)
