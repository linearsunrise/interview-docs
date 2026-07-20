---
sidebar_position: 5
title: API и коммуникации
---

# API и коммуникации

## Подтемы

1. [REST: методы, идемпотентность, коды ответов](./01-rest-semantics.md) — семантика PUT/PATCH/POST, 409 vs 422, 401 vs 403
2. [Idempotency keys для POST](./02-idempotency-keys.md) — схема на БД, конкурентные запросы с одним ключом, TTL
3. [Версионирование API](./03-api-versioning.md) — URI vs header, expand-contract, что считается breaking change
4. [Пагинация, фильтрация, сортировка — дизайн контрактов](./04-pagination-filtering-contracts.md) — непрозрачный курсор, whitelist полей сортировки, лимиты
5. [HTTP: keep-alive, кеширующие заголовки, сжатие](./05-http-caching-keepalive.md) — Cache-Control, ETag/304, gzip/brotli
6. [HTTP/2, HTTP/3](./06-http2-http3.md) — мультиплексирование, HOL blocking на TCP, QUIC
7. [GraphQL](./07-graphql.md) — schema-first vs code-first, DataLoader и N+1, complexity/depth limits
8. [WebSockets](./08-websockets.md) — handshake, sticky sessions vs Redis adapter, heartbeat
9. [SSE vs WebSocket vs long polling](./09-sse-vs-websocket-vs-polling.md) — критерии выбора по направлению потока
10. [gRPC](./10-grpc.md) — protobuf, 4 типа стриминга, deadlines, когда вместо REST
11. [Webhooks](./11-webhooks.md) — HMAC-подпись, retries провайдера, идемпотентность на приёмнике
12. [OpenAPI/Swagger в Nest, contract-first](./12-openapi-swagger.md) — code-first из тех же DTO, генерация клиентов

## Чеклист знаний

- [ ] REST: семантика методов, идемпотентность, коды ответов (когда 409, 422, 429)
- [ ] Idempotency keys для POST (платежи, создание заказов)
- [ ] Версионирование API: URI vs header vs без версий (expand-contract)
- [ ] Пагинация, фильтрация, сортировка — дизайн контрактов
- [ ] HTTP: keep-alive, кеширующие заголовки (ETag, Cache-Control), сжатие
- [ ] HTTP/2, HTTP/3 — что дают (мультиплексирование, HOL blocking)
- [ ] GraphQL: schema-first vs code-first в Nest, резолверы, N+1 и DataLoader, complexity limits
- [ ] WebSockets: handshake, масштабирование (sticky sessions vs Redis adapter), heartbeat
- [ ] SSE vs WebSocket vs long polling — критерии выбора
- [ ] gRPC: protobuf, типы стриминга, deadlines, когда брать вместо REST
- [ ] Webhooks: подпись, retries, идемпотентность на приёмнике
- [ ] OpenAPI/Swagger в Nest, contract-first разработка

## Вопросы с собеседований

1. PUT vs PATCH vs POST — семантика и идемпотентность. Почему идемпотентность важна при ретраях?
2. Как сделать POST /payments безопасным для повторов? (idempotency key: хранение, TTL, конкурентные запросы с одним ключом)
3. Клиент дважды кликнул «оплатить» — какие слои защиты?
4. Как правильно ломать обратную совместимость API? Стратегия deprecation.
5. GraphQL: как DataLoader чинит N+1? Почему это per-request инстанс?
6. Как защитить GraphQL от тяжёлых запросов? (depth/complexity limit, persisted queries)
7. Чат на WebSockets, 3 инстанса за балансером — как доставить сообщение пользователю на другом инстансе?
8. Когда SSE лучше WebSocket?
9. gRPC vs REST для внутренних сервисов — trade-offs (перф, контракты, отладка, браузер).
10. Проектируете webhook-провайдер: как гарантировать доставку и защитить получателя?

## Ключевые тезисы

**Idempotency key:** клиент шлёт уникальный ключ в заголовке → сервер атомарно сохраняет ключ (unique constraint) до выполнения операции → при повторе отдаёт сохранённый результат. Конкурентный дубль ловится на unique violation → отдать 409 или дождаться результата первого.

**WebSocket на N инстансов:** соединение живёт на конкретном инстансе → нужен pub/sub-слой (Redis adapter для socket.io) — каждый инстанс подписан и доставляет своим сокетам. Sticky sessions решают только handshake, не доставку.

**gRPC:** сильные типизированные контракты, HTTP/2, стриминг, меньше payload — но хуже с браузером (нужен grpc-web/gateway), сложнее дебаг. Хорош межсервисно, REST — наружу.

## Красные флаги

- «PATCH и PUT — одно и то же»
- Не может объяснить, зачем idempotency при at-least-once доставке
- «WebSocket масштабируется сам»

## Практика

- Реализовать idempotency-key middleware/interceptor в Nest с хранением в Redis
- Собрать GraphQL-модуль с DataLoader и убедиться в исчезновении N+1 по логам запросов
- Socket.io + Redis adapter на двух инстансах: доставка сообщения между инстансами
