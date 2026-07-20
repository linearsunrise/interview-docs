---
sidebar_position: 10
title: "Метрики: Prometheus, RED/USE, гистограммы"
---

# Метрики: Prometheus, RED (rate, errors, duration) / USE, гистограммы для latency

> **TL;DR:** Prometheus работает по модели **pull** — сам периодически ходит и забирает метрики с эндпоинта `/metrics` каждого сервиса, а не сервисы сами их куда-то отправляют (push). RED — набор метрик для **request-driven** компонентов (API-сервисы: сколько запросов, сколько ошибок, сколько времени занимают). USE — для **ресурсов** (CPU, память, диск: насколько заняты, есть ли очередь ожидания, есть ли ошибки). Гистограммы, а не средние значения — тот же принцип, что и перцентили в [08-performance-reliability/10-load-testing-percentiles.md](../08-performance-reliability/10-load-testing-percentiles.md), применённый к постоянному мониторингу, а не разовому нагрузочному тесту.

## Pull-модель Prometheus

```ts
import client from 'prom-client';
const register = new client.Registry();
client.collectDefaultMetrics({ register }); // CPU, memory, event loop lag и т.п. из коробки

app.get('/metrics', async (req, res) => {
  res.set('Content-Type', register.contentType);
  res.send(await register.metrics());
});
```

```yaml
# prometheus.yml — сервер САМ приходит и забирает метрики по расписанию
scrape_configs:
  - job_name: 'myapp'
    scrape_interval: 15s
    static_configs: [{ targets: ['myapp-service:3000'] }]
```

Приложение просто выставляет текущее состояние своих метрик на HTTP-эндпоинте `/metrics` в текстовом формате — Prometheus сам решает, когда и как часто их забирать (`scrape_interval`). Это отличается от push-моделей (StatsD и подобные, где приложение само отправляет метрики наружу) практическими следствиями: не нужно приложению знать адрес системы мониторинга и заботиться о доставке (просто отдать текущее состояние по запросу), Prometheus сам обнаруживает недоступность сервиса (не смог сделать scrape — сервис `down`), что даёт встроенную детекцию отказа без отдельного heartbeat-механизма.

## RED — для request-driven сервисов (типичный API)

```
Rate:     сколько запросов в секунду сервис обрабатывает
Errors:   какая доля из них завершается ошибкой
Duration: сколько времени занимает обработка (в виде распределения/перцентилей, не среднего)
```

```ts
const httpRequestDuration = new client.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status_code'],
  buckets: [0.01, 0.05, 0.1, 0.5, 1, 2, 5],
});

app.use((req, res, next) => {
  const end = httpRequestDuration.startTimer({ method: req.method, route: req.route?.path });
  res.on('finish', () => end({ status_code: res.statusCode }));
  next();
});
```

Три метрики RED — минимальный набор, отвечающий на самый частый вопрос при инциденте: "сервис вообще нормально работает прямо сейчас?" — Rate показывает, есть ли трафик вообще (провал до нуля — сам по себе инцидент, независимо от errors/duration), Errors — растёт ли доля неуспехов, Duration — растёт ли задержка. Разбивка по `labelNames` (`route`, `status_code`, `method`) позволяет локализовать проблему до конкретного эндпоинта, а не видеть только агрегат по всему сервису разом.

## USE — для ресурсов (не request-driven, а сами по себе сущности)

```
Utilization: какая доля ресурса занята (CPU busy %, memory used %)
Saturation:  насколько ресурс перегружен сверх номинальной ёмкости (очередь ожидания CPU, swap-активность)
Errors:      количество ошибок этого ресурса (не запросов приложения — именно ресурса: disk I/O errors и т.п.)
```

Применяется к инфраструктурным ресурсам (CPU узла, диск, сеть, пул соединений к БД — см. [04-databases/08-connection-pooling.md](../04-databases/08-connection-pooling.md) про то, почему пул сам по себе — ограниченный ресурс с собственной насыщенностью), а не к бизнес-логике запросов, как RED. Практическая разница между Utilization и Saturation важна: CPU может быть занят на 60% (Utilization) без единого признака проблемы, но если при этом уже выстраивается очередь процессов, ожидающих CPU-время (Saturation растёт) — это ранний сигнал приближающейся деградации ещё до того, как Utilization формально достигнет 100%.

## Гистограммы — почему не среднее и не просто "текущее значение"

