# CLAUDE.md — правила проекта Grocery ERP

Дипломный проект: веб-ERP для малых продуктовых магазинов Кыргызстана с партионным контролем сроков годности (FEFO) и объяснимым пополнением запасов. Работает один автор.

## Порядок работы

- Работа идёт **по фазам**; следующая фаза начинается только после явного утверждения автором. Фаза 1 — исследование, Фаза 2 — требования, Фаза 3 — архитектура, Фаза 4 — реализация.
- **Никогда не делать `git push` и не открывать PR без явного разрешения автора.** Коммитить локально и спрашивать.
- Не менять утверждённые документы и академические требования молча: расхождение → новое решение `D-xxx` в `docs/DECISIONS.md`.
- Значимое архитектурное решение → ADR в `docs/adr/` и строка в `docs/DECISIONS.md`.

## Язык

- Продукт **только на русском** (D-026). Кыргызской локализации нет.
- Документация и объяснения автору — на русском. Идентификаторы требований и код — на английском (`FR-001`, `BR-030`, `D-026`, имена классов, ключи i18n).
- Все строки интерфейса — в `frontend/src/locales/ru/*.json`, не в коде.

## Где что описано

| Тема | Документ |
|---|---|
| Требования, правила, критерии приёмки | `docs/REQUIREMENTS.md`, `docs/BUSINESS_RULES.md`, `docs/ACCEPTANCE_CRITERIA.md` |
| Права ролей | `docs/ROLE_PERMISSIONS.md` |
| Архитектура и модули | `docs/ARCHITECTURE.md` |
| Схема БД | `docs/DATABASE.md`, `docs/diagrams/er.md` |
| Алгоритмы (себестоимость, FEFO, блокировки, идемпотентность, риск, пополнение) | `docs/ALGORITHMS.md` |
| API | `docs/API.md` |
| Безопасность | `docs/SECURITY.md` |
| Развёртывание | `docs/DEPLOYMENT_ARCHITECTURE.md` |
| Решения | `docs/DECISIONS.md`, `docs/adr/` |

## Стек

Backend: Java 21, Spring Boot 4.2.x, Spring Security (JWT resource server), Spring Data JPA / Hibernate 7, PostgreSQL 16, Flyway, Bean Validation, springdoc-openapi 3.x, Maven Wrapper. Boot 4 использует **Jackson 3** (`tools.jackson.*`) и **JUnit 6**.
Frontend: React 19, TypeScript 6.0, Vite 8, Tailwind CSS 4, shadcn/ui, React Router 8 (SPA), TanStack Query 5, React Hook Form + Zod 4, i18next (только `ru`), Recharts, big.js, openapi-typescript.
Тесты: JUnit, Mockito, Testcontainers, ArchUnit, Vitest, Testing Library, MSW, Playwright.
Инфраструктура: Docker Compose, Nginx, GitHub Actions.

Не добавлять Redis, Kafka, Kubernetes, микросервисы без решения в `DECISIONS.md` (ADR-0004). Новые зависимости — только стабильные выпуски старше 14 дней.

## Неприкосновенные правила кода

1. **Деньги и количества — никогда `double`/`float`.** `BigDecimal`/`numeric`; деньги 2 знака, себестоимость единицы 4 знака, количества 3 знака; округление `HALF_UP`. В JSON — строки. На клиенте — `big.js`.
2. **Себестоимость — средневзвешенная** (D-001). Цена партии в себестоимость продаж не входит. Партионная себестоимость — только после решения научного руководителя (D-002).
3. **FEFO и оценка разделены**: `FefoAllocator` решает, какие партии уходят; `WeightedAverageCost` — по какой себестоимости.
4. **`storeId` берётся только из `CurrentUser`.** Каждый метод репозитория бизнес-таблиц принимает `storeId`. Чужой объект → 404.
5. **Остатки меняются только движениями** в одной транзакции с документом, аудитом и ключом идемпотентности. Блокировки: `product_stock` по возрастанию `product_id`, затем `batch` по возрастанию `id`, `lock_timeout` 5 с.
6. **Проведённые документы неизменяемы**; исправление — сторно или новым документом.
7. **STAFF не получает чувствительные поля** — отдельные DTO для OWNER и STAFF, а не скрытие на клиенте.
8. Каждый маршрут API объявляет `@RequiresRole`/`@RequiresPermission`; без аннотации маршрут не должен существовать.
9. «Сегодня» — только через `BusinessClock` (Asia/Bishkek); хранение времени — UTC.
10. Схема БД меняется только новой миграцией Flyway; применённые миграции не редактируются.

## Модули backend

Пакет `kg.groceryerp.<module>` с подпакетами `api`, `web`, `application`, `domain`, `persistence`. Другие модули используют только `api`. Граф зависимостей ацикличен (ArchUnit). Порядок: shared → stores → users → auth → catalog → suppliers → inventory → purchases → sales → returns → expenses → finance → expiration → replenishment → reporting.

## Тесты

- Примеры из `docs/ALGORITHMS.md` и критерии `AC-xxx` — юнит-тесты с этими идентификаторами в названии.
- Интеграционные тесты — на настоящем PostgreSQL 16 через Testcontainers, не H2.
- Обязательные: конкурентная продажа последней единицы (NFR-024), идемпотентность (NFR-023), матрица прав по всем маршрутам (NFR-032), изоляция магазинов (NFR-033).
- После падения тестов перезапускать только упавшие, пока идёт исправление.

## Команды (появятся в Фазе 4)

```
cd backend && ./mvnw verify          # все тесты backend
cd frontend && npm run typecheck && npm run lint && npm test
docker compose -f docker-compose.yml -f docker-compose.dev.yml up db
```
