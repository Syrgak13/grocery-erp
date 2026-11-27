# Архитектура системы

**Фаза 3 · Grocery ERP** · 27.11.2026 · статус: черновик на утверждение

Документ описывает архитектуру веб-ERP для малых продуктовых магазинов: технологический стек с проверенными версиями, модульный монолит и границы модулей, сквозные механизмы, устройство frontend и структуру репозитория. Связанные документы:

| Документ | Содержание |
|---|---|
| [`DATABASE.md`](DATABASE.md) | Таблицы, ограничения, индексы, роли БД |
| [`ALGORITHMS.md`](ALGORITHMS.md) | Себестоимость, FEFO, блокировки, идемпотентность, прибыль, риск, пополнение |
| [`API.md`](API.md) | REST API `/api/v1` |
| [`SECURITY.md`](SECURITY.md) | Аутентификация, авторизация, изоляция, STRIDE |
| [`DEPLOYMENT_ARCHITECTURE.md`](DEPLOYMENT_ARCHITECTURE.md) | Docker Compose, Nginx, CI/CD, резервные копии |
| [`diagrams/`](diagrams/) | 10 диаграмм Mermaid |
| [`adr/`](adr/) | Архитектурные решения (ADR) |
| [`DECISIONS.md`](DECISIONS.md) | Журнал решений проекта (D-xxx) |

---

## 1. Архитектурные драйверы

Что сильнее всего влияет на архитектуру (из требований Фазы 2):

| Драйвер | Требования | Архитектурный ответ |
|---|---|---|
| Ни одного отрицательного остатка и дубля продажи | BR-010–BR-013, BR-040–BR-054, NFR-020–NFR-024 | Одна реляционная БД с транзакциями, CHECK-ограничениями и блокировками строк; ключ идемпотентности в той же транзакции |
| Точные деньги | BR-070–BR-077, NFR-022 | `numeric` в БД, `BigDecimal` в Java, строки в JSON, `big.js` на клиенте |
| Изоляция магазинов | BR-120–BR-125, NFR-033 | `store_id` из токена, составные FK, тест изоляции всех маршрутов |
| STAFF не видит финансов | BR-211, SEC-002 | Отдельные DTO для OWNER и STAFF, тест схем ответов |
| Объяснимость | FR-185, FR-192, FR-208 | Чистые функции расчёта, возвращающие все промежуточные числа |
| Один разработчик, сроки диплома | CON-001 | Модульный монолит, один репозиторий, минимум инфраструктуры |
| Бюджет ≤ 20 USD/мес | CON-009 | Один сервер, Docker Compose, без управляемых облачных сервисов |
| Только русский язык | D-026 | i18next с одной локалью `ru`, без переключателя языка |

---

## 2. Технологический стек

### 2.1. Проверенные версии

Версии проверены 27.11.2026 по официальным источникам: страница системных требований Spring Boot, POM `spring-boot-dependencies` на Maven Central, реестр npm, Docker Hub. Правило фиксации версий в Фазе 4: **точные версии в `pom.xml` и `package-lock.json`, только выпуски старше 14 дней**, обновления — через Dependabot.

**Backend**

| Компонент | Версия | Источник / комментарий |
|---|---|---|
| Java | **21 LTS** (Eclipse Temurin 21.0.13) | Spring Boot 4.2 поддерживает Java 17–27. Java 21 выбрана автором; переход на Java 25 LTS возможен без изменений кода (ADR-0002) |
| Spring Boot | **4.2.x** (4.2.0 от 19.11.2026; брать последний патч ≥ 4.2.1 на старте Фазы 4) | Поддержка OSS до 31.12.2027 — покрывает диплом. Резерв: 4.1.3 (поддержка до 31.07.2027), ADR-0002 |
| Spring Framework | 7.1.0 | Управляется Boot 4.2.0 |
| Spring Security | 7.2.0 | Управляется Boot; JWT через `spring-boot-starter-oauth2-resource-server` (Nimbus JOSE) |
| Hibernate ORM | 7.4.15.Final | Управляется Boot (Jakarta Persistence 3.2) |
| Spring Data JPA | из Boot BOM | |
| PostgreSQL | **16** (образ `postgres:16.16`) | Поддержка до ноября 2028. Доступны 17, 18, 19; 16 оставлена по выбору автора |
| JDBC-драйвер PostgreSQL | 42.7.14 | Управляется Boot |
| Flyway | 12.11.0 (`flyway-core` + `flyway-database-postgresql`) | Управляется Boot. Модуль PostgreSQL обязателен отдельно |
| Bean Validation | Jakarta Validation 3.1.1 (Hibernate Validator) | `spring-boot-starter-validation` |
| OpenAPI | springdoc-openapi **3.2.x** (3.2.1 собрана на Boot 4.2.0) | Линия 3.x — для Boot 4; 3.1.x — для Boot 4.1 |
| Jackson | 3.1.8 | Boot 4 перешёл на Jackson 3 (пакеты `tools.jackson.*`) |
| Caffeine | 3.3.1 | Кэш прав пользователя ≤ 30 с и ограничение частоты |
| Bucket4j | 8.x (`bucket4j_jdk17-core`) | Ограничение частоты в памяти процесса (SEC-008) |
| Maven | 3.9.x через Maven Wrapper 3.3.4 | Boot требует Maven ≥ 3.6.3 |

