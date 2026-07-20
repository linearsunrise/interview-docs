---
sidebar_position: 9
title: "Очереди для сглаживания нагрузки: BullMQ"
---

# Очереди для сглаживания нагрузки (BullMQ): retries, backoff, DLQ, конкурентность

> **TL;DR:** Очередь превращает "принять запрос и сразу его полностью обработать" в "быстро принять запрос и отдать подтверждение, обработать — когда появится ёмкость" — тот же принцип развязки по времени, что разобран для async-коммуникации между сервисами (см. [06-architecture-patterns/08-sync-vs-async-communication.md](../06-architecture-patterns/08-sync-vs-async-communication.md)), только здесь — внутри одного приложения, между HTTP-слоем и фоновой обработкой. BullMQ (поверх Redis) — стандартный выбор для этого в Node-экосистеме: даёт retries, backoff, конкурентность и dead letter queue почти из коробки, не требуя отдельной инфраструктуры messaging-брокера.

## Базовая схема — producer/worker

```ts
import { Queue, Worker } from 'bullmq';

const emailQueue = new Queue('emails', { connection: redisConnection });

// producer: HTTP-хендлер быстро ставит задачу в очередь и отвечает клиенту
app.post('/orders', async (req, res) => {
  const order = await orderService.create(req.body);
  await emailQueue.add('order-confirmation', { orderId: order.id }); // не ждёт отправки письма
  res.status(201).json(order); // клиент получил ответ сразу, письмо уйдёт асинхронно
});

// worker: отдельный процесс/часть процесса, разбирающая задачи из очереди
const worker = new Worker('emails', async (job) => {
  await sendConfirmationEmail(job.data.orderId); // тяжёлая/медленная операция — здесь, не в HTTP-хендлере
}, { connection: redisConnection, concurrency: 10 });
```

HTTP-запрос пользователя не ждёт реальной отправки письма (сетевой вызов к email-провайдеру, потенциально медленный/нестабильный) — он получает ответ, как только задача гарантированно поставлена в очередь. Это снижает latency ответа пользователю и изолирует нестабильность email-провайдера от основного пути создания заказа (тот же аргумент "не блокировать критичный путь некритичной зависимостью", что и в graceful degradation, см. [08-bulkhead-load-shedding-degradation.md](./08-bulkhead-load-shedding-degradation.md)).

## Retries и backoff — то же самое, глубже разобрано в контексте межсервисных вызовов

```ts
await emailQueue.add('order-confirmation', { orderId: order.id }, {
  attempts: 5,
  backoff: { type: 'exponential', delay: 1000 }, // 1с, 2с, 4с, 8с, 16с между попытками
});
```

Механика ровно та же, что для retry между сервисами (см. [06-architecture-patterns/11-circuit-breaker-retry.md](../06-architecture-patterns/11-circuit-breaker-retry.md)) — экспоненциальный рост задержки между попытками, ограниченное число попыток, идемпотентность обработчика job обязательна (worker может упасть и перезапуститься на середине, BullMQ гарантирует **at-least-once** доставку job, не exactly-once — тот же принцип, что для консьюмеров сообщений, см. [06-architecture-patterns/14-delivery-guarantees-idempotent-consumers.md](../06-architecture-patterns/14-delivery-guarantees-idempotent-consumers.md)).

## Dead Letter Queue (DLQ) — куда деваются окончательно провалившиеся job

```ts
const worker = new Worker('emails', processJob, { connection: redisConnection });

worker.on('failed', async (job, err) => {
  if (job && job.attemptsMade >= job.opts.attempts!) {
    await deadLetterQueue.add('failed-email', { originalJob: job.data, error: err.message, failedAt: new Date() });
  }
});
```

После исчерпания всех разрешённых попыток job не должна просто молча исчезнуть — это означает потерю данных (письмо так и не отправлено, и никто об этом не узнал). DLQ — отдельная очередь/хранилище для job, которые исчерпали лимит retries — обычно с алертом (Sentry/PagerDuty), чтобы человек мог разобраться, почему job систематически проваливается (баг в обработчике, невалидные входные данные конкретной job, или временная, но продолжительная недоступность внешней зависимости), и при необходимости вручную заново поставить job в очередь после исправления причины.

## Конкурентность — сколько job обрабатывается параллельно

