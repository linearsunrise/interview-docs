---
sidebar_position: 7
title: Pipes и валидация
---

# Pipes и валидация

> **TL;DR:** Pipe делает две вещи: **валидирует** вход (кидает исключение) или **трансформирует** его (строка → число, plain object → class instance). Выполняется последним перед хендлером, на уровне параметров.

## Встроенные

`ValidationPipe`, `ParseIntPipe`, `ParseUUIDPipe`, `ParseEnumPipe`, `ParseArrayPipe`, `DefaultValuePipe`, `ParseFilePipe`.

```ts
@Get(':id')
findOne(@Param('id', ParseIntPipe) id: number) {} // 'abc' → 400 автоматически
```

## ValidationPipe — как он работает внутри

1. `class-transformer.plainToInstance()` — превращает JSON в инстанс DTO-класса.
2. `class-validator.validate()` — проверяет декораторы (`@IsString()`, `@IsEmail()`, ...).
3. Ошибки → `BadRequestException` со списком нарушений.

Опции, которые надо знать наизусть:

```ts
app.useGlobalPipes(new ValidationPipe({
  whitelist: true,            // выкидывает поля, которых нет в DTO
  forbidNonWhitelisted: true, // ...или кидает 400 при лишних полях
  transform: true,            // возвращает инстанс DTO + приводит примитивы
  transformOptions: { enableImplicitConversion: true },
}));
```

Без `whitelist` возможен **mass assignment** — клиент подсунет `isAdmin: true`, и объект пройдёт дальше. Это security-вопрос, любят спрашивать.

## Кастомный pipe (zod)

```ts
export class ZodValidationPipe implements PipeTransform {
  constructor(private schema: ZodSchema) {}
  transform(value: unknown, metadata: ArgumentMetadata) {
    const result = this.schema.safeParse(value);
    if (!result.success) throw new BadRequestException(result.error.format());
    return result.data;
  }
}

@Post()
create(@Body(new ZodValidationPipe(createCatSchema)) dto: CreateCatDto) {}
```

## Что спрашивают на собеседовании

1. **Что делает `transform: true`?** — без него в хендлер приходит plain object (декораторы отвалидировали, но `instanceof Dto === false`), с ним — настоящий инстанс + конверсия типов из query/params.
2. **Чем опасен ValidationPipe без `whitelist`?** — mass assignment: лишние поля проходят в сервис/ORM.
3. **Порядок выполнения pipes?** — глобальные → контроллера → метода → параметра; pipe параметра получает результат предыдущих.
4. **Как отвалидировать query-параметры?** — тот же DTO + `@Query()`, primитивы конвертирует `enableImplicitConversion` или `@Type(() => Number)`.
5. **class-validator vs zod?** — декораторы vs схемы; zod даёт вывод типов из схемы (single source of truth) и работает без `emitDecoratorMetadata`.

## Ссылки

- [Pipes — официальная документация](https://docs.nestjs.com/pipes)
- [Validation techniques](https://docs.nestjs.com/techniques/validation) — все опции ValidationPipe
- [class-validator](https://github.com/typestack/class-validator), [class-transformer](https://github.com/typestack/class-transformer), [zod](https://zod.dev)
- Исходники: [`packages/common/pipes/validation.pipe.ts`](https://github.com/nestjs/nest/blob/master/packages/common/pipes/validation.pipe.ts) — видно шаги plainToInstance → validate
