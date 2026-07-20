---
sidebar_position: 7
title: Безопасность
---

# Безопасность

## Подтемы

1. [JWT: структура, HS256 vs RS256, отзыв](./01-jwt.md) — почему не зашифрован, `alg: none`, где хранить на клиенте
2. [Access + refresh: ротация, reuse detection](./02-access-refresh-tokens.md) — одноразовый refresh, инвалидация всей семьи при краже
3. [Сессии vs JWT](./03-sessions-vs-jwt.md) — trade-offs, гибридный подход в проде
4. [OAuth2 flows: authorization code + PKCE, client credentials; OIDC](./04-oauth2-oidc.md) — зачем PKCE, id_token vs access_token
5. [RBAC vs ABAC, реализация в Nest](./05-rbac-abac.md) — guards + метаданные, CASL, почему роль не защищает от IDOR
6. [OWASP Top 10 применительно к Node](./06-owasp-top10-node.md) — карта категорий с конкретными примерами
7. [SQL/NoSQL инъекции](./07-injections-sql-nosql.md) — параметризация, `$ne`/операторы MongoDB из тела запроса
8. [XSS в контексте API, CSP](./08-xss-csp.md) — reflected vs stored, роль backend в защите
9. [CSRF: когда актуален и когда нет](./09-csrf.md) — cookie vs Bearer, SameSite, CSRF-токен
10. [CORS: что реально делает](./10-cors.md) — preflight, credentials, почему это не защита сервера
11. [SSRF: сценарии, защита](./11-ssrf.md) — метаданные облака, DNS rebinding, редиректы
12. [Rate limiting: алгоритмы, Redis](./12-rate-limiting.md) — граничная проблема fixed window, token bucket, атомарность
13. [Пароли: bcrypt/argon2, cost factor](./13-password-hashing.md) — почему SHA-256 с солью недостаточно, memory-hard
14. [Секреты: env vs vault, helmet, валидация](./14-secrets-management.md) — ротация без передеплоя, whitelist в ValidationPipe
15. [Prototype pollution, ReDoS](./15-prototype-pollution-redos.md) — специфика JS/Node, `__proto__`, catastrophic backtracking

## Чеклист знаний

- [ ] JWT: структура, подпись (HS256 vs RS256), где хранить на клиенте, почему нельзя «отозвать»
- [ ] Access + refresh: ротация, reuse detection, хранение refresh-токенов
- [ ] Сессии vs JWT — trade-offs (revocation, состояние, масштабирование)
- [ ] OAuth2 flows: authorization code + PKCE, client credentials; OIDC поверх OAuth2
- [ ] RBAC vs ABAC, реализация в Nest (guards + метаданные, CASL)
- [ ] OWASP Top 10 применительно к Node: инъекции, broken access control, SSRF
- [ ] SQL-инъекции и почему параметризация спасает; NoSQL-инъекции ($where, операторы в query)
- [ ] XSS в контексте API (отражение пользовательского контента), CSP
- [ ] CSRF: когда актуален (cookie-сессии) и когда нет (Bearer)
- [ ] CORS: что реально делает, preflight, credentials — и что это не защита сервера
- [ ] SSRF: сценарии (URL от пользователя, метаданные облака), защита
- [ ] Rate limiting: алгоритмы (token bucket, sliding window), распределённый на Redis
- [ ] Пароли: bcrypt/argon2, cost factor, соль; таймсейф сравнение
- [ ] Секреты: env vs vault, ротация; helmet, валидация всех входных данных
- [ ] Prototype pollution, ReDoS — специфика Node

## Вопросы с собеседований

1. Access token скомпрометирован — что делать? Почему короткий TTL + refresh rotation? Как ловить reuse refresh-токена?
2. JWT в localStorage vs httpOnly cookie — trade-offs (XSS vs CSRF)?
3. Когда сессии в Redis лучше JWT? (мгновенный logout/бан, один клиент)
4. Расскажите authorization code flow with PKCE — зачем PKCE?
5. Как реализовать permissions уровня «редактировать можно только свои посты»? (RBAC недостаточно — нужен resource-based check)
6. Пользователь передаёт URL, сервер его фетчит (превью ссылок) — какие атаки и защита? (SSRF: блок приватных диапазонов, allowlist, отдельный egress)
7. Как устроен распределённый rate limiter на Redis? Почему naive INCR имеет граничную проблему?
8. Чем argon2 лучше bcrypt? Что такое cost factor и как его выбирать?
9. ReDoS — пример уязвимой регулярки, как защищаться?
10. Найдите уязвимости в сниппете кода (типовое задание: конкатенация в SQL, отсутствие проверки владельца ресурса, `eval`, слабый JWT-секрет).

## Ключевые тезисы

**Refresh rotation + reuse detection:** каждый refresh одноразовый; при использовании выдаётся новая пара, старый инвалидируется. Если пришёл уже использованный refresh — это признак кражи → инвалидировать всё семейство токенов.

**CSRF:** атака работает потому, что браузер автоматически шлёт куки. Bearer-токен из памяти браузер сам не приложит → CSRF неактуален, но актуален XSS. httpOnly cookie — наоборот. Отсюда trade-off вопроса 2.

**SSRF-защита:** валидация схемы/хоста, резолв DNS и проверка IP на приватные диапазоны (с учётом DNS rebinding — проверять IP, по которому реально коннектимся), запрет редиректов или их перепроверка, egress-прокси.

## Красные флаги

- «JWT зашифрован» (он подписан, payload читается любым)
- «CORS защищает моё API от атак»
- Хранит пароли в SHA-256 «с солью, значит норм»
- Не проверяет владельца ресурса (IDOR) — только роль

## Практика

- Реализовать в Nest полный auth-флоу: access + refresh с ротацией и reuse detection на Redis
- Написать rate-limit guard на Redis (sliding window)
- Пройти пару лаб (например, PortSwigger) по SSRF и IDOR
