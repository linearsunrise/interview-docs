---
sidebar_position: 10
title: "TS: conditional, infer, mapped, template literal types"
---

# Conditional types, `infer`, mapped types, template literal types

> **TL;DR:** `T extends U ? X : Y` — ветвление на уровне типов. Если `T` — union, условие **автоматически распределяется** по каждому члену union (distributive conditional types) — источник и мощи, и неожиданного поведения. `infer` — способ "вытащить" подтип внутри условия (аргумент функции, элемент массива, тип из промиса). Mapped types трансформируют объект поле за полем. Template literal types строят строковые типы из union'ов через конкатенацию.

## Conditional types и distributivity

```ts
type ToArray<T> = T extends any ? T[] : never;
type Result = ToArray<string | number>; // string[] | number[], а не (string | number)[]
```

Это не баг, а поведение по спецификации: когда `T` в `T extends U ? X : Y` — "голый" параметр типа, а на его месте union — TS **распределяет** условие по каждому члену отдельно и объединяет результаты. Чтобы отключить distributivity (получить `(string | number)[]`), оборачивают в кортеж — так проверка перестаёт видеть "голый" параметр:

```ts
type ToArrayNonDist<T> = [T] extends [any] ? T[] : never;
type Result2 = ToArrayNonDist<string | number>; // (string | number)[]
```

## `infer` — извлечение типа внутри условия

```ts
type UnwrapPromise<T> = T extends Promise<infer U> ? U : T;
type A = UnwrapPromise<Promise<string>>; // string
type B = UnwrapPromise<number>;          // number — ветка false, T без изменений

type FirstArg<T> = T extends (arg: infer A, ...rest: any[]) => any ? A : never;
type C = FirstArg<(id: string, opts: object) => void>; // string
```

`infer` работает только внутри `extends`-условия conditional type — это единственное место, где TS позволяет "объявить" новую переменную типа для последующего использования в true-ветке.

## Mapped types

```ts
type DeepPartial<T> = T extends object
  ? { [P in keyof T]?: DeepPartial<T[P]> }
  : T;

type DeepReadonly<T> = T extends object
  ? { readonly [P in keyof T]: DeepReadonly<T[P]> }
  : T;
```

Модификаторы `?`/`readonly` можно и снимать через `-?`/`-readonly` (например, `Required<T>` — это `{ [P in keyof T]-?: T[P] }`). Ключи можно переименовывать через `as` (key remapping) — на этом строится, например, генерация геттеров из полей объекта:

```ts
type Getters<T> = { [P in keyof T as `get${string & P}`]: () => T[P] };
```

## Template literal types

```ts
type Route = '/users' | '/orders';
type Endpoint = `${Route}/:id`; // "/users/:id" | "/orders/:id"

type ExtractParams<T extends string> =
  T extends `${string}:${infer Param}/${infer Rest}`
    ? { [K in Param | keyof ExtractParams<Rest>]: string }
    : T extends `${string}:${infer Param}`
      ? { [K in Param]: string }
      : {};

type Params = ExtractParams<'/users/:id/orders/:orderId'>; // { id: string; orderId: string }
```

Комбинация template literal types + `infer` — то, как типизируют роутинг (например, в некоторых версиях React Router/tRPC): строка маршрута парсится на уровне типов, без единого рантайм-вызова.

## Что спрашивают на собеседовании

1. **Что такое distributive conditional types и в чём подвох?** — `T extends U ? X : Y` с union на месте "голого" `T` распределяется по членам union отдельно; чтобы отключить — обернуть в `[T]`.
2. **Как работает `infer`?** — объявляет переменную типа внутри `extends`-условия, доступную в true-ветке; способ "распаковать" параметризованный тип (`Promise<T>`, аргумент функции, элемент массива).
3. **Напишите `ReturnType` сами.** — `T extends (...args: any) => infer R ? R : never`.
4. **Напишите `DeepPartial`.** — рекурсивный mapped type: если `T[P]` — объект, рекурсивно применить `DeepPartial`, иначе оставить как есть.
5. **Как типизировать параметры строки маршрута (`/users/:id`) без рантайм-парсинга?** — template literal types + `infer` для извлечения имён параметров между `:` и `/`.

## Ссылки

- [TS Handbook — Conditional Types](https://www.typescriptlang.org/docs/handbook/2/conditional-types.html) (там же про distributive conditional types и `infer`)
- [TS Handbook — Mapped Types](https://www.typescriptlang.org/docs/handbook/2/mapped-types.html) (key remapping через `as`)
- [TS Handbook — Template Literal Types](https://www.typescriptlang.org/docs/handbook/2/template-literal-types.html)
- [TypeScript source — `lib.es5.d.ts`](https://github.com/microsoft/TypeScript/blob/main/src/lib/es5.d.ts) — `ReturnType`, `Parameters`, `Awaited` в реальном виде
