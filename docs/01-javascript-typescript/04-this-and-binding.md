---
sidebar_position: 4
title: "this и способы привязки"
---

# `this`: 4 правила привязки

> **TL;DR:** `this` определяется **в момент вызова**, а не объявления функции — кроме стрелочных функций, у которых своего `this` нет вообще, они берут его из окружающего лексического scope. Приоритет правил: `new` > явная привязка (`bind`) > неявная (`obj.method()`) > дефолтная (`undefined`/`globalThis`).

## 4 правила (по убыванию приоритета)

```js
function show() { console.log(this); }

// 1. new-binding — this = новый объект
new show(); // this = {} (новый инстанс)

// 2. explicit — call/apply/bind
show.call({ id: 1 });   // this = {id: 1}
const bound = show.bind({ id: 2 });
bound(); // this = {id: 2}, и это НАВСЕГДА — повторный bind/call не переопределит

// 3. implicit — вызов через объект
const obj = { id: 3, show };
obj.show(); // this = obj

// 4. default — просто вызов
show(); // strict mode: undefined; sloppy mode: globalThis
```

`bind` создаёт новую функцию с зашитым `this` — этот `this` уже нельзя перебить даже через `call`/`apply` на связанной функции. `new` игнорирует `bind`, только если использовать специальный конструктор-биндинг — в обычном коде на собеседовании этого хватит.

## Частая ловушка: потеря `this` при "вырезании" метода

```js
class Service {
  value = 42;
  getValue() { return this.value; }
}
const service = new Service();
const fn = service.getValue;
fn(); // TypeError / undefined — this потерян, это просто функция без контекста

setTimeout(service.getValue, 0); // та же ловушка — колбэк вызывается "голым"
```

Три способа починить: `fn = service.getValue.bind(service)`, обёртка `() => service.getValue()`, или **поле класса со стрелочной функцией** — она захватывает `this` лексически при создании инстанса, а не при вызове:

```js
class Service {
  value = 42;
  getValue = () => this.value; // this зафиксирован на момент создания инстанса
}
```

Компромисс: такое поле создаётся заново на каждый инстанс (не на прототипе) — чуть больше памяти на много инстансов, зато безопасно передавать как колбэк (частый паттерн в React-обработчиках и Node event-подписках).

## Стрелочные функции не могут быть конструкторами

У них нет собственного `[[Construct]]`, `prototype`, `arguments` — `new (() => {})()` бросает `TypeError`. Это не ограничение, а следствие того, что стрелочная функция вообще не участвует в правилах привязки `this` — ей нечего переопределять.

## Что спрашивают на собеседовании

1. **Перечислите правила определения `this` и их приоритет.** — new → bind/call/apply → неявный вызов через объект → дефолт; стрелочные функции — вне этих правил, берут `this` из места объявления.
2. **Почему `setTimeout(obj.method, 0)` ломает `this`?** — метод передаётся как голая ссылка на функцию, вызывается без объекта-получателя — implicit binding не применяется.
3. **Чем `bind` отличается от `call`/`apply`?** — `call`/`apply` вызывают функцию немедленно с указанным `this`; `bind` возвращает новую функцию с намертво зафиксированным `this` для будущих вызовов.
4. **Почему стрелочную функцию нельзя использовать как конструктор?** — у неё нет `[[Construct]]` и своего `this` — `new` физически нечего привязывать.
5. **Как правильно передать метод класса как колбэк, чтобы не потерять `this`?** — стрелочное поле класса, либо `.bind(this)` в конструкторе, либо обёртка в месте вызова.

## Ссылки

- [MDN — `this`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/this)
- [MDN — `Function.prototype.bind`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/bind)
- [You Don't Know JS — `this` & Object Prototypes](https://github.com/getify/You-Dont-Know-JS/blob/1st-ed/this%20%26%20object%20prototypes/README.md)
- [ECMA-262 — Arrow Function Definitions (нет `[[Construct]]`)](https://tc39.es/ecma262/#sec-arrow-function-definitions)
