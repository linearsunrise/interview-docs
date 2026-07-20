---
sidebar_position: 4
title: Request lifecycle
---

# Request lifecycle

> **TL;DR:** middleware → guards → interceptors (pre) → pipes → handler → interceptors (post) → exception filters. Это вопрос №1 на собеседованиях по NestJS — надо отвечать без запинки, включая порядок внутри каждого слоя.

## Полный порядок

```
Запрос
 → Middleware        (глобальные → модульные)
 → Guards            (глобальные → контроллера → метода)
 → Interceptors PRE  (глобальные → контроллера → метода)
 → Pipes             (глобальные → контроллера → метода → параметра)
 → Controller handler
 → Service (бизнес-логика)
 → Interceptors POST (метода → контроллера → глобальные)  ← обратный порядок!
 → Exception filters (при ошибке: метода → контроллера → глобальные)
Ответ
```

Ключевые мнемоники:
- **Guards и pipes — сверху вниз** (global → controller → method).
- **Interceptors — луковица**: pre-фаза сверху вниз, post-фаза снизу вверх (как middleware-обёртки, `next.handle()` — это «дырка» в середине).
- **Filters — снизу вверх**: срабатывает самый специфичный подходящий фильтр, а не все.

## Регистрация глобальных и DI

```ts
app.useGlobalGuards(new AuthGuard()); // ❌ нет DI — создаёте руками
// ✅ через токен — guard получает зависимости из контейнера:
{ provide: APP_GUARD, useClass: AuthGuard }
```

То же самое: `APP_PIPE`, `APP_INTERCEPTOR`, `APP_FILTER`. Это частый follow-up вопрос: «а как глобальному guard'у заинжектить сервис?»

## Зоны ответственности (кому что)

| Слой | Для чего | Не для чего |
|---|---|---|
| Middleware | raw body, cors, helmet, логирование до роутинга | авторизация (нет метаданных хендлера) |
| Guard | authN/authZ — «пускать или нет» | трансформация данных |
| Interceptor | логирование времени, кэш, маппинг ответа, timeout | валидация |
| Pipe | валидация и трансформация входа | side-эффекты |
| Filter | маппинг исключений в HTTP-ответ | бизнес-логика |

## Что спрашивают на собеседовании

1. **Полный порядок?** — см. схему; обязательно упомянуть обратный порядок post-interceptors и фильтров.
2. **Глобальный, контроллерный и метод-левел guard — кто первый?** — глобальный; если он вернул `false`, остальные не выполняются.
3. **Почему авторизация в guard, а не в middleware?** — у guard есть `ExecutionContext`: известно, какой хендлер сработает, доступны его метаданные (`@Roles()` через `Reflector`). Middleware выполняется до роутинга и ничего этого не знает.
4. **Выполнятся ли interceptors, если guard отклонил запрос?** — нет, guards идут раньше. А вот exception filter — сработает.
5. **Где перехватить ошибку из guard?** — в exception filter (interceptor'ы pre-фазы уже позади, catchError их не поймает).

## Ссылки

- [Request lifecycle — официальный FAQ](https://docs.nestjs.com/faq/request-lifecycle) — канонический ответ, стоит выучить близко к тексту
- Исходники: [`packages/core/router/router-execution-context.ts`](https://github.com/nestjs/nest/blob/master/packages/core/router/router-execution-context.ts) — метод `create()` буквально показывает порядок: `canActivate` → interceptors → pipes → handler
- [`packages/core/guards/guards-consumer.ts`](https://github.com/nestjs/nest/blob/master/packages/core/guards/guards-consumer.ts), [`packages/core/interceptors/interceptors-consumer.ts`](https://github.com/nestjs/nest/blob/master/packages/core/interceptors/interceptors-consumer.ts)