**Frontend**

| Компонент | Версия (линия) | Комментарий |
|---|---|---|
| Node.js | 24 LTS (сборка) | Vite 8 требует Node ≥ 20.19 или ≥ 22.12; React Router 8 — Node ≥ 22.22 |
| React | 19.3.x | |
| TypeScript | **6.0.x** | TypeScript 7.0 уже вышел, но `typescript-eslint` 8.74 поддерживает только `< 6.1` (ADR-0011) |
| Vite | 8.3.x | 8.4.0 вышел 26.11.2026 — слишком новый по правилу 14 дней |
| Tailwind CSS | 4.3.x + `@tailwindcss/vite` | |
| shadcn/ui | CLI `shadcn` 4.x | Официально поддерживает Tailwind v4 и React 19; компоненты копируются в репозиторий |
| React Router | 8.5.x, режим SPA (data router) | Требует React ≥ 19.2.7. Framework mode (SSR) не используется |
| TanStack Query | 5.x | Серверное состояние |
| React Hook Form | 7.x + `@hookform/resolvers` 7.x | Резолвер поддерживает Zod 3.25+ и 4 |
| Zod | 4.x | |
| i18next + react-i18next | 27.x + 18.x | Только локаль `ru` (раздел 6.6) |
| Recharts | 3.x | Поддерживает React 19 |
| big.js | 7.x | **Добавлено**: десятичная арифметика денег на клиенте (FR-093, BR-072) |
| openapi-typescript + openapi-fetch | 7.x + 0.17.x | **Добавлено**: типы клиента генерируются из OpenAPI, ручных DTO нет |
| lucide-react | 1.x | Иконки (используются shadcn/ui) |

**Тестирование**

| Компонент | Версия | Назначение |
|---|---|---|
| JUnit Jupiter | 6.1.3 (Boot BOM) | Boot 4.2 управляет JUnit 6 |
| Mockito | 5.24.0 (Boot BOM) | |
| Testcontainers | 2.0.5 (Boot BOM), модуль `testcontainers-postgresql` | Интеграционные тесты на настоящем PostgreSQL 16 |
| ArchUnit | 1.5.x | **Добавлено**: проверка границ модулей и отсутствия циклов |
| Vitest | 5.0.x | |
| Testing Library | `@testing-library/react` 16.x | |
| MSW | 3.x | Подмена API в тестах компонентов |
| Playwright | 1.65.x | E2E на Chromium, Firefox, WebKit (NFR-090) |
| k6 | последняя стабильная | Нагрузочные тесты NFR-001–NFR-007, NFR-024 (вне Maven) |

**Инфраструктура**: Docker Compose v2, Nginx 1.30 (stable), GitHub Actions.

### 2.2. Рекомендованные изменения стека

