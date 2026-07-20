---
sidebar_position: 6
title: Deadlocks
---

# Deadlocks: как возникают, диагностика, предотвращение

> **TL;DR:** Deadlock — две (или больше) транзакции ждут блокировки, удерживаемой друг другом, по кругу, — ни одна не может продолжить. PostgreSQL сам обнаруживает цикл ожидания (перебором графа ожидания раз в `deadlock_timeout`, по умолчанию 1с) и принудительно откатывает одну из транзакций с ошибкой — приложение обязано это отловить и ретраить. Главная профилактика — **единый порядок захвата блокировок** во всех местах кода.

## Классический пример на двух транзакциях

```sql
-- Транзакция A                          -- Транзакция B
BEGIN;                                   BEGIN;
UPDATE accounts SET balance = balance - 10
  WHERE id = 1; -- держит lock на id=1
                                          UPDATE accounts SET balance = balance - 10
                                            WHERE id = 2; -- держит lock на id=2
UPDATE accounts SET balance = balance + 10
  WHERE id = 2; -- ждёт lock на id=2 (держит B)
                                          UPDATE accounts SET balance = balance + 10
                                            WHERE id = 1; -- ждёт lock на id=1 (держит A)
-- ЦИКЛ: A ждёт B, B ждёт A → deadlock
```

Обе транзакции реализуют перевод денег между счетами, но захватывают строки в разном порядке (A: 1→2, B: 2→1). PostgreSQL детектирует цикл в графе ожидания и убивает одну из транзакций:

```
ERROR:  deadlock detected
DETAIL: Process 1234 waits for ShareLock on transaction 5678; blocked by process 5678.
        Process 5678 waits for ShareLock on transaction 1234; blocked by process 1234.
HINT:   See server log for query details.
```

## Почему это не баг СУБД, а баг приложения

Каждая отдельная блокировка — корректна и необходима (без неё было бы lost update, см. [04-transactions-isolation.md](./04-transactions-isolation.md)). Deadlock — следствие **порядка** захвата, который база не контролирует, потому что это бизнес-логика приложения. СУБД лишь гарантирует, что зависшая ситуация не будет длиться вечно — обнаружит цикл и разрубит его откатом одной стороны.

## Предотвращение

1. **Единый порядок захвата.** Самое надёжное: всегда захватывать строки в одном и том же порядке (например, по возрастанию PK) во всех транзакциях, трогающих несколько строк одной таблицы:
   ```sql
   -- вместо порядка "как в запросе" — явно отсортировать
   UPDATE accounts SET balance = balance - 10 WHERE id = LEAST(1, 2);
   UPDATE accounts SET balance = balance + 10 WHERE id = GREATEST(1, 2);
   ```
2. **Короче транзакции.** Чем дольше транзакция держит блокировку, тем выше шанс пересечься с другой — не делать сетевые вызовы/тяжёлые вычисления между `BEGIN` и `COMMIT`.
3. **Меньше блокировок за раз.** По возможности — один атомарный `UPDATE` с выражением (`balance = balance - x`) вместо `SELECT` + логика в приложении + `UPDATE`.
4. **`SELECT ... FOR UPDATE ORDER BY id`** — если блокируется набор строк пачкой, сортировка перед блокировкой убирает произвольность порядка.
5. **Retry с backoff на уровне приложения.** Deadlock — ожидаемая ситуация под конкурентной нагрузкой, а не авария; код, пишущий в БД конкурентно, должен ловить `40P01` (deadlock_detected) / `40001` (serialization_failure на SERIALIZABLE) и повторять транзакцию.

## Диагностика в проде

```sql
-- кто кого блокирует прямо сейчас (не deadlock, а обычное ожидание, которое может в него перерасти)
SELECT blocked_locks.pid AS blocked_pid, blocking_locks.pid AS blocking_pid,
       blocked_activity.query AS blocked_query
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_locks blocking_locks
  ON blocking_locks.locktype = blocked_locks.locktype
 AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
 AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
 AND blocking_locks.pid != blocked_locks.pid
JOIN pg_catalog.pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
WHERE NOT blocked_locks.granted;
```

Сами deadlock-и (уже случившиеся и разрешённые) видны в логе сервера при `log_lock_waits = on` — там же появляется полный текст обоих конфликтующих запросов, что обычно и нужно для разбора причины.

## Что спрашивают на собеседовании

1. **Что такое deadlock и как его обнаруживает СУБД?** — циклическое ожидание блокировок; PostgreSQL периодически (`deadlock_timeout`) строит граф ожидания и при обнаружении цикла откатывает одну из транзакций с ошибкой.
2. **Покажите пример deadlock на двух транзакциях.** — два перевода между теми же двумя счетами в разном порядке захвата (см. код выше).
3. **Как предотвратить deadlock на уровне кода?** — единый порядок захвата блокировок (сортировка по ключу перед `UPDATE`/`SELECT FOR UPDATE`), короткие транзакции, retry на стороне приложения при получении ошибки deadlock.
4. **Это ошибка базы или приложения?** — база ведёт себя корректно (не даёт зависнуть навечно); причина — порядок операций в бизнес-логике, который база не может контролировать сама.
5. **Deadlock vs обычная долгая блокировка (lock wait) — в чём разница?** — при обычном ожидании транзакция рано или поздно дождётся освобождения; deadlock — тупик, из которого без вмешательства СУБД выхода нет в принципе, потому что стороны ждут друг друга по кругу.

## Ссылки

- [PostgreSQL — Deadlocks](https://www.postgresql.org/docs/current/explicit-locking.html#LOCKING-DEADLOCKS)
- [PostgreSQL — Explicit Locking](https://www.postgresql.org/docs/current/explicit-locking.html) — полный список видов блокировок
- [PostgreSQL wiki — Lock Monitoring](https://wiki.postgresql.org/wiki/Lock_Monitoring) — источник запроса выше
