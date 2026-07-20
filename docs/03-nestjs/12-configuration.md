---
sidebar_position: 12
title: Конфигурация
---

# Конфигурация

> **TL;DR:** `@nestjs/config` = обёртка над dotenv + DI. Три правила senior-уровня: валидировать env на старте (fail fast), не читать `process.env` напрямую в коде, использовать типизированные namespace-конфиги через `registerAs`.

## База

```ts
ConfigModule.forRoot({
  isGlobal: true,          // не импортировать в каждом модуле
  envFilePath: ['.env.local', '.env'],
  cache: true,
});
// где угодно:
constructor(private config: ConfigService) {}
this.config.get<string>('DATABASE_URL');
```

## Валидация env (fail fast)

```ts
// вариант Joi (из доков):
ConfigModule.forRoot({
  validationSchema: Joi.object({
    NODE_ENV: Joi.string().valid('development', 'production', 'test').required(),
    PORT: Joi.number().default(3000),
    DATABASE_URL: Joi.string().required(),
  }),
});

// вариант zod (через кастомный validate):
const envSchema = z.object({ PORT: z.coerce.number().default(3000), DATABASE_URL: z.string().url() });
ConfigModule.forRoot({ validate: (env) => envSchema.parse(env) });
```

Приложение с невалидным конфигом должно **упасть на старте**, а не на первом запросе в 3 часа ночи.

## Типизированные конфиги: registerAs + ConfigType

```ts
export const dbConfig = registerAs('db', () => ({
  url: process.env.DATABASE_URL!,
  poolSize: parseInt(process.env.DB_POOL_SIZE ?? '10', 10),
}));

// модуль: ConfigModule.forRoot({ load: [dbConfig] })
// инжект всего namespace целиком, с типами:
constructor(@Inject(dbConfig.KEY) private db: ConfigType<typeof dbConfig>) {}
this.db.poolSize // number, автокомплит работает
```

## Что спрашивают на собеседовании

1. **Почему не читать `process.env` напрямую?** — нет валидации и типов, невозможно подменить в тестах, скрытая зависимость (не видна в конструкторе).
2. **Как гарантировать, что приложение не стартует без обязательных env?** — `validationSchema` (Joi) или `validate` (zod/class-validator).
3. **Как получить типизированный конфиг?** — `registerAs` + `@Inject(config.KEY)` + `ConfigType<typeof config>`.
4. **Как ConfigService попадает в опции другого модуля?** — `forRootAsync` + `useFactory` + `inject: [ConfigService]` (связка с темой dynamic modules).
5. **Секреты в env-файлах — ок?** — для локалки да; в проде — секрет-менеджер (Vault, AWS Secrets Manager), env внедряется платформой, `.env` не коммитится.

## Ссылки

- [Configuration — официальная документация](https://docs.nestjs.com/techniques/configuration) — включая validationSchema, registerAs, ConfigType, expandVariables
- [`@nestjs/config` исходники](https://github.com/nestjs/config) — маленький пакет, легко прочитать целиком
- [Joi](https://github.com/hapijs/joi), [zod](https://zod.dev)
- [The Twelve-Factor App: Config](https://12factor.net/config) — теоретическая база, на которую можно сослаться на собесе