| Изменение | Почему | Решение |
|---|---|---|
| Spring Boot 4.2.x вместо «любой 4.x» | Последняя ветка с самой долгой поддержкой (до конца 2027) | ADR-0002, D-027 |
| TypeScript 6.0 вместо 7.0 | Совместимость с `typescript-eslint` | ADR-0011, D-027 |
| JWT-доступ + refresh в httpOnly-cookie вместо сессии-cookie из SEC-005 | Автор выбрал JWT; refresh-токены в БД сохраняют требования FR-005, FR-006 (отзыв, бездействие 8 ч, 24 ч) | ADR-0005, D-030 |
| Добавить `big.js` | Расчёт корзины в реальном времени без `float` | D-027 |
| Добавить `openapi-typescript` | Один источник контракта API | D-027 |
| Добавить ArchUnit | Автоматическая проверка границ модульного монолита | ADR-0001 |
| Без Redis, Kafka, Kubernetes, микросервисов | Нет требования, которое их оправдывает (раздел 3.3) | ADR-0001, ADR-0004 |

Spring Modulith рассматривался как альтернатива ArchUnit для проверки модулей. Он удобнее, но добавляет зависимость, совместимость которой с Boot 4.2 нужно отдельно проверять; ArchUnit не зависит от версии Spring. Решение можно пересмотреть в Фазе 4.

---

## 3. Архитектурный стиль: модульный монолит

### 3.1. Решение

Один развёртываемый backend (Spring Boot JAR) с явными модулями-пакетами, одна база PostgreSQL, SPA на React, раздаваемая Nginx. Модули общаются **прямыми вызовами Java-интерфейсов** в одной транзакции, а не сообщениями.

### 3.2. Почему не микросервисы

| Критерий | Модульный монолит | Микросервисы |
|---|---|---|
| Атомарность продажи (остатки, документ, аудит, ключ) | Одна транзакция БД | Распределённая транзакция или сага с компенсациями |
| Нагрузка | 10 магазинов × 3 пользователя — одного процесса хватает с запасом в 100 раз | Нет выигрыша |
| Эксплуатация одним человеком | Один контейнер приложения | Сервис-дискавери, трассировка, N конвейеров |
| Стоимость | 1 сервер 2 vCPU / 4 ГБ | Несколько сервисов, брокер |
| Защита диплома | Границы модулей видны и проверяются тестом | Границы видны, но цена несоразмерна задаче |

Модули спроектированы так, чтобы при необходимости (R3) выделить, например, Reporting в отдельный сервис чтения: они уже общаются только через публичные интерфейсы.

### 3.3. Почему без Redis, Kafka, Kubernetes

| Технология | Для чего могла бы понадобиться | Почему не нужна |
|---|---|---|
| Redis | Сеансы, кэш, ограничение частоты | Сеансы — refresh-токены в PostgreSQL; кэш прав — Caffeine в процессе; ограничение частоты — Bucket4j в памяти и `limit_req` в Nginx. Один экземпляр приложения |
| Kafka / RabbitMQ | События между модулями | Модули в одном процессе; критичные действия должны быть в одной транзакции (AC-AUD-05) |
| Kubernetes | Оркестрация и масштабирование | Один сервер; Docker Compose с `restart: unless-stopped` и проверками здоровья |
| Elasticsearch | Поиск товаров | `pg_trgm` укладывается в NFR-003 (5 000 товаров, ≤ 500 мс) |

Условие пересмотра: несколько экземпляров backend (горизонтальное масштабирование) — тогда ограничение частоты и кэш прав переезжают в Redis или PostgreSQL. Записано в ADR-0004.

---

## 4. Модули

### 4.1. Ответственность и данные

