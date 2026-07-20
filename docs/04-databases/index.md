---
sidebar_position: 4
title: Базы данных
---

# Базы данных

## Подтемы

1. [B-tree индексы и leftmost prefix](./01-btree-indexes.md) — устройство дерева, порядок колонок в составном индексе, кластерный vs некластерный
2. [Partial, covering, функциональные индексы](./02-index-types.md) — `INCLUDE`, Index Only Scan, почему индекс молча не используется
3. [EXPLAIN (ANALYZE): чтение плана](./03-explain-analyze.md) — seq/index/bitmap scan, типы join, `BUFFERS`, алгоритм разбора медленного запроса
4. [Транзакции: ACID и уровни изоляции](./04-transactions-isolation.md) — аномалии, READ COMMITTED в PostgreSQL, lost update
5. [MVCC, VACUUM, bloat](./05-mvcc-vacuum.md) — как физически устроен UPDATE, autovacuum, transaction ID wraparound
6. [Deadlocks](./06-deadlocks.md) — пример на двух транзакциях, диагностика через `pg_locks`, предотвращение
7. [Блокировки: row-level, FOR UPDATE, advisory](./07-locking.md) — `SKIP LOCKED`, optimistic vs pessimistic locking
8. [Connection pooling](./08-connection-pooling.md) — почему не «чем больше, тем лучше», pgbouncer и режимы pool_mode
9. [ORM: N+1, транзакции, миграции](./09-orm-typeorm-prisma.md) — TypeORM/Prisma, eager vs lazy, join vs IN-батч
10. [Пагинация: offset vs cursor](./10-pagination.md) — почему offset деградирует, keyset-пагинация на составном индексе
11. [Репликация](./11-replication.md) — streaming, sync vs async, replication lag, read-your-writes
12. [Шардирование vs партиционирование](./12-sharding-partitioning.md) — выбор shard key, hot shard, resharding
13. [Redis](./13-redis.md) — структуры данных, eviction policy, RDB vs AOF, pub/sub vs streams
14. [MongoDB](./14-mongodb.md) — когда уместна, индексы, aggregation pipeline, embed vs reference

## Чеклист знаний

- [ ] B-tree индексы: как устроены, почему порядок колонок в составном индексе важен (leftmost prefix)
- [ ] Partial, covering (INCLUDE), функциональные индексы; когда индекс не используется
- [ ] EXPLAIN (ANALYZE): seq scan vs index scan vs bitmap scan, чтение плана
- [ ] Транзакции: ACID, уровни изоляции и аномалии (dirty/non-repeatable/phantom reads, lost update)
- [ ] MVCC в PostgreSQL, VACUUM, bloat
- [ ] Deadlocks: как возникают, как диагностировать, как предотвращать (порядок захвата)
- [ ] Блокировки: row-level, `SELECT FOR UPDATE`, advisory locks, optimistic vs pessimistic locking
- [ ] Connection pooling: зачем, pgbouncer, размер пула (не «чем больше, тем лучше»)
- [ ] TypeORM/Prisma: миграции, транзакции, N+1, eager vs lazy
- [ ] Пагинация: offset vs cursor (keyset) — почему offset деградирует
- [ ] Репликация: streaming, sync/async, replication lag, чтение с реплик
- [ ] Шардирование vs партиционирование: ключи, resharding, hot shards
- [ ] Redis: структуры данных, TTL, eviction policies, персистентность (RDB/AOF), pub/sub vs streams
- [ ] MongoDB: когда уместна, индексы, aggregation pipeline (базово)

## Вопросы с собеседований

1. Есть медленный запрос — ваши действия по шагам?
2. Индекс на `(a, b)` — сработает ли для `WHERE b = ?`? А для `WHERE a = ? ORDER BY b`?
3. Объясните уровни изоляции. Какой дефолт в PostgreSQL и какие аномалии он допускает?
4. Два конкурентных запроса читают баланс и списывают деньги — как избежать lost update? (3 способа: FOR UPDATE, optimistic с version, атомарный UPDATE)
5. Что такое deadlock, покажите пример и способ предотвращения.
6. Почему offset-пагинация на миллионной странице тормозит? Как устроена cursor-based?
7. N+1 в ORM: как обнаружить и починить? Чем join-стратегия отличается от отдельного запроса с IN?
8. Читаем с реплики сразу после записи в мастер — какие проблемы и решения? (read-your-writes)
9. Как выбрать shard key? Что делать с горячим шардом?
10. Redis как кеш vs Redis как хранилище — какие настройки различаются (eviction, persistence)?
11. Сколько соединений выставить в пуле приложения и почему «500» — плохой ответ?

## Ключевые тезисы

**Медленный запрос — алгоритм:** EXPLAIN ANALYZE → смотреть фактические rows vs предсказанные (устаревшая статистика?) → seq scan на большой таблице? → индекс / переписать запрос / денормализация. Не забывать: проблема может быть в блокировках, а не в плане.

**Lost update:** дефолтный READ COMMITTED его допускает. Решения: `SELECT ... FOR UPDATE` (пессимистично), version-колонка + `UPDATE ... WHERE version = ?` (оптимистично), либо атомарный `UPDATE balance = balance - x WHERE balance >= x`.

**Пул соединений:** каждое соединение PostgreSQL — процесс; сотни коннектов деградируют базу. Практическая формула — от числа ядер БД, обычно десятки, не сотни; при большом числе инстансов приложения — pgbouncer.

**Cursor-пагинация:** `WHERE (created_at, id) < ($1, $2) ORDER BY created_at DESC, id DESC LIMIT n` — стабильна и O(log n) при индексе, но нет «перейти на страницу 50».

## Красные флаги

- «Просто добавлю индекс на все колонки»
- Не знает дефолтный уровень изоляции своей БД
- «Транзакция защитит от гонок сама по себе»
- Никогда не читал EXPLAIN

## Практика

- Сгенерировать таблицу на 5–10 млн строк, поиграть с EXPLAIN: составные индексы, leftmost prefix, covering
- Воспроизвести deadlock двумя транзакциями в psql
- Реализовать в тестовом сервисе optimistic locking и cursor-пагинацию
