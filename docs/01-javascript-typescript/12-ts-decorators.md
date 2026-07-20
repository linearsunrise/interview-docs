---
sidebar_position: 12
title: "TS: декораторы и reflect-metadata"
---

# Декораторы и `reflect-metadata`

> **TL;DR:** В экосистеме одновременно живут два поколения декораторов: **legacy** (`experimentalDecorators`, старый TC39-черновик, на нём стоят NestJS и Angular) и **standard** (TS 5.0+, финальный TC39 stage-3 proposal, другая сигнатура, никакой встроенной метадаты). Nest использует legacy-декораторы вместе с `emitDecoratorMetadata` и библиотекой `reflect-metadata`, чтобы на рантайме доставать типы параметров конструктора — это и есть фундамент DI (см. [DI-контейнер и провайдеры](../03-nestjs/01-di-and-providers.md)).

## Два несовместимых мира

```json
// tsconfig.json — legacy (то, что использует NestJS)
{
  "compilerOptions": {
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true
  }
}
```

| | Legacy (`experimentalDecorators`) | Standard (TS 5.0+, без флага) |
|---|---|---|
| Сигнатура method-декоратора | `(target, propertyKey, descriptor)` | `(value, context)` |
| Метадата типов (`design:paramtypes`) | есть, через `emitDecoratorMetadata` | нет встроенной — не часть спеки |
| Параметр-декораторы | есть | отсутствуют как отдельный вид |
| Статус | старый TC39-черновик, "заморожен" | TC39 stage 3, финализируется в рантаймах |
| Кто использует | NestJS, Angular, TypeORM, class-validator | новые библиотеки, ориентированные на native decorators |

Nest **не может** просто перейти на standard-декораторы: вся DI-механика завязана на `design:paramtypes`, которого в новом стандарте нет и не планируется (комитет сознательно вынес метадату за скобки спецификации).

## Как декоратор получает типы параметров

```ts
import 'reflect-metadata';

@Injectable()
class OrdersService {
  constructor(private readonly db: DbConnection, private readonly logger: Logger) {}
}
```

С `emitDecoratorMetadata: true` компилятор при виде класса с хотя бы одним декоратором генерирует вызов `Reflect.defineMetadata('design:paramtypes', [DbConnection, Logger], OrdersService)` — массив **конструкторов** (не строк!) параметров. `reflect-metadata` — полифилл `Reflect.defineMetadata`/`getMetadata`, которого нет в спецификации ECMA-262 (это отдельное TC39-предложение, не входящее в основной язык). Именно отсюда Nest на старте узнаёт, что инжектить — без единой строки описания зависимостей руками.

## Свой декоратор с метадатой

```ts
function LogCalls(): MethodDecorator {
  return (target, key, descriptor: PropertyDescriptor) => {
    const original = descriptor.value;
    descriptor.value = function (...args: any[]) {
      console.log(`${String(key)} called with`, args);
      return original.apply(this, args);
    };
  };
}

function Roles(...roles: string[]): ClassDecorator {
  return (target) => Reflect.defineMetadata('roles', roles, target);
}

@Roles('admin')
class AdminController {
  @LogCalls()
  deleteUser(id: string) { /* ... */ }
}

Reflect.getMetadata('roles', AdminController); // ['admin'] — так Nest читает @Roles() через Reflector
```

Это ровно механизм, на котором в Nest построены `@Roles()` + `Reflector.get()` в guard'ах (см. [Guards](../03-nestjs/06-guards.md)) и кастомные декораторы параметров (см. [Кастомные декораторы](../03-nestjs/10-custom-decorators.md)).

## Порядок выполнения при нескольких декораторах

```ts
@first()
@second()
class Example {}
// порядок вычисления фабрик (top-down): first(), затем second()
// порядок ПРИМЕНЕНИЯ результата (bottom-up): second-декоратор применяется первым, затем first
```

Аналогия с композицией функций: `first(second(Example))` — самый близкий к классу декоратор применяется первым.

## Что спрашивают на собеседовании

1. **Чем legacy-декораторы отличаются от стандартных (TS 5.0)?** — разные сигнатуры колбэков, разный статус в TC39, и главное — у standard-декораторов нет автоматической эмиссии метадаты типов, на которой держится DI в Nest/Angular.
2. **Как NestJS узнаёт, какой сервис инжектить в конструктор?** — `emitDecoratorMetadata` заставляет TS записать типы параметров конструктора через `Reflect.defineMetadata('design:paramtypes', ...)`, DI-контейнер читает это на старте.
3. **Почему нельзя инжектить по интерфейсу?** — интерфейсы существуют только на этапе компиляции и стираются; в `design:paramtypes` попадёт `Object`, а не конкретный тип — нужен класс или токен с `@Inject()`.
4. **В каком порядке выполняются несколько декораторов на одном классе/методе?** — фабрики вычисляются сверху вниз, но обёртывание (применение к цели) идёт снизу вверх — как вложенные вызовы функций.
5. **Что такое `reflect-metadata` и почему это не часть языка?** — полифилл отдельного TC39-предложения (`Reflect.defineMetadata`/`getMetadata`), не вошедшего в основную спецификацию; подключается вручную (`import 'reflect-metadata'` один раз в точке входа).

## Ссылки

- [TS Handbook — Decorators](https://www.typescriptlang.org/docs/handbook/decorators.html) (legacy) и [TS 5.0 Release Notes — Decorators](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-0.html#decorators) (standard)
- [TC39 — proposal-decorators](https://github.com/tc39/proposal-decorators) (standard, stage 3) и [TC39 — proposal-decorator-metadata](https://github.com/tc39/proposal-decorator-metadata)
- [reflect-metadata](https://github.com/rbuckton/reflect-metadata) — полифилл, на котором держится Nest DI
- [DI-контейнер и провайдеры в NestJS](../03-nestjs/01-di-and-providers.md), [Guards](../03-nestjs/06-guards.md), [Кастомные декораторы](../03-nestjs/10-custom-decorators.md)
