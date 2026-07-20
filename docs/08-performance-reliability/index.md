---
sidebar_position: 8
title: Производительность и надёжность
---

# Производительность и надёжность

## Подтемы

1. [Кеширование: cache-aside, write-through, write-behind](./01-caching-patterns.md) — TTL vs явная инвалидация
2. [Проблемы кеша: stampede, penetration, hot keys](./02-cache-problems.md) — распределённый лок, negative caching, jitter
3. [Многоуровневый кеш: in-memory (LRU) + Redis](./03-multi-level-caching.md) — pub/sub-инвалидация, версионирование ключей
4. [Масштабирование и балансировка нагрузки](./04-scaling-load-balancing.md) — stateless, round robin vs least connections, health checks
5. [Профилирование: CPU flame graph, clinic.js](./05-profiling-cpu-memory.md) — `--prof`, `--cpu-prof`, как читать flame graph
6. [CPU-bound в Node, event loop lag, worker threads](./06-cpu-bound-event-loop-lag.md) — `monitorEventLoopDelay`, оффлоад тяжёлых вычислений
7. [Retry budget и таймауты повсюду](./07-retry-budget-timeouts.md) — клиент/сервер/пул БД, timeout budget по цепочке
8. [Bulkhead, load shedding, graceful degradation](./08-bulkhead-load-shedding-degradation.md) — изоляция ресурсов, когда отдавать 503
9. [Очереди для сглаживания нагрузки: BullMQ](./09-queues-bullmq.md) — retries, DLQ, конкурентность, batch-постановка
10. [Нагрузочное тестирование и перцентили](./10-load-testing-percentiles.md) — autocannon/k6, coordinated omission

## Чеклист знаний

- [ ] Кеширование: cache-aside, write-through, write-behind; TTL vs явная инвалидация
- [ ] Проблемы кеша: stampede (dogpile), cache penetration, hot keys; решения (лок, jitter TTL, negative caching)
- [ ] Многоуровневый кеш: in-memory (LRU) + Redis — консистентность между инстансами
- [ ] Масштабирование: stateless-приложения, горизонтальное vs вертикальное, load balancing (round robin, least connections)
- [ ] Профилирование: CPU-профиль (flame graph, clinic.js, `--prof`), memory
- [ ] CPU-bound в Node: выявление (event loop lag), оффлоад в worker threads / отдельный сервис
- [ ] Event loop lag как метрика здоровья
- [ ] Retry: exponential backoff + jitter, retry budget, только идемпотентные операции
- [ ] Timeouts повсюду: клиентские, серверные, на пул БД; timeout budget сквозь цепочку вызовов
- [ ] Circuit breaker, bulkhead, load shedding, graceful degradation
- [ ] Очереди для сглаживания нагрузки (BullMQ): retries, backoff, DLQ, конкурентность
- [ ] Нагрузочное тестирование: autocannon/k6, что мерить (p50/p95/p99, RPS, error rate)
- [ ] Почему среднее время ответа — плохая метрика (важны перцентили)

## Вопросы с собеседований

1. p99 latency вырос, p50 в норме — какие гипотезы и как копать? (GC-паузы, пул соединений, медленные запросы у части пользователей, hot key)
2. Кеш популярной записи истёк, 1000 запросов одновременно пошли в БД — что это и как чинить? (stampede: mutex/single flight, probabilistic early refresh)
3. Как инвалидировать кеш при обновлении сущности на 5 инстансах с локальным LRU? (pub/sub-инвалидация, версионирование ключей)
4. Эндпоинт делает ресайз изображений и тормозит всё приложение — диагностика и решение.
5. Как измерить event loop lag и на какие цифры алертить?
6. Ретраи уронили нижестоящий сервис — что пошло не так и как правильно? (storm: backoff+jitter, budget, circuit breaker)
7. Спроектируйте фоновую обработку: отправка 100k писем — очередь, конкурентность, retries, DLQ, идемпотентность.
8. Что такое load shedding и когда лучше отдать 503, чем поставить запрос в очередь?
9. Как вы проводите нагрузочное тестирование? Что такое coordinated omission (базово)?
10. Расскажите реальный кейс оптимизации из вашей практики — что мерили, что нашли, что дало эффект.

## Ключевые тезисы

**Cache stampede:** при истечении горячего ключа множество воркеров одновременно идут в БД. Решения: распределённый лок «пересчитывает один, остальные ждут/отдают stale», разброс TTL (jitter), фоновое обновление до истечения.

**Retry правильно:** только идемпотентные операции, exponential backoff + jitter, ограниченное число попыток, retry budget (не более X% трафика — ретраи), в связке с circuit breaker и таймаутами. Таймауты по цепочке должны уменьшаться (budget), иначе внешний уже отвалился, а внутренние продолжают работу.

**Перцентили:** среднее скрывает хвост; p99 при 100 запросах на страницу почти гарантированно затрагивает каждого пользователя.

## Красные флаги

- «Добавим кеш» без ответа про инвалидацию
- Ретраи без backoff «просто 3 раза подряд»
- Оптимизация без измерений («мне кажется, тут медленно»)

## Практика

- Реализовать cache-aside с защитой от stampede (Redis lock) в Nest-интерсепторе или сервисе
- Снять flame graph с искусственно CPU-тяжёлого эндпоинта (clinic flame / 0x), перенести работу в worker thread, сравнить autocannon до/после
- Собрать пайплайн на BullMQ с retries, backoff и DLQ