| Модуль | Ответственность | Владеет таблицами | Требования |
|---|---|---|---|
| **shared** (платформа) | Деньги и количества (`Money`, `Quantity`), ошибки (`BusinessException`, коды), контекст магазина и пользователя, бизнес-время (`BusinessClock`), идемпотентность, аудит, нумерация документов | `idempotency_key`, `audit_log`, `document_sequence`, `job_run` | DIR-006, DIR-007, FR-211–FR-214 |
| **auth** | Вход, выход, выпуск и ротация токенов, блокировка после неудач, смена пароля | `refresh_token` | FR-001–FR-008 |
| **users** | Сотрудники, роли, делегированные права, лимит учётных записей | `app_user`, `user_permission` | FR-013–FR-018 |
| **stores** | Регистрация магазина, настройки | `store`, `store_settings` | FR-009, FR-011, FR-012 |
| **catalog** | Товары, категории, штрихкоды, розничные цены, импорт | `product`, `category`, `product_barcode`, `product_price_history` | FR-021–FR-038 |
| **suppliers** | Поставщики, связь товар–поставщик, сроки поставки | `supplier`, `product_supplier` | FR-041–FR-047 |
| **inventory** | Партии, остатки, средняя себестоимость, движения, FEFO, списания, инвентаризация, сверка | `product_stock`, `batch`, `stock_movement`, `write_off*`, `stock_count*` | FR-061–FR-087, FR-121–FR-137 |
| **purchases** | Приёмки и заказы поставщику | `purchase*`, `purchase_order*` | FR-051–FR-060 |
| **sales** | Касса, чек, скидки, статистика продаж (D) | `sale`, `sale_item`, `sale_batch_allocation` | FR-091–FR-107 |
| **returns** | Возвраты покупателей | `customer_return`, `return_item`, `return_allocation` | FR-111–FR-118 |
| **expenses** | Расходы, категории, шаблоны | `expense*`, `recurring_expense` | FR-161–FR-166 |
| **finance** | Валовая и операционная прибыль, P&L, стоимость склада, маржинальность (только чтение) | — | FR-152–FR-154, FR-076 |
| **expiration** | Списки сроков, оценка риска, центр действий, «сохранено», ночной пересчёт | `batch_risk_event`, `expiry_action` | FR-141–FR-147, FR-181–FR-187 |
| **replenishment** | Объяснимые рекомендации, группировка по поставщикам, создание заказа из рекомендаций | — | FR-191–FR-198 |
| **reporting** | Отчёты о продажах, панели OWNER и STAFF, экспорт CSV/XLSX, журнал аудита для просмотра | — | FR-151, FR-155–FR-158, FR-171–FR-175, FR-212 |

**Где списания и инвентаризация.** Требование Фазы 3 перечисляет 14 модулей без отдельного модуля списаний. Списания и инвентаризация помещены в **inventory**: они меняют остатки и среднюю под теми же блокировками, что и продажа, и их нельзя отделить без циклической зависимости.

**Где себестоимость.** `CostingService` находится в inventory, потому что средняя меняется атомарно с остатком в строке `product_stock`. Finance только читает зафиксированные суммы. Так у Finance нет зависимости «на запись» от склада, и цикл inventory ↔ finance невозможен.

### 4.2. Зависимости (ацикличный граф)

```
shared ◄── все модули

stores ◄── users ◄── auth
stores ◄── catalog ◄── suppliers
catalog ◄── inventory
inventory, suppliers, catalog ◄── purchases
inventory, catalog ◄── sales
sales, inventory ◄── returns
stores ◄── expenses
sales, returns, inventory, expenses ◄── finance
inventory, sales, catalog ◄── expiration
expiration, purchases, suppliers, sales, inventory ◄── replenishment
finance, expiration, replenishment, sales, users ◄── reporting
```

Диаграмма: [`diagrams/component.md`](diagrams/component.md). Топологический порядок (он же порядок реализации): shared → stores → users → auth → catalog → suppliers → inventory → purchases → sales → returns → expenses → finance → expiration → replenishment → reporting.

Как разрываются «естественные» циклы:

| Потенциальный цикл | Решение |
|---|---|
| purchases ↔ replenishment (рекомендации создают заказ, заказы влияют на рекомендации) | replenishment вызывает `PurchaseOrderApi.createFromRecommendations(...)` и читает `ExpectedDeliveriesApi`; purchases о пополнении не знает |
| inventory ↔ sales (продажа уменьшает остаток, отчёт склада читает продажи) | sales вызывает `StockApi.allocateAndIssue(...)`; inventory не знает о продажах, документ передаётся как `DocumentRef(type, id, lineId)` |
| returns ↔ inventory (DAMAGED-возврат создаёт списание) | returns вызывает `WriteOffApi.createReturnDamaged(...)` |
| Любой модуль ↔ audit | Аудит в shared; модули вызывают `AuditLog.record(...)` в своей транзакции |
| expiration ↔ replenishment (оба используют прогноз FEFO) | Функция `ExpectedSalesProjection` в expiration; replenishment её вызывает |

### 4.3. Правила границ

