---
sidebar_position: 6
title: Архитектура и паттерны
---

# Архитектура и паттерны

Самый важный блок для senior: здесь проверяют мышление, а не память.

## Подтемы

1. [SOLID с примерами из бэкенда](./01-solid.md) — не фигуры, а сервисы заказов/оплаты/уведомлений
2. [Слоистая архитектура: controller → service → repository](./02-layered-architecture.md) — направление зависимостей, анемичная модель
3. [Clean/Hexagonal: ports & adapters](./03-clean-hexagonal.md) — независимость домена от фреймворка, цена в Nest
4. [DDD: entity, value object, агрегат, bounded context](./04-ddd.md) — инварианты, почему один агрегат — одна транзакция
5. [CQRS](./05-cqrs.md) — когда оправдано, уровни разделения, eventual consistency в UI
6. [Event Sourcing](./06-event-sourcing.md) — проекции, снапшоты, почему это редко нужно
7. [Монолит vs модульный монолит vs микросервисы](./07-monolith-vs-microservices.md) — критерии перехода, Strangler Fig
8. [Межсервисная коммуникация: sync vs async](./08-sync-vs-async-communication.md) — temporal coupling, оркестрация vs хореография
9. [Saga: choreography vs orchestration, компенсации](./09-saga.md) — заказ → оплата → склад → доставка
10. [Transactional Outbox + relay/CDC](./10-transactional-outbox.md) — dual write problem, поллер vs Debezium
11. [Circuit Breaker, retry + backoff + jitter](./11-circuit-breaker-retry.md) — retry storm, timeout budget
12. [API Gateway, BFF, service discovery](./12-api-gateway-bff.md) — client-side vs server-side discovery
13. [Kafka vs RabbitMQ](./13-kafka-vs-rabbitmq.md) — log vs queue, ordering, consumer groups
14. [Delivery guarantees, идемпотентные консьюмеры](./14-delivery-guarantees-idempotent-consumers.md) — at-least-once, дедупликация
15. [Distributed transactions: почему 2PC избегают](./15-distributed-transactions-2pc.md) — блокирующая неопределённость, CAP

## Чеклист знаний

- [ ] SOLID с примерами из бэкенда (не «квадрат-прямоугольник», а реальные сервисы)
- [ ] Слоистая архитектура: controller → service → repository, где границы и зачем
- [ ] Clean/Hexagonal: ports & adapters, независимость домена от фреймворка; цена этого в Nest
- [ ] DDD: entity vs value object, агрегат и его инварианты, bounded context, ubiquitous language
- [ ] CQRS: когда разделение читающей/пишущей модели оправдано, `@nestjs/cqrs`
- [ ] Event Sourcing: суть, проекции, снапшоты — и почему это редко нужно
- [ ] Монолит vs модульный монолит vs микросервисы: критерии перехода
- [ ] Межсервисная коммуникация: sync vs async, оркестрация vs хореография
- [ ] Saga: choreography vs orchestration, компенсации
- [ ] Transactional Outbox + relay/CDC — надёжная публикация событий
- [ ] Circuit Breaker, retry с exponential backoff + jitter, timeout budget
- [ ] API Gateway, BFF, service discovery
- [ ] Kafka vs RabbitMQ: модель (log vs queue), ordering, consumer groups, гарантии доставки
- [ ] At-least-once / at-most-once / effectively-once; дедупликация и идемпотентные консьюмеры
- [ ] Distributed transactions: почему 2PC избегают

## Вопросы с собеседований

1. Когда микросервисы — ошибка? Какие проблемы монолита они реально решают, а какие создают?
2. Сервис A пишет в свою БД и должен уведомить сервис B. Что не так с «записал в БД, потом отправил в Kafka»? (dual write) Как чинит outbox?
3. Заказ → оплата → резерв склада → доставка. Спроектируйте сагу. Что при падении на шаге 3?
4. Kafka vs RabbitMQ — как выберете для конкретной задачи? Как в Kafka гарантируется порядок и что такое consumer group rebalance?
5. Consumer получил сообщение дважды — как сделать обработку идемпотентной?
6. Что такое Circuit Breaker, состояния, чем отличается от retry? Почему retry без backoff и jitter опасен (retry storm)?
7. Расскажите про агрегат в DDD. Почему «один агрегат — одна транзакция»?
8. CQRS: какие проблемы приносит eventual consistency между моделями чтения и записи, как жить с ней в UI?
9. Как бы вы распилили монолит? С чего начать (границы, strangler fig)?
10. Хореография vs оркестрация саг — trade-offs (связность, наблюдаемость, сложность отладки).

## Ключевые тезисы

**Dual write:** запись в БД и отправка в брокер — две системы без общей транзакции; между ними процесс может упасть → потерянное или лишнее событие. **Outbox:** событие пишется в таблицу outbox в той же транзакции, что и бизнес-данные; отдельный relay (поллер или CDC/Debezium) публикует в брокер. Даёт at-least-once → консьюмер обязан быть идемпотентным.

**Идемпотентный консьюмер:** таблица processed_message_ids (unique) + обработка и вставка id в одной транзакции; либо естественная идемпотентность операции (UPSERT).

**Микросервисы:** решают организационное масштабирование (независимые команды/деплой/скейлинг), платят распределённостью: сеть, консистентность, наблюдаемость, контракты. Дефолт для новой системы — модульный монолит с чёткими границами; выделять сервисы по мере доказанной необходимости.

**Saga:** последовательность локальных транзакций + компенсации при откате. Оркестрация — центральный координатор (проще наблюдать, единая точка знаний), хореография — события (меньше связность, сложнее видеть весь флоу).

## Красные флаги

- «Микросервисы — это современно, монолит — легаси»
- Не знает про dual write problem
- «Kafka гарантирует exactly-once сама» (без оговорок про транзакции/идемпотентность)
- SOLID рассказывает только на абстрактных фигурах

## Практика

- Реализовать outbox в Nest: транзакция с записью события + поллер, публикующий в RabbitMQ/Kafka
- Написать идемпотентный консьюмер с дедупликацией
- Взять свой рабочий проект и вслух обосновать его архитектуру + что бы изменил — это готовый ответ на интервью