```
http_request_duration_seconds_bucket{le="0.1"} 8500   -- 8500 запросов уложились в ≤100мс
http_request_duration_seconds_bucket{le="0.5"} 9800   -- 9800 запросов уложились в ≤500мс
http_request_duration_seconds_bucket{le="+Inf"} 10000 -- все 10000 запросов

-- PromQL: вычислить p99 latency из накопленных bucket'ов
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))
```

Гистограмма в Prometheus — набор "корзин" (buckets), каждая считает, сколько наблюдений попало **не выше** заданной границы (`le` — less than or equal) — из этого распределения `histogram_quantile()` вычисляет перцентиль на лету в момент запроса к системе мониторинга, а не требует заранее решить, какой именно перцентиль понадобится (в отличие от прямого хранения "текущий p99", которое зафиксировало бы только один конкретный перцентиль и потеряло бы возможность посчитать, например, p999 задним числом). Ровно тот же аргумент против среднего, что разобран для нагрузочного тестирования (см. [08-performance-reliability/10-load-testing-percentiles.md](../08-performance-reliability/10-load-testing-percentiles.md)) — среднее latency скрывает хвост, гистограмма его показывает явно, причём непрерывно в постоянном мониторинге, а не только в разовом тесте.

## Что снимать с Node-сервиса в первую очередь

```
RED: rate/errors/duration по HTTP-эндпоинтам (route-level, не только агрегат)
+ event loop lag (см. 08-performance-reliability/06-cpu-bound-event-loop-lag.md) — специфично для Node
+ heap используемая/доступная (см. 02-nodejs-core/07-memory-gc.md)
+ насыщенность пула соединений к БД (активные/idle/waiting, см. 04-databases/08-connection-pooling.md)
+ GC-паузы (частота, длительность) — коррелируют со всплесками latency
```

Помимо стандартного RED для HTTP-слоя, Node-специфичные метрики — обязательная часть минимального набора именно из-за однопоточной модели исполнения (см. [02-nodejs-core/01-architecture.md](../02-nodejs-core/01-architecture.md)): event loop lag — самый прямой индикатор, что что-то блокирует единственный поток, а heap/GC-метрики объясняют случаи, когда latency скачет синхронно с паузами сборщика мусора, не связанными напрямую с логикой конкретного запроса.

## Что спрашивают на собеседовании

1. **Какие метрики снимаете с Node-сервиса в первую очередь?** — RED по HTTP-эндпоинтам (rate/errors/duration с разбивкой по route), плюс Node-специфичные: event loop lag, heap usage, насыщенность пула соединений к БД, частота и длительность GC-пауз.
2. **Чем RED отличается от USE и когда что применять?** — RED для request-driven компонентов (API-сервис: запросы, ошибки, длительность обработки), USE для ресурсов (CPU/память/диск/пул соединений: занятость, перегруженность сверх ёмкости, ошибки самого ресурса).
3. **Почему для latency используют гистограммы, а не хранят просто среднее или текущий p99?** — гистограмма хранит распределение по корзинам, позволяя вычислить любой перцентиль на лету в момент запроса, а не только заранее зафиксированный один — тот же аргумент, что среднее скрывает хвост распределения, только применённый к постоянному мониторингу.
4. **Как работает pull-модель Prometheus и чем она отличается от push?** — Prometheus сам периодически забирает метрики с эндпоинта `/metrics` каждого сервиса по расписанию; сервису не нужно знать адрес системы мониторинга и заботиться о доставке — Prometheus сам обнаруживает недоступность сервиса при неудачном scrape.
5. **Чем Utilization отличается от Saturation в USE-методе?** — Utilization — доля занятости ресурса прямо сейчас, Saturation — есть ли уже очередь ожидания сверх номинальной ёмкости; ресурс может быть не полностью утилизирован, но уже насыщен (растущая очередь) — ранний сигнал деградации до формального достижения 100% занятости.

## Ссылки

- [Prometheus — официальная документация, Data model и Querying](https://prometheus.io/docs/concepts/data_model/)
- [prom-client (npm) — клиентская библиотека для Node](https://github.com/siimon/prom-client)
- [Google SRE Book — The Four Golden Signals (основа RED)](https://sre.google/sre-book/monitoring-distributed-systems/)
- [Brendan Gregg — The USE Method](https://www.brendangregg.com/usemethod.html)
- [Нагрузочное тестирование и перцентили](../08-performance-reliability/10-load-testing-percentiles.md)