1. У каждого модуля пакет `api` — единственное, что видят другие модули: интерфейсы сервисов, DTO-записи (`record`), события. Всё остальное — `internal`.
2. Модуль не обращается к таблицам и JPA-сущностям другого модуля. Связь между таблицами разных модулей — только по id (FK в БД есть, ассоциаций JPA между модулями нет).
3. Транзакцию открывает модуль, который владеет процессом (sales для продажи); вызовы `api` других модулей участвуют в ней (`Propagation.MANDATORY` для методов, меняющих остатки).
4. ArchUnit-тест в CI проверяет: нет циклов между модулями; `..internal..` не используется извне; `web`-слой не обращается к репозиториям; классы домена не зависят от Spring Web.

### 4.4. Слои внутри модуля

Без «чистой архитектуры» с портами и адаптерами для каждого модуля: для CRUD-справочников это лишние абстракции. Слои:

| Пакет | Что лежит | Пример |
|---|---|---|
| `api` | Публичные интерфейсы и DTO для других модулей | `StockApi`, `BatchView` |
| `web` | Контроллеры REST, DTO запросов и ответов (раздельные для OWNER/STAFF), маппинг | `SaleController`, `SaleResponseOwner`, `SaleResponseStaff` |
| `application` | Сервисы сценариев, транзакции, проверка прав на уровне сценария | `PostSaleService` |
| `domain` | Сущности JPA, перечисления, **чистые функции** алгоритмов без Spring и БД | `Batch`, `FefoAllocator`, `WeightedAverageCost`, `RiskCalculator` |
| `persistence` | Репозитории Spring Data, запросы с блокировками, проекции для отчётов | `BatchRepository.lockCandidates(...)` |

Чистые функции алгоритмов (`domain`) тестируются юнит- и property-based тестами без контекста Spring — это основа NFR-050 (≥ 80% покрытия ядра).

---

## 5. Сквозные механизмы backend

### 5.1. Контекст магазина

Фильтр безопасности проверяет JWT и кладёт `CurrentUser(userId, storeId, role, permissions, tokenVersion)` в `SecurityContext`. Сервисы получают `storeId` только из `CurrentUser` (BR-121). Все методы репозиториев принимают `storeId` первым параметром; ArchUnit-правило запрещает методы `findById(id)` без `storeId` в репозиториях бизнес-таблиц. Hibernate `@Filter` не используется как основной механизм: явный параметр виднее в коде и в тестах.

### 5.2. Ошибки

Единый формат — RFC 9457 Problem Details с расширениями `code`, `params`, `errors[]` (подробно в `API.md`, раздел 3). `BusinessException(code, params)` в домене → `@RestControllerAdvice` → ответ. Стек вызовов, SQL и внутренние id не попадают в ответ (SEC-013); полная ошибка пишется в журнал с `requestId`.

### 5.3. Транзакции и блокировки

`@Transactional` на сервисах сценариев, уровень `READ COMMITTED`. Процессы, меняющие остатки, используют `LockService.lockProducts(storeId, productIds)` и `lockBatches(...)` с порядком по возрастанию id и `SET LOCAL lock_timeout = '5s'` (ALGORITHMS, раздел 7). SQLSTATE `55P03`/`40P01` преобразуются в `CONCURRENCY_RETRY`.

### 5.4. Время

`BusinessClock` даёт `Instant now()` и `LocalDate today()` в `Asia/Bishkek`. В тестах — фиксированные часы. Нигде в коде нет `LocalDate.now()` без часов (правило ArchUnit).

### 5.5. Деньги и количества

`Money` (обёртка над `BigDecimal` scale 2, HALF_UP), `UnitCost` (scale 4), `Quantity` (scale 3, проверка целого для PCS). JPA-конвертеры на `numeric`. В JSON — строки (`"1234.56"`). Статическая проверка запрещает `double`/`float` в пакетах `domain` и `web` (NFR-022).

### 5.6. Сериализация по роли

Контроллер выбирает DTO по роли: `ProductOwnerResponse` содержит `avgCost`, `ProductStaffResponse` — нет. Поле нельзя «забыть скрыть», потому что его нет в классе STAFF (ROLE_PERMISSIONS, раздел 4.3). Тест сравнивает OpenAPI-схемы STAFF-ответов со списком запрещённых полей.

### 5.7. Фоновые задачи

`@Scheduled` с часовым поясом `Asia/Bishkek`:

