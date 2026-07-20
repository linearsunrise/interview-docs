---
sidebar_position: 9
title: "TS: generics и utility types"
---

# Generics и utility types

> **TL;DR:** Дженерик — это параметр типа, который связывает вход и выход функции/типа так, что компилятор проверяет конкретное соответствие, а не стирает информацию до `any`. `extends` в дженерике — не наследование, а **ограничение** ("T должен иметь хотя бы это"). Большинство utility types (`Partial`, `Pick`, `Omit`, `Record`, `ReturnType`) — не магия компилятора, а обычные дженерики поверх mapped/conditional types, объявленные в `lib.es5.d.ts` — их можно написать самому.

## Зачем дженерик, если можно `any`

```ts
function first(arr: any[]): any { return arr[0]; }
first([1, 2, 3]) + 1; // any — компилятор ничего не проверяет дальше

function firstTyped<T>(arr: T[]): T { return arr[0]; }
firstTyped([1, 2, 3]) + 1;      // T = number, ок
firstTyped(['a', 'b'])[0]; // T = string, .toUpperCase() будет с автодополнением
```

`any` разрывает связь между входом и выходом — компилятор теряет тип уже на первом шаге. Дженерик сохраняет эту связь: тип `T` выводится один раз из аргумента и подставляется везде, где встречается в сигнатуре.

## Constraints (`extends`)

```ts
function getLength<T extends { length: number }>(x: T): number {
  return x.length; // без extends TS не знает, что у T вообще есть .length
}
getLength('str'); // ok, строка подходит под ограничение
getLength(42);     // ошибка компиляции — у number нет .length
```

Ограничение сужает допустимые типы для `T`, не превращая дженерик в конкретный тип — это критично отличать от наследования классов.

## Как устроены самые частые utility types

```ts
// Partial<T> — все поля опциональны
type Partial<T> = { [P in keyof T]?: T[P] };

// Pick<T, K> — оставить только перечисленные ключи
type Pick<T, K extends keyof T> = { [P in K]: T[P] };

// Omit<T, K> — обратное Pick, через Exclude
type Omit<T, K extends keyof any> = Pick<T, Exclude<keyof T, K>>;

// Record<K, V> — словарь с заданными ключами и типом значения
type Record<K extends keyof any, V> = { [P in K]: V };

// ReturnType<T> — вытащить тип возврата функции через infer (см. следующий файл)
type ReturnType<T extends (...args: any) => any> = T extends (...args: any) => infer R ? R : any;
```

Практический вывод: если библиотечного utility type не хватает — обычно можно собрать свой из mapped/conditional типов теми же инструментами.

## Дефолтные параметры и множественные дженерики

```ts
interface ApiResponse<TData, TError = { message: string }> {
  data: TData | null;
  error: TError | null;
}
// ApiResponse<User> — TError подставится по умолчанию
```

## Что спрашивают на собеседовании

1. **Зачем дженерики, если типы всё равно стираются в рантайме?** — они работают на этапе компиляции: сохраняют связь между входными и выходными типами, чего `any`/перегрузки без параметра не дают.
2. **Что делает `extends` в `<T extends U>`?** — не наследование, а ограничение: "T обязан быть совместим с U", иначе ошибка компиляции.
3. **Напишите свой `Pick<T, K>`.** — mapped type `{ [P in K]: T[P] }` с ограничением `K extends keyof T`.
4. **Чем `Omit` отличается от `Pick` под капотом?** — `Omit<T, K>` = `Pick<T, Exclude<keyof T, K>>` — просто вычисляет "остальные" ключи через `Exclude` и переиспользует `Pick`.
5. **Когда нужен дефолтный параметр дженерика?** — когда тип обычно один и тот же и хочется не писать его каждый раз, но оставить возможность переопределить (частый случай — тип ошибки в обёртке ответа API).

## Ссылки

- [TS Handbook — Generics](https://www.typescriptlang.org/docs/handbook/2/generics.html)
- [TS Handbook — Utility Types](https://www.typescriptlang.org/docs/handbook/utility-types.html)
- [TypeScript source — `lib.es5.d.ts`](https://github.com/microsoft/TypeScript/blob/main/src/lib/es5.d.ts) — реальные объявления `Partial`, `Pick`, `Omit`, `Record` и т.д.
