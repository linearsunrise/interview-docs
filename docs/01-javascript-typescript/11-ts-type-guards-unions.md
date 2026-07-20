---
sidebar_position: 11
title: "TS: type guards, discriminated unions, satisfies"
---

# Type guards, discriminated unions, `satisfies`

> **TL;DR:** Type guard — рантайм-проверка (`typeof`, `instanceof`, `in`, кастомный предикат), после которой компилятор **сужает** тип в конкретной ветке кода. Discriminated union — паттерн: общее строковое литеральное поле-тег в каждом варианте union, позволяющее TS автоматически сужать по `switch`/`if` на этом поле. `satisfies` (TS 4.9+) проверяет значение на соответствие типу, **не расширяя** и не теряя выведенный литеральный тип — в отличие от аннотации `: Type`.

## Встроенные и кастомные type guards

```ts
function process(value: string | number) {
  if (typeof value === 'string') value.toUpperCase(); // сужено до string
  else value.toFixed(2); // сужено до number
}

class Dog { bark() {} }
class Cat { meow() {} }
function speak(animal: Dog | Cat) {
  if (animal instanceof Dog) animal.bark(); else animal.meow();
}

// кастомный предикат — сигнатура "value is T"
function isString(x: unknown): x is string {
  return typeof x === 'string';
}
const values: unknown[] = [1, 'a', true];
values.filter(isString); // string[] — TS доверяет предикату, сам он не проверяет тело функции
```

`in` полезен для сужения объектных union без общего тега: `if ('bark' in animal) animal.bark()`.

## Discriminated unions — основной паттерн для результатов операций

```ts
type ApiResult<T> =
  | { status: 'ok'; data: T }
  | { status: 'error'; error: string };

function handle<T>(result: ApiResult<T>) {
  switch (result.status) {
    case 'ok': return result.data;      // сужено до варианта с data
    case 'error': throw new Error(result.error); // сужено до варианта с error
    default: {
      const _exhaustive: never = result; // ошибка компиляции, если добавили новый вариант и забыли обработать
      throw new Error('unreachable');
    }
  }
}
```

Приём `const x: never = result` в `default` — стандартный способ **exhaustiveness checking**: если позже в union добавится третий вариант, эта строка перестанет компилироваться, что заставит явно обработать новый случай — ловится на этапе сборки, не в проде.

## `satisfies` — проверка без потери литерального типа

```ts
const config: Record<string, number> = { timeout: 5000, retries: 3 };
config.timeout; // number — литерал 5000 потерян, тип объекта "стал" Record<string, number>

const config2 = { timeout: 5000, retries: 3 } satisfies Record<string, number>;
config2.timeout; // 5000 — литеральный тип сохранён, но TS всё равно проверил соответствие Record<string, number>
```

Разница принципиальна для конфигов и объектов-мапперов: аннотация `:` типа **виджет-тайп сразу**, `satisfies` — только проверяет совместимость, оставляя самый узкий выведенный тип. Это же ловит опечатки в ключах (ошибка, если объект не соответствует форме), чего не даёт `as const` в одиночку.

## Что спрашивают на собеседовании

1. **Как написать кастомный type guard?** — функция с возвращаемым типом `x is T`; TS доверяет объявленной сигнатуре, тело не анализируется на предмет "честности" проверки.
2. **Что такое discriminated union и зачем общий тег-поле?** — единое литеральное поле (`status`, `type`, `kind`) во всех вариантах union, по которому TS автоматически сужает тип в `switch`/`if`.
3. **Как гарантировать, что обработаны все варианты union?** — exhaustiveness checking через `default: const _x: never = value` — не скомпилируется, если появится необработанный вариант.
4. **Чем `satisfies` отличается от аннотации типа `: T`?** — аннотация сразу расширяет (widen) тип значения до `T`, `satisfies` только проверяет совместимость, сохраняя самый узкий литеральный тип из фактического значения.
5. **Как безопасно типизировать `JSON.parse(...)`?** — результат типа `any` (или `unknown` в новых версиях lib) нужно сузить: явный type guard/схема валидации (zod/io-ts) перед использованием, а не приведение типа `as`.

## Ссылки

- [TS Handbook — Narrowing](https://www.typescriptlang.org/docs/handbook/2/narrowing.html)
- [TS Handbook — Discriminated Unions (Narrowing raздел)](https://www.typescriptlang.org/docs/handbook/2/narrowing.html#discriminated-unions)
- [TS 4.9 Release Notes — `satisfies`](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-9.html#the-satisfies-operator)