| Задача | Время | Действие |
|---|---|---|
| Пересчёт событий риска | 00:05 | Для каждого магазина: уровни риска, запись `batch_risk_event` (FR-144, FR-187) |
| Сверка инвариантов | 03:00 | DIR-002; нарушения — в аудит и на панель (FR-075) |
| Повторяющиеся расходы | 00:10 | FR-166 |
| Очистка токенов | 04:00 | Удаление цепочек refresh-токенов старше 30 дней |

Экземпляр приложения один; защита от двойного запуска после перезапуска — таблица `job_run` и `pg_try_advisory_lock`. ShedLock не нужен.

### 5.8. Наблюдаемость

Spring Boot Actuator: `/actuator/health` (liveness/readiness, доступен только из сети Docker), `/actuator/prometheus` не публикуется в MVP. Журнал — JSON в stdout с `requestId`, `storeId`, `userId`; пароли, токены и тела запросов кассы не логируются. Docker хранит журналы с ротацией (`max-size`).

### 5.9. OpenAPI

springdoc генерирует `/v3/api-docs` (доступ только в профиле `dev`). В CI спецификация выгружается в `docs/api/openapi.json`, сравнивается с закоммиченной (NFR-052) и используется для генерации типов frontend.

---

## 6. Frontend

### 6.1. Принципы

- SPA (React Router, data router), раздаётся Nginx как статика; API на том же домене (`/api/v1`) — CORS не нужен.
- Серверное состояние — только TanStack Query; локальное состояние кассы — `useReducer` + `localStorage` (корзина и ключ идемпотентности переживают перезагрузку, FR-097). Глобальный менеджер состояния не нужен.
- Формы — React Hook Form + Zod. Схемы Zod повторяют правила сервера для подсказок, но **сервер — единственный источник истины** (BR-210).
- Деньги — строки из API → `big.js` → форматирование `Intl.NumberFormat('ru-KG')`.

### 6.2. Структура по функциям (feature-sliced, упрощённо)

```
frontend/src/
├── app/                    # точка входа, роутер, провайдеры (QueryClient, i18n, auth)
├── shared/
│   ├── api/                # openapi-fetch клиент, сгенерированные типы, перехватчик токена
│   ├── ui/                 # компоненты shadcn/ui (Button, Dialog, Table, ...)
│   ├── lib/                # money.ts (big.js), quantity.ts, dates.ts (Asia/Bishkek), errors.ts
│   ├── i18n/               # инициализация i18next
│   └── auth/               # контекст пользователя, guard по роли и праву
├── features/
│   ├── auth/               # вход, смена пароля
│   ├── pos/                # касса: корзина, сканер, оплата, повтор продажи
│   ├── catalog/            # товары, категории, штрихкоды, импорт
│   ├── suppliers/
│   ├── purchases/          # приёмки, заказы поставщику
│   ├── inventory/          # остатки, партии, движения, инвентаризация
│   ├── write-offs/
│   ├── returns/
│   ├── expiry/             # сроки, центр действий, риск
│   ├── replenishment/
│   ├── expenses/
│   ├── reports/            # отчёты, P&L, графики Recharts
│   ├── dashboard/          # панели OWNER и STAFF
│   ├── staff/              # сотрудники и права
│   ├── settings/
│   └── audit/
└── locales/ru/*.json       # пространства имён common, pos, errors, ...
```

Каждая папка `features/x` содержит `routes.tsx`, `api.ts` (хуки TanStack Query), `components/`, `schemas.ts` (Zod). Функции не импортируют друг друга, только `shared` (правило ESLint `import/no-restricted-paths`).

### 6.3. Маршруты и доступ

`/login`, `/pos`, `/dashboard`, `/catalog/...`, `/purchases/...`, `/inventory/...`, `/expiry/...`, `/replenishment`, `/reports/...`, `/expenses`, `/staff`, `/settings`, `/audit`. Маршрут объявляет требуемую роль или право; guard скрывает недоступное (FR-010). Это удобство, а не защита.

### 6.4. Токены на клиенте

Access-токен хранится **только в памяти** (переменная модуля `shared/api`), не в `localStorage`. При 401 клиент один раз вызывает `POST /auth/refresh` (refresh-токен в httpOnly-cookie) и повторяет запрос; параллельные запросы ждут один общий refresh. При старте приложения — тихий refresh. Подробно — `SECURITY.md`, раздел 2.

### 6.5. Касса и сбои сети

