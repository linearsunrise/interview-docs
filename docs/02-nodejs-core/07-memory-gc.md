---
sidebar_position: 7
title: Память и GC в V8
---

# Память и GC в V8

> **TL;DR:** Heap V8 поколенческий: new space (молодые объекты, быстрый Scavenger) и old space (выжившие, Mark-Sweep-Compact). Гипотеза: большинство объектов умирают молодыми. Лимит old space настраивается `--max-old-space-size` — контейнер с 512 МБ и дефолтным лимитом получит OOMKill раньше, чем V8 начнёт агрессивно чистить.

## Структура heap

```
new space (semi-spaces from/to, единицы–десятки МБ)  ← аллокации попадают сюда
old space   ← объекты, пережившие 2 minor GC ("promotion")
large object space ← объекты > ~256 КБ сразу сюда
code space, map space ← скомпилированный код, hidden classes
+ вне heap: external (Buffer, ArrayBuffer) — V8 их не двигает
```

## Два сборщика

- **Minor GC (Scavenger):** копирует живые объекты из from-space в to-space, мёртвые просто игнорирует. Очень быстрый, стоимость пропорциональна **живым** объектам. Поэтому короткоживущий мусор в Node почти бесплатен.
- **Major GC (Mark-Sweep-Compact):** маркирует достижимое от корней, выметает, дефрагментирует. В современном V8 (Orinoco) — параллельный, инкрементальный и частично конкурентный, но stop-the-world паузы всё равно есть — на больших heap это видимые задержки p99.

## Настройка и метрики

```bash
node --max-old-space-size=1536 app.js  # МБ; ставить ~75-80% от лимита контейнера
```

```js
process.memoryUsage();
// rss        — весь процесс (правда для контейнера)
// heapTotal  — сколько V8 зарезервировал
// heapUsed   — сколько реально занято объектами
// external   — Buffer/ArrayBuffer вне heap
```

Важно: V8 определяет дефолтный лимит по памяти машины, но **не знает про cgroup-лимиты контейнера** — в k8s задавать `--max-old-space-size` явно. RSS может сильно превышать heapUsed (буферы, стеки, сам V8) — алертить надо на RSS.

Наблюдение за GC: `--trace-gc`, [`v8.getHeapStatistics()`](https://nodejs.org/api/v8.html#v8getheapstatistics), perf_hooks `PerformanceObserver({ entryTypes: ['gc'] })`.

## Что спрашивают на собеседовании

1. **Почему GC поколенческий?** — гипотеза поколений: большинство объектов умирают молодыми; дешёвый частый Scavenger для молодых + редкий дорогой Major для старых.
2. **Что значит «объект перешёл в old space»?** — пережил ~2 minor GC; долгоживущие объекты чистить дорого, поэтому долгоживущий «мусор» (кэши без лимита) — худший случай.
3. **Приложение в k8s убивается OOMKilled, хотя heapUsed маленький** — смотреть RSS: external-память (буферы), либо лимит V8 больше лимита контейнера, либо фрагментация.
4. **Как GC влияет на латентность?** — паузы major GC на больших heap; лечение: меньше heap, меньше долгоживущих аллокаций, разнести на несколько процессов.
5. **Утечка vs высокое потребление?** — утечка монотонно растёт между major GC; высокое, но стабильное плато — это рабочий набор.

## Ссылки

- [Trash talk: the Orinoco garbage collector — блог V8](https://v8.dev/blog/trash-talk) — лучший официальный текст про поколения и параллельность
- [Memory management — доки Node](https://nodejs.org/en/learn/diagnostics/memory)
- [`process.memoryUsage()`](https://nodejs.org/api/process.html#processmemoryusage), [`v8.getHeapStatistics()`](https://nodejs.org/api/v8.html#v8getheapstatistics)
- [CLI: `--max-old-space-size`](https://nodejs.org/api/cli.html#--max-old-space-sizesize-in-mib)
