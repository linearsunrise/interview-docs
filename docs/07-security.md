---
sidebar_position: 7
title: Безопасность
---

# Безопасность

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
