---
sidebar_position: 10
title: Graceful shutdown
---

# Graceful shutdown

> **TL;DR:** Порядок: получить SIGTERM → перестать принимать новое (`server.close` + fail readiness) → дождаться in-flight запросов с таймаутом → закрыть БД/брокеры → выйти. В k8s обязательна пауза перед закрытием: под получает SIGTERM **раньше**, чем его уберут из endpoints, и без паузы часть трафика получит connection refused.

## Скелет

```js
const server = app.listen(3000);

let shuttingDown = false;
process.on('SIGTERM', async () => {
  if (shuttingDown) return;
  shuttingDown = true;

  healthcheck.setNotReady();                    // 1. readiness → fail
  await sleep(5_000);                           // 2. ждём, пока LB уберёт нас из ротации
  server.close(async () => {                    // 3. не принимаем новые, ждём активные
    await queueConsumer.stop();                 // 4. дообработать/вернуть в очередь сообщения
    await db.destroy();                         // 5. закрыть коннекты
    process.exit(0);
  });
  setTimeout(() => process.exit(1), 30_000).unref(); // 6. жёсткий таймаут-страховка
});
```

## Нюансы, которые отличают senior-ответ

- **`server.close()` не рвёт keep-alive сокеты без активного запроса** — исторически их приходилось добивать вручную (трекать сокеты); с Node 18.2 есть [`server.closeIdleConnections()`](https://nodejs.org/api/http.html#servercloseidleconnections) и `closeAllConnections()`.
- **k8s-гонка:** удаление пода из Endpoints и доставка SIGTERM идут **параллельно**; kube-proxy/ingress обновляются с лагом. Отсюда пауза (или `preStop: sleep`) перед `server.close`. `terminationGracePeriodSeconds` (default 30s) должен покрывать паузу + максимальный запрос.
- **Консьюмеры очередей:** остановить приём, дообработать текущие сообщения или отдать их обратно (nack) — иначе получите «зависшие» сообщения до visibility timeout.
- **SIGKILL перехватить нельзя** — если не уложились в grace period, всё, что не идемпотентно, может остаться в полусостоянии → проектировать обработчики идемпотентными.
- **`process.exit()` посреди работы** — обрывает event loop немедленно, недописанные логи/ответы теряются; сначала закрытие, exit — последним.

В NestJS всё это раскладывается по хукам: см. [Lifecycle hooks](../03-nestjs/13-lifecycle-hooks.md) (`enableShutdownHooks`, `onApplicationShutdown`).

## Что спрашивают на собеседовании

1. **Что и в каком порядке закрывать?** — readiness fail → пауза → server.close → консьюмеры → БД → exit; таймаут-страховка обязательна.
2. **Почему при деплое проскакивают 502/refused, хотя shutdown «graceful»?** — гонка SIGTERM vs endpoints; нужен sleep/preStop.
3. **Что с долгими запросами (стрим на 5 минут)?** — не влезут в grace period: либо увеличить его, либо уметь обрывать с ретраем на клиенте.
4. **Почему нельзя просто `process.exit(0)` по SIGTERM?** — обрыв in-flight запросов и коннектов, потеря буферизованных логов, полуобработанные сообщения очередей.
5. **Как это тестировать?** — нагрузка (autocannon) + SIGTERM, смотреть коды ответов; интеграционный тест на то, что все close-хуки вызвались.

## Ссылки

- [`server.close`](https://nodejs.org/api/http.html#serverclosecallback), [`closeIdleConnections`](https://nodejs.org/api/http.html#servercloseidleconnections)
- [Process signal events](https://nodejs.org/api/process.html#signal-events)
- [Kubernetes: Pod termination flow](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination) + [Container hooks (preStop)](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/)
- [terminus (@godaddy/terminus)](https://github.com/godaddy/terminus) — готовая обвязка healthcheck + graceful shutdown, полезно посмотреть исходники
