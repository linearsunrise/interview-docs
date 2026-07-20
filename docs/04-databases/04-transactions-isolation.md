---
sidebar_position: 4
title: "Транзакции: ACID и уровни изоляции"
---

# Транзакции: ACID и уровни изоляции

> **TL;DR:** ACID — Atomicity/Consistency/Isolation/Durability, но на собеседовании реально проверяют понимание **Isolation**: как раз тут прячутся все трейд-оффы. Уровни изоляции — это не "больше/меньше защиты" абстрактно, а конкретный список аномалий, от которых каждый уровень защищает. Дефолт в PostgreSQL — **READ COMMITTED**, и это допускает non-repeatable read и phantom read (в отличие от многих учебников — в Postgres READ COMMITTED и REPEATABLE READ не эквивалентны стандарту SQL один в один, см. ниже).

## ACID коротко

- **Atomicity** — транзакция выполняется целиком или не выполняется вовсе; сбой на середине откатывает всё.
- **Consistency** — транзакция переводит базу из одного валидного состояния (по ограничениям/constraints) в другое; это следствие Atomicity + Isolation + бизнес-ограничений схемы, а не отдельный независимый механизм.
- **Isolation** — конкурентные транзакции не видят "промежуточных" эффектов друг друга сильнее, чем разрешает выбранный уровень.
- **Durability** — после `COMMIT` данные переживут падение процесса/сервера (WAL — write-ahead log, сброшенный на диск до подтверждения коммита клиенту).

## Аномалии — то, от чего защищают уровни изоляции

| Аномалия | Суть |
|---|---|
| **Dirty read** | читаем данные другой транзакции, которая ещё не закоммичена (и может откатиться) |
| **Non-repeatable read** | дважды читаем одну строку в рамках своей транзакции — второй раз видим другое значение (кто-то успел закоммитить `UPDATE` между чтениями) |
| **Phantom read** | дважды выполняем один и тот же диапазонный запрос — второй раз появились/исчезли строки (кто-то закоммитил `INSERT`/`DELETE`, попадающий под условие) |
| **Lost update** | две транзакции читают одно значение, обе независимо изменяют и коммитят — одно из изменений теряется, будто его и не было |

## Таблица уровней изоляции (стандарт SQL)

| Уровень | Dirty read | Non-repeatable read | Phantom read | Lost update |
|---|---|---|---|---|
| READ UNCOMMITTED | возможен | возможен | возможен | возможен |
| READ COMMITTED | нет | возможен | возможен | возможен |
| REPEATABLE READ | нет | нет | возможен (по стандарту) | возможен |
| SERIALIZABLE | нет | нет | нет | нет |

## PostgreSQL — не совсем по стандарту, и это важно знать

PostgreSQL не реализует READ UNCOMMITTED вообще (ведёт себя как READ COMMITTED, если его запросить) — грязных чтений там нет никогда, в отличие от, например, старого MySQL/InnoDB, где READ UNCOMMITTED — реальный отдельный уровень. Ещё важнее: PostgreSQL реализует **REPEATABLE READ** через snapshot isolation (весь снимок данных фиксируется на старте транзакции), из-за чего в Postgres на REPEATABLE READ **phantom read физически невозможен** — уровень строже, чем требует стандарт. При этом это не полный SERIALIZABLE: сериализационные аномалии (write skew — конкурентные транзакции читают пересекающиеся данные и пишут непересекающиеся, но итоговое состояние невозможно получить ни при каком последовательном порядке выполнения) на REPEATABLE READ всё ещё возможны, для них нужен именно SERIALIZABLE (SSI — serializable snapshot isolation, с реальными откатами по serialization failure).

```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
-- весь снимок базы зафиксирован здесь, дальнейшие SELECT в транзакции его не "обновляют"
SELECT balance FROM accounts WHERE id = 1;
-- ... другая транзакция коммитит UPDATE ...
SELECT balance FROM accounts WHERE id = 1; -- то же самое значение, что и в первый раз
COMMIT;
```

## Lost update — конкретный пример и три способа защиты

```sql
-- Транзакция A и B параллельно:
SELECT balance FROM accounts WHERE id = 1; -- обе читают 100
-- A: UPDATE accounts SET balance = 100 - 30 WHERE id = 1; COMMIT; -- 70
-- B: UPDATE accounts SET balance = 100 - 20 WHERE id = 1; COMMIT; -- 80, "съело" списание A
```

На READ COMMITTED (дефолт в Postgres) это реально происходит, потому что каждый `UPDATE` работает со своим свежепрочитанным значением, а не с тем, что видела транзакция изначально. Три рабочих решения — подробнее в [Блокировки: row-level, FOR UPDATE, advisory, optimistic vs pessimistic](./07-locking.md):
1. `SELECT ... FOR UPDATE` — пессимистичная блокировка строки на время транзакции.
2. Optimistic locking через колонку `version` — `UPDATE ... SET balance = ?, version = version + 1 WHERE id = ? AND version = ?`, 0 затронутых строк = конфликт, повторить.
3. Атомарный `UPDATE accounts SET balance = balance - 30 WHERE id = 1` — база сама читает-и-пишет в одной операции под row-lock, гонки в принципе нет.

## Что спрашивают на собеседовании

1. **Расшифруйте ACID.** — не просто расшифровка: суметь объяснить, что Consistency — не отдельный независимый механизм, а следствие остальных трёх + ограничений схемы.
2. **Какой уровень изоляции по умолчанию в PostgreSQL и от каких аномалий он защищает?** — READ COMMITTED: защищает от dirty read, допускает non-repeatable read, phantom read, lost update.
3. **Чем REPEATABLE READ в PostgreSQL отличается от стандарта SQL?** — благодаря snapshot isolation в Postgres на этом уровне phantom read невозможен физически, хотя стандарт формально его допускает на REPEATABLE READ.
4. **Что такое lost update и как её предотвратить?** — конкурентная перезапись без учёта чужого параллельного изменения; решения — `FOR UPDATE`, optimistic locking с версией, атомарный `UPDATE` с выражением.
5. **Почему "просто обернуть в транзакцию" не защищает от гонок само по себе?** — транзакция гарантирует атомарность и изоляцию по выбранному уровню, но на READ COMMITTED (дефолт) конкурентные транзакции всё ещё видят чужие коммиты между своими операциями — нужен явный выбор уровня/блокировки под конкретный сценарий.

## Ссылки

- [PostgreSQL — Transaction Isolation](https://www.postgresql.org/docs/current/transaction-iso.html) — с явным перечислением, чем Postgres отличается от стандарта
- [PostgreSQL — Serializable Isolation vs True Serializability](https://www.postgresql.org/docs/current/transaction-iso.html#XACT-SERIALIZABLE) — write skew и SSI
- [Jepsen — A Critique of the CAP Theorem / Consistency Models](https://jepsen.io/consistency) — таксономия аномалий и уровней в целом, не только PostgreSQL
- [Use The Index, Luke — не по теме напрямую, но полезно держать рядом с транзакциями](https://use-the-index-luke.com/)
