# Архитектурные решения (ADR)

Формат: контекст → решение → альтернативы → последствия. Статус «предложено» становится «принято» после утверждения Фазы 3 автором. Сводный журнал решений проекта — [`../DECISIONS.md`](../DECISIONS.md).

| ADR | Решение | D-xxx |
|---|---|---|
| [ADR-0001](0001-modular-monolith.md) | Модульный монолит | D-031 |
| [ADR-0002](0002-backend-stack-versions.md) | Версии backend: Java 21, Spring Boot 4.2, PostgreSQL 16 | D-027, D-028 |
| [ADR-0003](0003-costing-and-fefo-separation.md) | Средневзвешенная себестоимость отдельно от физического FEFO | D-001, D-002, D-018 |
| [ADR-0004](0004-no-redis-kafka-kubernetes.md) | Без Redis, Kafka и Kubernetes | D-029 |
| [ADR-0005](0005-authentication-jwt-refresh.md) | Аутентификация: access JWT + refresh-токен в httpOnly-cookie | D-030 |
| [ADR-0006](0006-pessimistic-locking.md) | Пессимистические блокировки остатков в фиксированном порядке | D-033 |
| [ADR-0007](0007-ids-and-tenant-isolation.md) | Идентификаторы bigint и изоляция магазинов составными FK | D-032 |
| [ADR-0008](0008-idempotency-in-transaction.md) | Ключ идемпотентности в транзакции документа | D-034 |
| [ADR-0009](0009-error-format.md) | Формат ошибок: RFC 9457 с кодами | D-041 |
| [ADR-0010](0010-russian-only-i18n.md) | Только русский язык, i18next сохраняется | D-026 |
| [ADR-0011](0011-frontend-stack.md) | Frontend: TypeScript 6, React Router SPA, big.js, типы из OpenAPI | D-027 |
| [ADR-0012](0012-single-vps-compose.md) | Развёртывание: Docker Compose на одном VPS, ручной deploy | D-040 |