1. Корзина и `idempotencyKey` (создаётся при нажатии «Оплата», BR-050) сохраняются в `localStorage`.
2. Ошибка сети, 409 `CONCURRENCY_RETRY` или `REQUEST_IN_PROGRESS` → повтор с тем же ключом через 0,5 / 1 / 2 с.
3. После трёх неудач — баннер «Продажа не подтверждена, повторить», корзина заблокирована для изменений.
4. Ответ 201/200 → печать нефискального чека, очистка корзины и ключа.

### 6.6. Локализация: только русский

Решение D-026: интерфейс только на русском. i18next **сохраняется**:

- все строки — в ресурсах `locales/ru/{namespace}.json`, а не в коде; это проверяемо (FR-207 переформулирован: «0 строк интерфейса вне ресурсов»);
- ошибки сервера переводятся по коду (`errors.STOCK_INSUFFICIENT`), объяснения риска и пополнения — по шаблонам с параметрами (FR-208);
- формы множественного числа `one/few/many` — встроенный механизм i18next через `Intl.PluralRules('ru')`; ICU MessageFormat не нужен;
- переключателя языка нет; `lng: 'ru'`, `fallbackLng: 'ru'`.

Стоимость сохранения i18next невелика (≈ 15 КБ gzip), а добавление второго языка в будущем не потребует переписывать интерфейс.

### 6.7. Производительность

Разделение кода по маршрутам (`lazy`), касса — отдельный чанк. Бюджет стартового JS ≤ 500 КБ gzip (NFR-006) проверяется в CI (`vite build` + проверка размера). Поиск в кассе — запрос к серверу с задержкой 150 мс, штрихкод — точный запрос без задержки.

---

## 7. Организация Java-пакетов

```
backend/src/main/java/kg/groceryerp/
├── GroceryErpApplication.java
├── shared/
│   ├── money/              # Money, UnitCost, Quantity, конвертеры JPA
│   ├── error/              # BusinessException, ErrorCode, ProblemDetails advice
│   ├── security/           # CurrentUser, SecurityConfig, JwtConfig, @RequiresPermission
│   ├── tenant/             # StoreScoped, StoreContext
│   ├── time/               # BusinessClock
│   ├── idempotency/        # IdempotencyService, RequestHasher
│   ├── audit/              # AuditLog (api), AuditLogRepository
│   ├── numbering/          # DocumentNumberService
│   └── web/                # PageResponse, сортировка, общие валидаторы
├── auth/        {api, web, application, domain, persistence}
├── users/       {api, web, application, domain, persistence}
├── stores/      ...
├── catalog/     ...
├── suppliers/   ...
├── inventory/
│   ├── api/                # StockApi, WriteOffApi, BatchQueryApi, LockService
│   ├── web/
│   ├── application/        # PostWriteOffService, PostStockCountService, ReconciliationJob
│   ├── domain/             # Batch, ProductStock, StockMovement, FefoAllocator, WeightedAverageCost
│   └── persistence/
├── purchases/   ...
├── sales/
│   ├── api/                # SalesStatistics (D)
│   ├── web/                # SaleController, SaleQuoteController
│   ├── application/        # PostSaleService, PricingService
│   ├── domain/             # Sale, SaleItem, SaleBatchAllocation, ReceiptDiscountDistributor
│   └── persistence/
├── returns/     ...
├── expenses/    ...
├── finance/     {api, web, application}           # без собственных таблиц
├── expiration/  {api, web, application, domain, persistence}
├── replenishment/ {web, application, domain}
└── reporting/   {web, application, export}
```

Корневой пакет `kg.groceryerp` — предложение; автор может выбрать другой (например, по домену университета).

---

## 8. Структура репозитория