```ts
const worker = new Worker('emails', processJob, {
  connection: redisConnection,
  concurrency: 10, // до 10 job параллельно на ЭТОМ воркер-процессе
});
```

`concurrency` контролирует, сколько job этот конкретный worker-процесс обрабатывает одновременно — не число отдельных worker-процессов (тех можно поднять несколько, каждый со своим `concurrency`, для дополнительного горизонтального масштабирования обработки очереди). Выбор значения — trade-off между пропускной способностью (выше concurrency — больше job обрабатывается параллельно) и нагрузкой на зависимости, которые эти job используют (10 параллельных job, каждая из которых бьёт в внешний email API — может упереться в rate limit самого провайдера, или, если job работают с БД, — в лимит пула соединений, см. [04-databases/08-connection-pooling.md](../04-databases/08-connection-pooling.md); тот же принцип bulkhead — изолированный пул под конкретный тип фоновой обработки).

## Проектирование фоновой обработки — 100k писем

```ts
async function sendBulkEmails(userIds: string[]) {
  const jobs = userIds.map((userId) => ({
    name: 'bulk-email',
    data: { userId, campaignId },
    opts: { attempts: 3, backoff: { type: 'exponential', delay: 2000 }, jobId: `campaign:${campaignId}:${userId}` },
  }));
  await emailQueue.addBulk(jobs); // batch-добавление эффективнее 100k отдельных .add()
}
```

- **`jobId`** — явный детерминированный ID (не случайный) даёт естественную идемпотентность на уровне самой очереди: BullMQ не добавит job с уже существующим `jobId` повторно — повторный вызов `sendBulkEmails` (например, из-за ретрая на уровне вызывающего кода) не создаст дублирующиеся письма одному пользователю.
- **`addBulk`** вместо цикла с отдельными `.add()` — один batch-запрос к Redis вместо 100 000 отдельных round-trip'ов, заметная разница в производительности постановки в очередь.
- **`concurrency`** воркера подбирается под реальный лимит email-провайдера (rate limit API), не произвольно высоко — иначе воркер сам создаёт себе проблему, упираясь в 429 от провайдера чаще, чем реально отправляет письма.
- **Retries + DLQ** — письма, не отправившиеся после исчерпания попыток (невалидный email, постоянная ошибка провайдера для конкретного адреса), не теряются молча, а попадают в DLQ для разбора, не блокируя обработку остальных 99 999 писем кампании.

## Что спрашивают на собеседовании

1. **Спроектируйте фоновую обработку: отправка 100k писем.** — batch-постановка в очередь (`addBulk`), детерминированный `jobId` для идемпотентности, `concurrency` воркера под реальный rate limit провайдера, retries с backoff, DLQ для окончательно провалившихся job без блокировки остальной кампании.
2. **Зачем нужна очередь, если можно просто отправить письмо прямо в HTTP-хендлере создания заказа?** — развязка по времени: пользователь не ждёт медленный/нестабильный внешний вызов (email-провайдер), latency ответа не зависит от него, а нестабильность провайдера не приводит к ошибке основной операции (создания заказа).
3. **Что происходит с job, которая проваливается систематически после всех retries?** — попадает в Dead Letter Queue вместо молчаливого исчезновения — обычно с алертом, чтобы человек разобрался в причине и при необходимости переобработал вручную после исправления.
4. **Гарантирует ли BullMQ exactly-once обработку job?** — нет, at-least-once — воркер может упасть и перезапуститься на середине обработки job; обработчик обязан быть идемпотентным, тот же принцип, что для консьюмеров сообщений в целом.
5. **Как выбрать значение `concurrency` для воркера?** — под реальную пропускную способность зависимостей, которые job использует (rate limit внешнего API, лимит пула соединений к БД) — не произвольно высоко, иначе воркер сам создаёт себе узкое место, упираясь в лимиты зависимостей быстрее, чем реально продвигается обработка.

## Ссылки

- [BullMQ — официальная документация](https://docs.bullmq.io/)
- [BullMQ — Retrying failing jobs](https://docs.bullmq.io/guide/retrying-failing-jobs)
- [BullMQ — Rate limiting](https://docs.bullmq.io/guide/rate-limiting)
- [Delivery guarantees и идемпотентные консьюмеры](../06-architecture-patterns/14-delivery-guarantees-idempotent-consumers.md)
