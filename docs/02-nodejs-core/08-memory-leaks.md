---
sidebar_position: 8
title: Поиск утечек памяти
---

# Поиск утечек памяти

> **TL;DR:** Методика: подтвердить утечку графиком RSS/heapUsed → снять **два-три heap snapshot** с нагрузкой между ними → в Chrome DevTools режим Comparison → смотреть, каких объектов стало больше и **кто их удерживает** (retainers). На собесе ждут именно рассказ процесса + типовые причины.

## Типовые причины (знать списком)

1. **Глобальные кэши/массивы без лимита** — `cache[key] = value` и никогда не чистим. Лечение: LRU (lru-cache), TTL, WeakMap.
2. **Накопление слушателей** — `emitter.on()` в обработчике запроса без `off()`; симптом — warning `MaxListenersExceededWarning`.
3. **Замыкания, удерживающие большие объекты** — колбэк в долгоживущей структуре держит весь контекст.
4. **Таймеры** — `setInterval` без `clearInterval` держит и себя, и замыкание.
5. **Незакрытые ресурсы** — сокеты, стримы после ошибки (см. `pipe` vs `pipeline`), курсоры БД.
6. **`subarray`/slice от больших буферов** — маленький view держит весь родительский буфер.

## Инструменты

```js
// снять снапшот прямо из прода (осторожно: STW на время снятия, файл размером с heap)
require('v8').writeHeapSnapshot('/tmp/heap.heapsnapshot');
```

- **Chrome DevTools** (`node --inspect`, chrome://inspect): Memory → Heap snapshot → Comparison между двумя снимками; колонки Delta и Retained Size; у объекта смотреть Retainers — цепочку до GC root.
- **`process.memoryUsage()`** в метрики Prometheus — базовая телеметрия, с которой всё начинается.
- **clinic.js** (`clinic doctor`, `clinic heapprofiler`) — быстрая диагностика: покажет паттерн «пила растёт» и горячие места аллокаций.
- **`--heapsnapshot-near-heap-limit=3`** — Node сам снимет снапшоты перед падением по OOM: бесценно для «падает раз в неделю в проде».

## Как рассказывать «реальный кейс» (структура ответа)

1. Симптом: рестарты по OOM раз в N часов, пила на графике RSS.
2. Локализация: сняли снапшоты через `writeHeapSnapshot` с интервалом под нагрузкой.
3. Диагноз: в Comparison росли, например, closures/строки; retainers привели к подписке на event emitter в каждом запросе.
4. Фикс + защита: убрали подписку/добавили LRU; алерт на heapUsed тренд; нагрузочный тест в CI.

## Что спрашивают на собеседовании

1. **Как найти утечку в проде?** — методика выше; ключевые слова: comparison, retained size, retainers, GC root.
2. **`heapUsed` стабилен, RSS растёт — что это?** — external (Buffer/ArrayBuffer), нативные аддоны или фрагментация — heap-снапшот этого не покажет.
3. **Чем опасен снапшот на проде?** — stop-the-world + удвоение памяти на время снятия; делать на одном инстансе, выведенном из балансировки.
4. **WeakMap/WeakRef — как помогают?** — ключи WeakMap не удерживают объект: кэш метаданных к объекту умирает вместе с объектом.
5. **Почему «утечка» может оказаться нормой?** — lazy-инициализация, JIT-код, рост до плато; утечка — только монотонный рост после major GC.

## Ссылки

- [Memory diagnostics — официальный гайд Node](https://nodejs.org/en/learn/diagnostics/memory/using-heap-snapshot) (+ [heap profiler](https://nodejs.org/en/learn/diagnostics/memory/using-heap-profiler))
- [`v8.writeHeapSnapshot`](https://nodejs.org/api/v8.html#v8writeheapsnapshotfilenameoptions), [`--heapsnapshot-near-heap-limit`](https://nodejs.org/api/cli.html#--heapsnapshot-near-heap-limitmax_count)
- [clinic.js](https://clinicjs.org/) — doctor / flame / heapprofiler
- [lru-cache](https://github.com/isaacs/node-lru-cache) — правильный кэш вместо объекта-словаря