```
grocery-erp/
├── backend/
│   ├── .mvn/wrapper/               # Maven Wrapper
│   ├── mvnw, mvnw.cmd
│   ├── pom.xml
│   ├── Dockerfile                  # многоэтапная сборка, образ eclipse-temurin:21-jre
│   └── src/
│       ├── main/java/kg/groceryerp/...
│       ├── main/resources/
│       │   ├── application.yml     # без секретов; значения из переменных окружения
│       │   └── db/migration/       # V001__baseline.sql, V002__..., R__grants.sql
│       └── test/java/...           # unit, integration (Testcontainers), architecture (ArchUnit)
├── frontend/
│   ├── package.json, package-lock.json
│   ├── vite.config.ts, tsconfig.json, eslint.config.js, components.json (shadcn)
│   ├── index.html
│   ├── src/...                     # раздел 6.2
│   ├── tests/e2e/                  # Playwright
│   └── Dockerfile                  # сборка статики → образ nginx
├── infra/
│   ├── nginx/nginx.conf            # TLS, CSP, проксирование /api, статика SPA
│   ├── nginx/conf.d/
│   ├── postgres/init/              # создание ролей grocery_owner/app/backup
│   ├── backup/                     # скрипт pg_dump + шифрование + выгрузка
│   └── k6/                         # нагрузочные сценарии
├── docs/
│   ├── research/                   # Фаза 1
│   ├── *.md                        # Фазы 2–3
│   ├── diagrams/
│   ├── adr/
│   └── api/openapi.json            # генерируется в CI (Фаза 4)
├── .github/
│   ├── workflows/ci.yml            # backend, frontend, e2e, openapi-drift, секреты
│   ├── workflows/deploy.yml        # ручной запуск
│   ├── dependabot.yml
│   └── pull_request_template.md
├── docker-compose.yml              # production-подобный стек
├── docker-compose.dev.yml          # переопределения для локальной разработки
├── .env.example                    # имена переменных без значений
├── .gitignore                      # .env, target/, node_modules/, dist/, *.dump
├── CLAUDE.md                       # правила для ИИ-ассистента в проекте
└── README.md                       # запуск, структура, ссылки на docs
```

Файлы `docker-compose.yml`, `.env.example`, `README.md` и каркас кода создаются в Фазе 4; их содержание описано в `DEPLOYMENT_ARCHITECTURE.md`. В Фазе 3 создан только `CLAUDE.md`.

---

## 9. Соответствие нефункциональным требованиям

| NFR | Как обеспечивается |
|---|---|
| NFR-001–NFR-003 (касса, поиск) | Индексы раздела 8 `DATABASE.md`; одна транзакция; без сетевых переходов между сервисами |
| NFR-004, NFR-005 (отчёты, панель) | Агрегаты по `business_date` с индексами; при необходимости — материализованные представления (не в MVP) |
| NFR-007 (пересчёт риска) | Один проход по партиям магазина в порядке FEFO, O(n log n) |
| NFR-011, NFR-021 (нет потерь и частичных документов) | Ответ после COMMIT; одна транзакция |
| NFR-020, NFR-024 (инварианты, конкуренция) | CHECK, блокировки, тесты ALGORITHMS 7.3 |
| NFR-022 (деньги) | `BigDecimal`/`numeric`/строки/`big.js`; запрет `float` |
| NFR-023 (идемпотентность) | ALGORITHMS, раздел 8 |
| NFR-030–NFR-036 (безопасность) | `SECURITY.md` |
| NFR-050–NFR-053 (сопровождаемость) | Тесты, ArchUnit, CI, OpenAPI drift, ADR |
| NFR-061 (10 магазинов на 2 vCPU / 4 ГБ) | JVM `-Xmx1g`, PostgreSQL `shared_buffers=512MB`, Nginx — раздел 4 `DEPLOYMENT_ARCHITECTURE.md` |
| NFR-070–NFR-072 (резервные копии) | `DEPLOYMENT_ARCHITECTURE.md`, раздел 6 |

---

## 10. Влияние решения «только русский язык» на архитектуру

| Было (Фаза 2) | Стало (D-026) |
|---|---|
| `product.name_ru`, `product.name_ky`, то же у категорий | Одно поле `name` |
| `app_user.language`, `store_settings.default_language` | Удалены |
| Переключатель RU/KY, сохранение языка в профиле | Удалён |
| `locales/{ru,ky}/...`, проверка полноты ky в CI | `locales/ru/...`, проверка «нет строк вне ресурсов» |
| `Accept-Language` для `message` в ошибках API | `message` всегда на русском |
| Экспорт на языке пользователя | Экспорт на русском |
| Поиск с учётом ң, ө, ү | Поиск по `lower(name)` с `pg_trgm`; кыргызские буквы в названиях по-прежнему допустимы как данные |

Полный список затронутых требований — `DECISIONS.md`, D-026.
