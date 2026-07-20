---
sidebar_position: 3
title: Прототипы и class
---

# Прототипное наследование и `class`

> **TL;DR:** У каждого объекта есть внутренний слот `[[Prototype]]` — ссылка на другой объект, куда движок идёт при неудачном поиске свойства. `class` — синтаксический сахар над этой же моделью: методы кладутся на `prototype` конструктора, `extends` выставляет **два** звена `[[Prototype]]` (для инстансов и отдельно для статики). Приватные поля `#x` — исключение: это НЕ часть прототипной цепочки.

## Prototype chain

```js
const animal = { eats: true };
const rabbit = Object.create(animal); // rabbit.[[Prototype]] === animal
rabbit.jumps = true;

rabbit.eats; // true — не найдено в rabbit, идём по [[Prototype]] в animal
rabbit.hasOwnProperty('eats'); // false — свойство чужое, не собственное
```

Поиск свойства — это обход цепочки `[[Prototype]]` до `null` (вершина цепочки — `Object.prototype`, у него `[[Prototype]] === null`). `obj.__proto__` — legacy-геттер/сеттер к этому же слоту; правильный API — `Object.getPrototypeOf` / `Object.setPrototypeOf` (или `Object.create` при создании).

## `class` — то же самое, но со страховкой от ошибок

```js
class Animal {
  static count = 0;
  constructor(name) { this.name = name; }
  eat() { return `${this.name} ест`; } // на Animal.prototype, не на инстансе
}
class Rabbit extends Animal {
  jump() { return `${this.name} прыгает`; }
}
```

Эквивалент на функциях-конструкторах:

```js
function Animal(name) { this.name = name; }
Animal.prototype.eat = function () { return `${this.name} ест`; };
function Rabbit(name) { Animal.call(this, name); }
Rabbit.prototype = Object.create(Animal.prototype); // связь для инстансов
Object.setPrototypeOf(Rabbit, Animal); // связь для статики (Rabbit.count и т.п.)
```

Отличия `class` от ручных конструкторов — не косметика, а поведенческие гарантии:
- вызов `Animal()` без `new` бросает `TypeError` (у обычной функции — тихо отработает с сюрпризами в `this`);
- тело класса выполняется в strict mode всегда;
- методы на `prototype` — **неперечисляемые** (`for...in` их не покажет, в отличие от методов, добавленных вручную через `Prototype.method = fn`);
- класс не поднимается (hoisting) как функция — есть temporal dead zone, использование до объявления кидает `ReferenceError`.

## Приватные поля — не прототип

```js
class Counter {
  #count = 0; // недоступно снаружи, нет в prototype chain
  increment() { return ++this.#count; }
}
new Counter().#count; // SyntaxError на этапе парсинга, не runtime-ошибка
```

Механизм не через `WeakMap` (как эмулировали до стандарта) — это отдельная спека с brand-check: движок при доступе к `#field` проверяет, что у объекта есть такой приватный слот, иначе кидает `TypeError`. Поля недоступны через `Object.keys`, `JSON.stringify`, `Reflect` — полная инкапсуляция, в отличие от соглашения `_field` (просто нижнее подчёркивание, которое ничего не запрещает технически).

## Что спрашивают на собеседовании

1. **Как работает поиск свойства через прототипы?** — обход `[[Prototype]]`-цепочки до `null`; `hasOwnProperty` отличает собственное свойство от унаследованного.
2. **`class` — просто сахар? Чем тогда отличается от function + prototype?** — да, сахар, но с гарантиями: TDZ, обязательный `new`, strict mode, неперечисляемые методы.
3. **Что делает `extends` под капотом?** — два `setPrototypeOf`: `Sub.prototype → Super.prototype` (для инстансов) и `Sub → Super` (для статических методов/полей).
4. **Чем `#private` отличается от `_private` по соглашению?** — реальная изоляция на уровне движка (brand check, SyntaxError при обращении извне), а не просто договорённость команды.
5. **Реализуйте наследование без `class`** — `Object.create` для прототипа инстансов + явный вызов родительского конструктора через `Parent.call(this, ...)`.

## Ссылки

- [MDN — Object prototypes](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Object_prototypes)
- [MDN — Classes](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes) и [Private class features](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes/Private_properties)
- [ECMA-262 — Ordinary Object Internal Methods (`[[Prototype]]`)](https://tc39.es/ecma262/#sec-ordinary-object-internal-methods-and-internal-slots)
- [TC39 — Class fields / private methods proposal](https://github.com/tc39/proposal-class-fields)
