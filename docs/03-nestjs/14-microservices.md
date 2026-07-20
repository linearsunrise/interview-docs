---
sidebar_position: 14
title: Микросервисы
---

# Микросервисы

> **TL;DR:** `@nestjs/microservices` — транспортная абстракция: один код, разные брокеры (TCP, Redis, NATS, RabbitMQ, Kafka, gRPC). Два стиля: `@MessagePattern` — request/response, `@EventPattern` — fire-and-forget. Клиент: `ClientProxy.send()` / `.emit()`.

## Сервер (обработчик)

```ts
@Controller()
export class OrdersController {
  @MessagePattern({ cmd: 'get_order' })       // request/response
  getOrder(@Payload() id: number) { return this.repo.find(id); }

  @EventPattern('order_created')              // fire-and-forget
  handleCreated(@Payload() event: OrderCreatedEvent) { /* ... */ }
}

// main.ts
const app = await NestFactory.createMicroservice(AppModule, {
  transport: Transport.RMQ,
  options: { urls: ['amqp://localhost'], queue: 'orders' },
});
```

## Клиент

```ts
// ClientsModule.register([{ name: 'ORDERS', transport: Transport.RMQ, options: {...} }])
constructor(@Inject('ORDERS') private client: ClientProxy) {}

// send → холодный Observable: запрос уйдёт только при подписке!
const order = await firstValueFrom(this.client.send({ cmd: 'get_order' }, 42));

// emit → событие, ответ не ждём
this.client.emit('order_created', { id: 42 });
```

Классическая ловушка: вызвать `client.send()` и не подписаться — **ничего не отправится** (cold Observable).

## Hybrid application

HTTP-приложение, которое дополнительно слушает брокер:

```ts
const app = await NestFactory.create(AppModule);
app.connectMicroservice({ transport: Transport.REDIS, options: {...} });
await app.startAllMicroservices();
await app.listen(3000);
```

## Что ещё знать

- **Ошибки:** кидать `RpcException`; на клиенте она прилетит в error-канал Observable (`catchError`). Обычные exception filters не работают — нужен `@Catch(RpcException)` с `switchToRpc()`.
- **Kafka-специфика:** нужен `subscribeToResponseOf()` для request/response, consumer groups, партиции — порядок только внутри партиции.
- **Guards/interceptors/pipes работают** и в RPC-контексте — контекст через `context.switchToRpc().getData()/getContext()`.
- **Честный ответ про ограничения:** встроенный транспорт — это удобная абстракция, но для сложных сценариев (ретраи, DLQ, exactly-once) команды часто берут брокерные библиотеки напрямую (например, kafkajs/amqplib в кастомном транспортере).

## Что спрашивают на собеседовании

1. **`@MessagePattern` vs `@EventPattern`?** — req/res (клиент ждёт ответ) vs fire-and-forget (клиент не ждёт); send vs emit соответственно.
2. **Почему `send()` «не работает»?** — cold Observable, нужен subscribe/`firstValueFrom`.
3. **Как совместить HTTP API и обработку очереди в одном приложении?** — hybrid app: `connectMicroservice` + `startAllMicroservices`.
4. **Как обрабатываются ошибки между сервисами?** — `RpcException` → error-канал у клиента; таймауты добавлять самому (`timeout` оператор RxJS).
5. **Когда НЕ использовать @nestjs/microservices?** — когда нужны гарантии доставки/ретраи/DLQ тонкой настройки — прямая работа с брокером или отдельная шина (иногда это тоже валидный ответ, показывает зрелость).

## Ссылки

- [Microservices overview — официальная документация](https://docs.nestjs.com/microservices/basics)
- [Kafka](https://docs.nestjs.com/microservices/kafka), [RabbitMQ](https://docs.nestjs.com/microservices/rabbitmq), [gRPC](https://docs.nestjs.com/microservices/grpc)
- [Hybrid application — FAQ](https://docs.nestjs.com/faq/hybrid-application)
- [Exception filters для RPC](https://docs.nestjs.com/microservices/exception-filters)
- Исходники: [`packages/microservices/client/client-proxy.ts`](https://github.com/nestjs/nest/blob/master/packages/microservices/client/client-proxy.ts) — видно, как send строит cold Observable
