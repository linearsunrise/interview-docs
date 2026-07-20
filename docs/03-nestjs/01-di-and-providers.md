---
sidebar_position: 1
title: DI-контейнер и провайдеры
---

# DI-контейнер и провайдеры

> **TL;DR:** Провайдер — это всё, что можно заинжектить. Токен — ключ, по которому контейнер находит инстанс. Класс сам себе токен; для остального нужны `@Inject()` и кастомные провайдеры (`useClass / useValue / useFactory / useExisting`).

## Как это работает под капотом

1. TypeScript с флагом `emitDecoratorMetadata` записывает типы параметров конструктора в метаданные `design:paramtypes` (через `reflect-metadata`).
2. При старте `DependenciesScanner` обходит модули и строит граф зависимостей.
3. `Injector` рекурсивно резолвит зависимости каждого провайдера и кэширует инстансы (по умолчанию — синглтоны на весь app).

Именно поэтому **интерфейс нельзя использовать как токен** — интерфейсы стираются при компиляции, в `design:paramtypes` попадает `Object`. Для абстракций используют абстрактный класс или строковый/symbol токен + `@Inject()`.

## Виды кастомных провайдеров

```ts
// useValue — готовое значение (конфиг, мок в тестах)
{ provide: 'API_KEY', useValue: 'secret' }

// useClass — подмена реализации (по окружению, для тестов)
{ provide: LoggerService, useClass: isProd ? JsonLogger : PrettyLogger }

// useFactory — значение вычисляется, можно инжектить зависимости
{
  provide: 'DB_CONNECTION',
  useFactory: (config: ConfigService) => createConnection(config.get('DB_URL')),
  inject: [ConfigService],
}

// useExisting — алиас: два токена, один инстанс
{ provide: 'AliasedLogger', useExisting: LoggerService }
```

Инжект не-классового токена: `constructor(@Inject('DB_CONNECTION') private db: Connection)`.

Полезное рядом: `@Optional()` (зависимость может отсутствовать), `ModuleRef.get()` / `ModuleRef.resolve()` (достать провайдер из контейнера вручную), `LazyModuleLoader` (ленивая загрузка модулей).

## Что спрашивают на собеседовании

1. **Как Nest узнаёт, что инжектить в конструктор?** — `reflect-metadata` + `design:paramtypes`; сканер строит граф модулей, инжектор его резолвит.
2. **Почему нельзя инжектить по интерфейсу?** — интерфейсы стираются при компиляции; нужен класс или токен + `@Inject()`.
3. **Когда `useFactory` вместо `useClass`?** — когда создание требует логики или других зависимостей (`inject`), в т.ч. асинхронных (`async useFactory`).
4. **Зачем `useExisting`?** — алиас: постепенная миграция токенов, узкий интерфейс поверх «толстого» сервиса.
5. **Что будет, если провайдер не экспортирован из модуля?** — он приватный: другой модуль его не увидит, даже импортировав модуль. Экспорт — это API модуля.

## Ссылки

- [Providers — официальная документация](https://docs.nestjs.com/providers)
- [Custom providers — все виды `use*`](https://docs.nestjs.com/fundamentals/custom-providers)
- [Modules](https://docs.nestjs.com/modules)
- [ModuleRef](https://docs.nestjs.com/fundamentals/module-ref) и [Lazy-loading modules](https://docs.nestjs.com/fundamentals/lazy-loading-modules)
- Исходники: [`packages/core/injector/injector.ts`](https://github.com/nestjs/nest/blob/master/packages/core/injector/injector.ts) — сам резолвинг; [`packages/core/scanner.ts`](https://github.com/nestjs/nest/blob/master/packages/core/scanner.ts) — построение графа модулей; [`packages/core/injector/instance-loader.ts`](https://github.com/nestjs/nest/blob/master/packages/core/injector/instance-loader.ts) — инстанцирование
- [reflect-metadata](https://github.com/rbuckton/reflect-metadata) и [TC39 proposal-decorators](https://github.com/tc39/proposal-decorators) — на чём всё держится
