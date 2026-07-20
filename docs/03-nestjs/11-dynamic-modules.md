---
sidebar_position: 11
title: Dynamic modules
---

# Dynamic modules

> **TL;DR:** Обычный модуль статичен; динамический — функция, возвращающая `DynamicModule` (модуль + провайдеры, собранные из переданных опций). Конвенции имён: `register` — конфиг для одного модуля, `forRoot` — один раз на всё приложение, `forFeature` — фичёвая часть поверх `forRoot`.

## Форма

```ts
@Module({})
export class RedisModule {
  static forRoot(options: RedisOptions): DynamicModule {
    return {
      module: RedisModule,
      global: true, // чтобы не импортировать в каждом модуле
      providers: [
        { provide: REDIS_OPTIONS, useValue: options },
        { provide: REDIS_CLIENT, useFactory: (o) => createClient(o), inject: [REDIS_OPTIONS] },
      ],
      exports: [REDIS_CLIENT],
    };
  }
}
```

## forRoot vs forRootAsync

`forRoot(options)` требует опции **на момент компиляции модуля**. Если опции зависят от других провайдеров (почти всегда — от `ConfigService`), нужен async-вариант:

```ts
static forRootAsync(options: {
  imports?: any[];
  useFactory: (...args: any[]) => RedisOptions | Promise<RedisOptions>;
  inject?: any[];
}): DynamicModule {
  return {
    module: RedisModule,
    imports: options.imports ?? [],
    providers: [
      { provide: REDIS_OPTIONS, useFactory: options.useFactory, inject: options.inject ?? [] },
      { provide: REDIS_CLIENT, useFactory: (o) => createClient(o), inject: [REDIS_OPTIONS] },
    ],
    exports: [REDIS_CLIENT],
  };
}

// использование:
RedisModule.forRootAsync({
  useFactory: (config: ConfigService) => ({ url: config.get('REDIS_URL') }),
  inject: [ConfigService],
})
```

Суть `async`-варианта — **не** асинхронность сама по себе, а возможность инжектить зависимости в фабрику опций.

## forFeature

`TypeOrmModule.forRoot()` настраивает коннект один раз; `TypeOrmModule.forFeature([Cat])` в каждом фичёвом модуле регистрирует репозитории поверх общего коннекта. Тот же паттерн: глобальная инфраструктура + локальная регистрация.

## ConfigurableModuleBuilder

С NestJS 9 бойлерплейт выше можно сгенерировать:

```ts
export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN } =
  new ConfigurableModuleBuilder<RedisOptions>().build();
// RedisModule extends ConfigurableModuleClass → register/registerAsync из коробки
```

## Что спрашивают на собеседовании

1. **`forRoot` vs `forRootAsync`?** — async даёт `useFactory + inject`: опции модуля могут зависеть от ConfigService и других провайдеров.
2. **`register` vs `forRoot`?** — конвенция: register — разные конфиги в разных модулях (HttpModule), forRoot — одна конфигурация на приложение (TypeOrmModule).
3. **Когда `forFeature`?** — фичёвая регистрация поверх глобальной инфраструктуры (`forFeature([Entity])`).
4. **Напишите обёртку над Redis-клиентом как dynamic module** — см. код выше; ключевые элементы: токен опций, токен клиента, exports.
5. **Что делает `global: true`?** — модуль виден везде без импорта; удобно для инфраструктуры, но злоупотребление убивает явность графа зависимостей.

## Ссылки

- [Dynamic modules — официальная документация](https://docs.nestjs.com/fundamentals/dynamic-modules) (включая ConfigurableModuleBuilder)
- [Advanced: dynamic modules — глубокий разбор в блоге NestJS](https://dev.to/nestjs/advanced-nestjs-how-to-build-completely-dynamic-nestjs-modules-1370)
- Исходники-образцы: [`@nestjs/typeorm`](https://github.com/nestjs/typeorm/blob/master/lib/typeorm.module.ts) и [`@nestjs/jwt`](https://github.com/nestjs/jwt/blob/master/lib/jwt.module.ts) — эталонные реализации forRoot/forRootAsync
