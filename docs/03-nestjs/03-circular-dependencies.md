---
sidebar_position: 3
title: Циклические зависимости
---

# Циклические зависимости

> **TL;DR:** A зависит от B, B зависит от A. `forwardRef()` заставит это работать, но цикл — почти всегда признак кривой границы между модулями. Лечится выносом общей логики или заменой прямого вызова на события.

## Симптом

```
Nest can't resolve dependencies of the AService (?). 
Please make sure that the argument dependency at index [0] 
is available in the AModule context.
```

Часто выглядит как «зависимость есть, но undefined» — потому что в момент резолва один из классов ещё не определён (JS-модули грузятся циклом).

## Костыль: forwardRef

```ts
// с обеих сторон!
@Injectable()
export class AService {
  constructor(@Inject(forwardRef(() => BService)) private b: BService) {}
}
// и на уровне модулей:
@Module({ imports: [forwardRef(() => BModule)] })
```

`forwardRef` откладывает резолв токена до момента, когда оба класса уже определены. Работает, но: скрывает проблему дизайна, делает порядок инициализации хрупким, ломается при рефакторинге.

## Как лечить по-настоящему

1. **Вынести общее в третий сервис/модуль** — чаще всего цикл значит, что у A и B есть общая ответственность C.
2. **События вместо прямого вызова** — `@nestjs/event-emitter`: A эмитит `order.created`, B подписан. Зависимость исчезает.
3. **`ModuleRef`** — получить зависимость лениво в момент вызова, а не в конструкторе (точечный костыль лучше forwardRef).
4. **Проверить баррел-файлы** — `index.ts`, реэкспортирующий всё подряд, — типичный *скрытый* источник циклов на уровне файлов ("A file circular dependency"). Импортируйте из конкретных файлов.

Детект: [madge](https://github.com/pahen/madge) (`madge --circular src`) или правило `import/no-cycle` в eslint-plugin-import.

## Что спрашивают на собеседовании

1. **Почему forwardRef — плохое долгосрочное решение?** — не убирает связность, а маскирует её; хрупкий порядок инициализации; сигнал о неверных границах модулей.
2. **Как разрулить цикл между OrderService и NotificationService?** — событиями: заказ не должен знать о нотификациях.
3. **Чем file-level цикл отличается от DI-цикла?** — file-level (через баррелы) ломает загрузку JS-модулей и даёт `undefined` в декораторах ещё до DI; DI-цикл — про граф провайдеров.
4. **Как найти циклы в большом проекте?** — madge, eslint `import/no-cycle`, анализ графа модулей.

## Ссылки

- [Circular dependency — официальная документация](https://docs.nestjs.com/fundamentals/circular-dependency) (там же предупреждение про баррел-файлы)
- [ModuleRef](https://docs.nestjs.com/fundamentals/module-ref)
- [Events (@nestjs/event-emitter)](https://docs.nestjs.com/techniques/events)
- [madge](https://github.com/pahen/madge), [eslint-plugin-import `no-cycle`](https://github.com/import-js/eslint-plugin-import/blob/main/docs/rules/no-cycle.md)
