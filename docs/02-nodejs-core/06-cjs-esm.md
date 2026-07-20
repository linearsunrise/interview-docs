---
sidebar_position: 6
title: CommonJS vs ESM
---

# CommonJS vs ESM

> **TL;DR:** CJS: `require` — синхронный, динамический, экспорт — копия значения на момент экспорта. ESM: `import` — статический (анализируется до выполнения), асинхронная загрузка, **live bindings**, top-level await. Node решает, чем считать файл, по расширению (`.cjs`/`.mjs`) и полю `"type"` в package.json.

## Ключевые различия

| | CommonJS | ESM |
|---|---|---|
| Синтаксис | `require` / `module.exports` | `import` / `export` |
| Загрузка | синхронная, в рантайме | асинхронная, граф строится до выполнения |
| Биндинги | копия значения | live binding (видны изменения переменной) |
| `require` в середине кода / по условию | да | только `await import()` |
| Top-level await | нет | да |
| `__dirname` | есть | `import.meta.dirname` (Node 20+) / `import.meta.url` |
| Резолв `./file` | допишет `.js`, найдёт `index.js` | расширение обязательно (в строгом режиме) |
| Tree shaking (бандлерами) | плохо | да — благодаря статичности |

## Interop — то, что реально спрашивают

- **ESM → CJS: работает.** `import pkg from 'cjs-lib'` — `module.exports` становится default; именованные экспорты Node пытается угадать статическим анализом (cjs-module-lexer).
- **CJS → ESM: исторически нельзя** (`require(esm)` кидал `ERR_REQUIRE_ESM`) — потому что ESM асинхронный. Обходной путь — `await import()`. **С Node 22+ `require(esm)` работает**, если в модуле нет top-level await — это сняло главную боль экосистемы dual-package.
- **Dual package**: поле `"exports"` с условиями `"import"`/`"require"`; hazard — два инстанса одной библиотеки (CJS и ESM копии) в одном процессе → `instanceof` ломается.

```json
{ "exports": { ".": { "import": "./dist/index.mjs", "require": "./dist/index.cjs" } } }
```

## Что учитывать в NestJS-мире

NestJS-проекты компилируются tsc в CommonJS (декораторы + `emitDecoratorMetadata` исторически завязаны на CJS-пайплайн). Отсюда типовая боль: ESM-only пакеты (`chalk` 5+, `nanoid` 4+ и т.п.) нельзя `require` из скомпилированного кода на старых Node — либо `await import()`, либо даунгрейд пакета, либо Node 22+.

## Что спрашивают на собеседовании

1. **Почему `import` нельзя вызвать по условию?** — статический граф модулей строится до выполнения; для динамики есть `await import()`.
2. **Что такое live bindings?** — импортированное имя — ссылка на переменную модуля, а не копия: `export let counter` изменится у всех импортёров.
3. **Как Node определяет тип модуля?** — `.mjs`/`.cjs` явно; `.js` — по ближайшему `package.json` `"type": "module" | "commonjs"`.
4. **Циклические импорты: CJS vs ESM?** — CJS вернёт **частично заполненный** `module.exports` (что успело выполниться); ESM за счёт hoisting и live bindings чаще работает корректно, но TDZ-ошибки возможны.
5. **Dual package hazard?** — один пакет загружен дважды (как CJS и как ESM) → разные инстансы, ломаются `instanceof` и синглтоны.

## Ссылки

- [ECMAScript modules — официальная документация](https://nodejs.org/api/esm.html) (включая interop и резолв)
- [Modules: Packages — `"exports"`, `"type"`, dual package hazard](https://nodejs.org/api/packages.html)
- [`require(esm)` в Node 22](https://nodejs.org/en/blog/announcements/v22-release-announce) — анонс релиза; [PR nodejs/node#51977](https://github.com/nodejs/node/pull/51977)
- [cjs-module-lexer](https://github.com/nodejs/cjs-module-lexer) — как Node угадывает именованные экспорты CJS
