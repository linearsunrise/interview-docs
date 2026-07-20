---
sidebar_position: 18
title: Инструменты разработчика
---

# Инструменты разработчика: CLI, Devtools, REPL, дебаг

> **TL;DR:** Официальный тулинг: **Nest CLI** (генераторы, монорепо, SWC-сборка, `--debug`), **NestJS Devtools** (граф зависимостей + CI-интеграция, платный), **REPL** (интерактивная консоль к DI-контейнеру), **`NEST_DEBUG`** (логи инжектора). Дебаггер — обычный Node-инспектор, VS Code/WebStorm/Chrome — лишь клиенты к нему.

## Nest CLI

```bash
nest g resource orders   # CRUD-модуль целиком: module/controller/service/dto/тесты
nest g library shared    # библиотека в монорепо (workspaces)
nest start --watch --debug   # dev-режим с инспектором
nest build -b swc            # сборка через SWC вместо tsc
```

Что стоит знать:
- **Генераторы** (`nest g --help` — полный список) работают на schematics; можно писать свои для корпоративных шаблонов (`@nestjs/schematics` как база).
- **Монорепо-режим**: `nest g app` / `nest g library` превращают проект в workspace с общими библиотеками — официальный ответ на «как расшарить код между сервисами» (см. [структуру проекта](./17-project-structure.md)).
- **SWC** (`-b swc`, или `compilerOptions.builder` в nest-cli.json) — компиляция на Rust, на порядок быстрее tsc. Нюанс: SWC не делает type-check — добавляют `--type-check` (тайпчек параллельно) и флаг для плагинов CLI (`@nestjs/swagger` требует plugin-обработки метаданных).

## Дебаггер — это просто Node inspector

`nest start --debug` = `node --inspect`. Дальше любой клиент: VS Code (attach на 9229), WebStorm, `chrome://inspect`.

В докере — два обязательных шага, иначе «не цепляется»:

```bash
node --inspect=0.0.0.0:9229 dist/main.js   # слушать не localhost!
# + пробросить порт: docker run -p 9229:9229 ...
```

VS Code attach-конфиг: `"request": "attach", "port": 9229, "restart": true` + `localRoot`/`remoteRoot`, чтобы совпадали пути к сорсам. Для точек останова в TS нужны source maps (`"sourceMap": true` — в Nest-шаблоне уже включено).

## NestJS Devtools (официальный, платный)

Продукт core-команды: [devtools.nestjs.com](https://devtools.nestjs.com) + пакет `@nestjs/devtools-integration`.

```ts
// app.module.ts
DevtoolsModule.register({ http: process.env.NODE_ENV !== 'production' }),
// main.ts
NestFactory.create(AppModule, { snapshot: true });
```

Возможности:
- **Интерактивный граф** модулей и провайдеров: наглядно видно, откуда циклическая зависимость и по какой цепочке не резолвится токен («Nest can't resolve dependencies» превращается в картинку).
- **Routes explorer** — все маршруты с их пайплайном (guards/interceptors/pipes).
- **CI/CD-интеграция**: строит снапшот графа на каждый коммит и показывает диф между ветками — ревьюишь изменения архитектуры, а не только кода (детали подключения — в [доке по CI/CD](https://docs.nestjs.com/devtools/ci-cd)).
- Есть **триал**, дальше подписка — это единственный платный элемент официального тулинга.

Бесплатная альтернатива для графа — [nestjs-spelunker](https://github.com/jmcdo29/nestjs-spelunker): `SpelunkerModule.explore(app)` → mermaid-диаграмма зависимостей (поддерживается автором из core-team, но статус неофициальный).

## REPL

С NestJS 9 приложение можно поднять как интерактивную консоль — без HTTP, но с полным DI-контейнером:

```ts
// repl.ts
import { repl } from '@nestjs/core';
await repl(AppModule);
```

```bash
npm run start -- --entryFile repl
> get(CatsService).findAll()      # дёрнуть метод провайдера
> $(CatsService)                  # шорткат для get
> methods(CatsService)            # список методов
> debug(CatsModule)               # состав модуля
```

Применения: исследование чужой кодовой базы, ручные операции (пересчитать что-то в staging), проверка сервиса без написания контроллера. С флагом `--watch` REPL переживает пересборку.

## `NEST_DEBUG` и логи инжектора

```bash
NEST_DEBUG=true nest start
```

Печатает процесс резолва зависимостей: какой модуль инстанцируется, какой провайдер в каком контексте ищется. Самый дешёвый способ разобраться с ошибками DI, когда стек-трейс бесполезен. Дополнительно: `NestFactory.create(AppModule, { logger: ['verbose'] })` — verbose-логи самого фреймворка (маппинг роутов, инициализация модулей).

## Что спрашивают на собеседовании

1. **Как дебажите «Nest can't resolve dependencies»?** — читать сообщение (index аргумента + модуль), `NEST_DEBUG=true`, граф в Devtools/spelunker; проверить exports и forwardRef.
2. **Как подключить дебаггер к приложению в контейнере?** — `--inspect=0.0.0.0:9229`, проброс порта, attach + source maps; в k8s — `kubectl port-forward`.
3. **Чем SWC-сборка отличается от tsc и что теряем?** — скорость vs type-check и обработка декораторных метаданных плагинами; лечится `--type-check` и plugin-опциями.
4. **Как быстро проверить сервис без HTTP-слоя?** — REPL: `get(Service).method()`; либо юнит-тест с TestingModule.
5. **Как контролировать архитектуру (границы модулей) в CI?** — Devtools graph diff; бесплатно — spelunker + snapshot-тест на mermaid-вывод или eslint-boundaries.

## Ссылки

- [Nest CLI overview](https://docs.nestjs.com/cli/overview), [monorepo/workspaces](https://docs.nestjs.com/cli/monorepo), [scripts](https://docs.nestjs.com/cli/scripts)
- [SWC — официальный рецепт](https://docs.nestjs.com/recipes/swc)
- [Devtools: overview](https://docs.nestjs.com/devtools/overview) и [CI/CD](https://docs.nestjs.com/devtools/ci-cd); сам сервис — [devtools.nestjs.com](https://devtools.nestjs.com)
- [REPL — официальный рецепт](https://docs.nestjs.com/recipes/repl)
- [Debugging — гайд Node.js](https://nodejs.org/en/learn/getting-started/debugging) — inspector, флаги, клиенты
- [nestjs-spelunker](https://github.com/jmcdo29/nestjs-spelunker)
- Исходники: [`packages/core/repl`](https://github.com/nestjs/nest/tree/master/packages/core/repl) — как REPL строит нативные функции поверх контейнера
