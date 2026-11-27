# REST API

**Фаза 3 · Grocery ERP** · 27.11.2026 · статус: черновик на утверждение

Контракт REST API backend. В Фазе 4 он оформляется аннотациями springdoc, а OpenAPI-спецификация (`docs/api/openapi.json`) генерируется и проверяется в CI. Этот документ — источник истины до появления кода.

---

## 1. Общие правила

| Тема | Правило |
|---|---|
| Базовый путь | `/api/v1`. Несовместимые изменения — только в `/api/v2` |
| Формат | JSON, UTF-8, `Content-Type: application/json` (кроме экспорта и импорта файлов) |
| Имена полей | `camelCase` |
| Деньги | Строка с 2 знаками: `"1234.56"`. Себестоимость единицы — 4 знака: `"57.5000"` (BR-070) |
| Количества | Строка с 3 знаками: `"1.250"`; для PCS — `"3.000"` или `"3"` на входе |
| Проценты | Строка: `"10.00"` |
| Дата | `YYYY-MM-DD` (бизнес-дата Asia/Bishkek) |
| Момент времени | ISO 8601 UTC: `2026-11-26T17:30:00Z`; клиент показывает в Asia/Bishkek |
| Идентификаторы | Целые числа (`id`), номера документов — отдельное поле `number` |
| Аутентификация | `Authorization: Bearer <access-token>` для всех маршрутов, кроме `/auth/login`, `/auth/refresh`, `/stores/register` |
| Магазин | Из токена. Параметр `storeId` в API **отсутствует** (BR-121) |
| Язык | Только русский; поле `message` в ошибках — на русском (D-026) |
| Трассировка | Ответ всегда содержит заголовок `X-Request-Id` |

### 1.1. Пагинация

Списочные маршруты принимают `page` (с 0), `size` (по умолчанию 20, максимум 100) и `sort`:

```
GET /api/v1/products?page=0&size=50&sort=name,asc&sort=code,desc
```

```json
{ "items": [ ... ], "page": 0, "size": 50, "totalItems": 1234, "totalPages": 25 }
```

Поля сортировки — белый список для каждого маршрута; неизвестное поле → 400 `VALIDATION_SORT_FIELD`. Журнал аудита и движения (большие таблицы) используют курсорную пагинацию: `?cursor=<opaque>&size=50`, ответ `{ "items": [...], "nextCursor": "..." }`.

### 1.2. Фильтры

Простые параметры запроса: `?categoryId=5&active=true&q=молоко&from=2026-11-01&to=2026-11-30`. Периоды — включительно по бизнес-датам. Максимальный период отчёта — 366 дней (`VALIDATION_PERIOD_TOO_LONG`).

### 1.3. Идемпотентность

Заголовок `Idempotency-Key: <UUID v4>` **обязателен** для проведения документов (BR-054):

| Маршрут | Операция |
|---|---|
| `POST /sales` | SALE |
| `POST /returns` | RETURN |
| `POST /purchases/{id}/post` | PURCHASE_POST |
| `POST /purchases/{id}/cancel` | PURCHASE_CANCEL |
| `POST /write-offs`, `POST /write-offs/bulk-expired` | WRITE_OFF |
| `POST /write-offs/{id}/reverse` | WRITE_OFF_REVERSAL |
| `POST /stock-counts/{id}/post`, `POST /stock-adjustments` | STOCK_COUNT_POST |

| Ситуация | Ответ |
|---|---|
| Первый запрос | 201 Created (или 200 для `/post`) |
| Повтор с тем же ключом и тем же телом | **200 OK** с тем же документом и заголовком `Idempotent-Replayed: true` |
| Тот же ключ, другое тело | 409 `IDEMPOTENCY_CONFLICT` |
| Первый запрос ещё выполняется | 409 `REQUEST_IN_PROGRESS`, `Retry-After: 1` |
| Нет заголовка | 400 `IDEMPOTENCY_KEY_REQUIRED` |

Алгоритм — `ALGORITHMS.md`, раздел 8.

### 1.4. Оптимистическая блокировка справочников

Изменяемые справочники (товар, поставщик, настройки, расход) возвращают поле `version`. `PUT`/`PATCH` обязан передать его; несовпадение → 409 `VERSION_CONFLICT` (кто-то изменил запись раньше).

---

## 2. Коды состояния HTTP

| Код | Когда |
|---|---|
| 200 OK | Чтение; изменение; проведение существующего документа; идемпотентный повтор |
| 201 Created | Создание; заголовок `Location` |
| 204 No Content | Удаление черновика; выход |
| 400 Bad Request | Ошибка формата и валидации полей (`VALIDATION_*`) |
| 401 Unauthorized | Нет токена, токен истёк или отозван (`AUTH_TOKEN_EXPIRED`, `AUTH_INVALID`) |
| 403 Forbidden | Нет роли или права (`ACCESS_DENIED`), тело без данных (FR-004) |
| 404 Not Found | Объекта нет **или он принадлежит другому магазину** (BR-122); для STAFF — и чужой документ |
| 409 Conflict | Состояние не позволяет операцию: `STOCK_INSUFFICIENT`, `EXPIRED_ONLY`, `DOCUMENT_NOT_DRAFT`, `VERSION_CONFLICT`, `CONCURRENCY_RETRY`, `IDEMPOTENCY_CONFLICT`, `PRICE_CHANGED` |
| 422 Unprocessable Content | Бизнес-правило нарушено при корректном формате: `DISCOUNT_LIMIT_EXCEEDED`, `RETURN_WINDOW_EXPIRED`, `EXPIRED_ON_RECEIPT`, `COMMENT_REQUIRED` |
| 423 Locked | Учётная запись временно заблокирована после неудачных входов (`AUTH_LOCKED`, FR-003) |
| 429 Too Many Requests | Превышен лимит частоты, заголовок `Retry-After` |
| 500 | Внутренняя ошибка, без подробностей (SEC-013) |

Правило выбора 409 и 422: **409** — конфликт с текущим состоянием данных, который может исчезнуть (остаток, версия, параллельный запрос); **422** — запрос нарушает правило и повтор без изменения запроса бессмыслен.

---

## 3. Формат ошибки

RFC 9457 Problem Details (`application/problem+json`) с расширениями проекта (LOC-006):

```json
{
  "type": "/problems/stock-insufficient",
  "title": "Недостаточно товара",
  "status": 409,
  "code": "STOCK_INSUFFICIENT",
  "message": "Недостаточно товара «Молоко 1 л»: доступно 2 шт.",
  "params": { "productId": 42, "available": "2.000", "unit": "PCS" },
  "errors": [
    { "field": "lines[0].quantity", "code": "STOCK_INSUFFICIENT", "params": { "available": "2.000" } }
  ],
  "requestId": "6f1c2d3e-...",
  "timestamp": "2026-11-26T17:30:00Z"
}
```

- `code` — стабильный машинный код; клиент переводит по `errors.<code>` из `locales/ru/errors.json`.
- `message` — готовый русский текст для прямых клиентов API.
- `errors[]` — ошибки полей (валидация Bean Validation и бизнес-ошибки конкретной строки).
- В `params` никогда нет данных, скрытых от роли: для STAFF ошибка `BELOW_COST` не возникает (BR-085), а `STOCK_INSUFFICIENT` не содержит себестоимости.

**Справочник кодов** (основные): `VALIDATION_REQUIRED`, `VALIDATION_RANGE`, `VALIDATION_PATTERN`, `VALIDATION_QUANTITY_SCALE`, `VALIDATION_AMOUNT`, `AUTH_INVALID_CREDENTIALS`, `AUTH_LOCKED`, `AUTH_TOKEN_EXPIRED`, `AUTH_REFRESH_REUSED`, `ACCESS_DENIED`, `NOT_FOUND`, `VERSION_CONFLICT`, `STOCK_INSUFFICIENT`, `EXPIRED_ONLY`, `EXPIRED_BATCH`, `BATCH_BLOCKED`, `EXPIRED_ON_RECEIPT`, `PRICE_CHANGED`, `DISCOUNT_LIMIT_EXCEEDED`, `DISCOUNT_EXCEEDS_AMOUNT`, `ZERO_TOTAL_NOT_ALLOWED`, `RETURN_QTY_EXCEEDED`, `RETURN_WINDOW_EXPIRED`, `RETURN_STAFF_TODAY_ONLY`, `COMMENT_REQUIRED`, `REASON_EXPIRED_NOT_EXPIRED`, `DOCUMENT_NOT_DRAFT`, `BATCH_ALREADY_USED`, `BATCH_SET_CHANGED`, `COUNT_STALE`, `COST_REQUIRED`, `UNIT_LOCKED`, `BARCODE_DUPLICATE`, `BARCODE_CHECKSUM`, `CATEGORY_NOT_EMPTY`, `CATEGORY_DEPTH`, `USER_LIMIT_REACHED`, `LAST_OWNER`, `IDEMPOTENCY_KEY_REQUIRED`, `IDEMPOTENCY_CONFLICT`, `REQUEST_IN_PROGRESS`, `CONCURRENCY_RETRY`, `RATE_LIMITED`.

---

## 4. Авторизация

Каждый маршрут объявляет требование аннотацией (`@RequiresRole(OWNER)`, `@RequiresPermission("purchase.receive")`, `@Authenticated`). Маршрут без аннотации не регистрируется — проверка при старте (ROLE_PERMISSIONS, раздел 4.1). Порядок проверки: 401 → `storeId` из токена → 403 → 404 → «своё» для STAFF → 409/422.

Обозначения в таблицах: **O** — только OWNER; **A** — любой аутентифицированный; **P:x** — OWNER или STAFF с правом `x`; **(разд.)** — разные DTO для OWNER и STAFF.

---

## 5. Маршруты

### 5.1. Аутентификация (`auth`)

| Метод | Путь | Доступ | Описание |
|---|---|---|---|
| POST | `/auth/login` | публично | Вход. Ответ: access-токен в теле, refresh — `Set-Cookie` |
| POST | `/auth/refresh` | cookie | Ротация refresh, новый access |
| POST | `/auth/logout` | A | Отзыв цепочки refresh, удаление cookie, 204 |
| POST | `/auth/logout-all` | A | Отзыв всех сеансов пользователя (`token_version + 1`) |
| GET | `/auth/me` | A | Профиль: id, имя, роль, права, магазин, `mustChangePassword` |
| POST | `/auth/change-password` | A | Смена своего пароля (FR-007) |

`LoginRequest`:

| Поле | Тип | Валидация |
|---|---|---|
| `login` | string | 3–40, `^[A-Za-z0-9._-]+$`, приводится к нижнему регистру |
| `password` | string | 1–128 (длина не раскрывает политику) |

`LoginResponse`: `{ "accessToken": "eyJ...", "expiresIn": 900, "user": MeResponse }`. Cookie: `refresh_token=<random>; HttpOnly; Secure; SameSite=Strict; Path=/api/v1/auth; Max-Age=86400`.

Ошибки входа: 401 `AUTH_INVALID_CREDENTIALS` одинаково для неверного логина и пароля (не раскрывает существование логина); 423 `AUTH_LOCKED` с `params.lockedUntil`; 429 при превышении частоты.

`ChangePasswordRequest`: `currentPassword`, `newPassword` (8–128, не совпадает с логином и текущим паролем). Успех → 200 и новые токены; остальные сеансы отозваны (FR-006).

### 5.2. Магазин и сотрудники (`stores`, `users`)

| Метод | Путь | Доступ | Описание |
|---|---|---|---|
| POST | `/stores/register` | публично (+ код приглашения, если включён) | Регистрация магазина и первого OWNER (FR-009) |
| GET | `/store` | A (разд.) | Настройки; STAFF — ограниченный набор (ROLE_PERMISSIONS §2) |
| PUT | `/store` | O | Изменение названия, адреса, телефона, порогов, лимитов (FR-011) |
| GET | `/users` | O | Сотрудники с последним входом (FR-017) |
| POST | `/users` | O | Создание STAFF (FR-013), 422 `USER_LIMIT_REACHED` |
| GET | `/users/{id}` | O | |
| PUT | `/users/{id}` | O | Имя |
| POST | `/users/{id}/deactivate` | O | 409 `LAST_OWNER` для последнего OWNER (FR-018) |
| POST | `/users/{id}/activate` | O | |
| POST | `/users/{id}/reset-password` | O | Временный пароль для STAFF (FR-008) |
| PUT | `/users/{id}/permissions` | O | `{ "permissions": ["purchase.receive"] }` (FR-015) |

`RegisterStoreRequest`: `storeName` (1–120), `ownerFullName` (1–100), `login`, `password` (8–128), `inviteCode` (если включён), `acceptTerms: true`. Ответ 201 как у входа. Валюта и часовой пояс не принимаются (FR-012). Поля языка нет (D-026).

`CreateUserRequest`: `fullName`, `login`, `temporaryPassword`. Роль всегда STAFF (второго OWNER в MVP создаёт оператор, если потребуется).

### 5.3. Каталог (`catalog`)

| Метод | Путь | Доступ | Описание |
|---|---|---|---|
| GET | `/products` | A (разд.) | Список: фильтры `q`, `categoryId` (с подкатегориями, FR-038), `active`, `lowStock`, `trackExpiry`; сортировка `name`, `code`, `retailPrice`, `qtyOnHand` |
| GET | `/products/lookup?barcode=...` | A (разд.) | Касса: точный поиск по штрихкоду или коду (NFR-001) |
| GET | `/products/{id}` | A (разд.) | Карточка; OWNER видит `avgCost`, `lastPurchasePrice`, `markupPct` (FR-025) |
| POST | `/products` | O | Создание (FR-021) |
| PUT | `/products/{id}` | O | Изменение; смена цены → аудит `PRICE_CHANGED`; смена единицы после движений → 409 `UNIT_LOCKED` |
| POST | `/products/{id}/archive`, `/restore` | O | FR-023 |
| GET | `/products/{id}/price-history` | O | FR-029 |
| PUT | `/products/{id}/replenishment-settings` | O | `preferredSupplierId`, `orderMultiple`, `minOrderQty`, `safetyDays` (FR-028) |
| POST | `/products/{id}/barcodes` | O | Добавить штрихкод; 409 `BARCODE_DUPLICATE`, 400 `BARCODE_CHECKSUM` |
| DELETE | `/products/{id}/barcodes/{barcodeId}` | O | |
| POST | `/products/{id}/barcodes/generate` | O | Внутренний EAN-13 с префиксом 2 (FR-036) |
| POST | `/products/import` | O | `multipart/form-data`, CSV/XLSX, режим `dryRun=true` для проверки (FR-027) |
| GET | `/products/import/template` | O | Шаблон файла |
| GET | `/categories` | A | Дерево категорий |
| POST | `/categories` | O | |
| PUT | `/categories/{id}` | O | |
| POST | `/categories/{id}/archive` | O | 409 `CATEGORY_NOT_EMPTY` (FR-032) |

`CreateProductRequest`:

| Поле | Тип | Валидация |
|---|---|---|
| `name` | string | обязательно, 1–200 |
| `categoryId` | integer? | существует в магазине |
| `unit` | `PCS`/`KG`/`L` | обязательно |
| `retailPrice` | money | 0,00–1 000 000,00 |
| `trackExpiry` | boolean | по умолчанию true |
| `shelfLifeDays` | integer? | 1–3650; допустимо только при `trackExpiry`. Необязательно, но без него не действуют подсказка срока при приёмке и ограничения BR-142, BR-175 |
| `barcodes` | string[] | 0–10, каждый 1–32 символа; контрольная цифра EAN-8/UPC-A/EAN-13 для числовых кодов 8/12/13 знаков |

`ProductOwnerResponse` = `ProductStaffResponse` + `avgCost`, `lastPurchasePrice`, `markupPct`, `stockValue`, `preferredSupplier`, `orderMultiple`, `minOrderQty`, `safetyDays`.

`ProductStaffResponse`: `id`, `code`, `name`, `category {id, name}`, `unit`, `retailPrice`, `trackExpiry`, `shelfLifeDays`, `barcodes[]`, `qtyOnHand`, `qtyAvailable`, `active`, `version`.

### 5.4. Поставщики (`suppliers`)

| Метод | Путь | Доступ | Описание |
|---|---|---|---|
| GET | `/suppliers` | A (разд.) | STAFF — только `id`, `name` (FR-047) |
| GET | `/suppliers/{id}` | O | Контакты, сроки |
| POST, PUT | `/suppliers`, `/suppliers/{id}` | O | `name`, `contactPerson`, `phone`, `note`, `leadTimeDays` (0–60), `orderCycleDays` (1–30) |
| POST | `/suppliers/{id}/archive` | O | |
| GET | `/suppliers/{id}/products` | O | Связи товар–поставщик |
| PUT | `/suppliers/{id}/products/{productId}` | O | `leadTimeDays` (переопределение) |
| GET | `/suppliers/{id}/purchases` | O | История и суммы, неоплаченное (FR-045, FR-046) |

### 5.5. Закупки (`purchases`)

| Метод | Путь | Доступ | Описание |
|---|---|---|---|
| GET | `/purchases` | P:purchase.receive (разд.) | STAFF — только свои (ROLE_PERMISSIONS) |
| POST | `/purchases` | P:purchase.receive | Черновик |
| GET | `/purchases/{id}` | P:purchase.receive | |
| PUT | `/purchases/{id}` | P:purchase.receive | Только DRAFT, иначе 409 `DOCUMENT_NOT_DRAFT` |
| DELETE | `/purchases/{id}` | P:purchase.receive | Только DRAFT, 204 |
| POST | `/purchases/{id}/post` | P:purchase.receive | **Проведение**, `Idempotency-Key` |
| POST | `/purchases/{id}/cancel` | O | Сторно (FR-054), 409 `BATCH_ALREADY_USED` |
| PATCH | `/purchases/{id}/payment-status` | O | `PAID`/`UNPAID` (FR-046) |
| GET | `/purchase-orders` | O | Фильтр `status`, `supplierId`, `overdue=true` |
| POST | `/purchase-orders` | O | Черновик заказа |
| PUT | `/purchase-orders/{id}` | O | Только DRAFT |
| POST | `/purchase-orders/{id}/order` | O | DRAFT → ORDERED; `expectedDate` обязательна |
| POST | `/purchase-orders/{id}/cancel` | O | Отмена остатка (BR-153) |
| POST | `/purchase-orders/{id}/receive` | P:purchase.receive | Создать черновик приёмки по заказу (FR-058) |
| GET | `/purchase-orders/expected` | O | Ожидаемые поставки с отметкой просроченных (FR-059) |

`PurchaseDraftRequest`:

```json
{
  "supplierId": 3, "invoiceNumber": "НК-1182", "invoiceDate": "2026-11-26", "purchaseOrderId": null,
  "lines": [
    { "productId": 42, "quantity": "24", "unitCost": "38.50", "expiryDate": "2026-12-10", "productionDate": "2026-11-24" }
  ]
}
```

Валидация строки: `quantity > 0`, по правилам единицы (BR-062); `unitCost` 0–1 000 000 с 4 знаками; `expiryDate` обязательна при `trackExpiry`, запрещена без него; `productionDate ≤ expiryDate`. Проверка `expiryDate ≥ сегодня` и подозрительно долгого срока выполняется при проведении.

`PostPurchaseRequest`: `{ "confirmZeroCost": false, "confirmSuspiciousExpiry": false }`. Ответ 200 — документ со статусом POSTED, номером и созданными партиями `[{ lineNo, batchId, batchNumber }]`. Предупреждения без подтверждения → 422 `CONFIRMATION_REQUIRED` с `params.warnings`.

### 5.6. Склад (`inventory`)

| Метод | Путь | Доступ | Описание |
|---|---|---|---|
| GET | `/stock` | A (разд.) | Остатки: `categoryId`, `filter=LOW|EXPIRING|EXPIRED`, `q` (FR-073) |
| GET | `/stock/value` | O | Стоимость склада по средней (FR-076) |
| GET | `/stock/as-of?date=` | O | Остатки на дату (FR-077) |
| GET | `/products/{id}/batches` | A (разд.) | Партии товара по возрастанию срока (FR-062); `unitCost` только OWNER |
| GET | `/batches/{id}` | A (разд.) | |
| POST | `/batches/{id}/block`, `/unblock` | O | FR-063 |
| PUT | `/batches/{id}/markdown` | O | `{ "price": "45.00" }` < розничной (BR-084) |
| DELETE | `/batches/{id}/markdown` | O | Снять уценку |
| GET | `/movements` | A (разд.) | `productId`, `batchId`, `type`, `from`, `to`; курсорная пагинация; STAFF — без стоимостей (FR-074) |
| GET | `/write-offs` | P:stock.writeoff (разд.) | |
| POST | `/write-offs` | P:stock.writeoff | Проведение списания, `Idempotency-Key` |
| POST | `/write-offs/bulk-expired/preview` | O | Список истёкших партий и сумма |
| POST | `/write-offs/bulk-expired` | O | `{ "expectedBatchCount": 12 }`, 409 `BATCH_SET_CHANGED` (BR-106) |
| POST | `/write-offs/{id}/reverse` | O | Сторно (FR-125) |
| GET, POST | `/stock-counts` | P:stock.count | Список; создание `{ scopeType, categoryId?, productIds? }` |
| GET | `/stock-counts/{id}` | P:stock.count (разд.) | Строки с учётным остатком на момент ввода |
| PUT | `/stock-counts/{id}/lines` | P:stock.count | Ввод фактических количеств (пакетно) |
| POST | `/stock-counts/{id}/post` | P:stock.count | Проведение, 409 `COUNT_STALE` с расхождениями |
| DELETE | `/stock-counts/{id}` | P:stock.count | Только DRAFT |
| POST | `/stock-adjustments` | P:stock.count | Быстрая корректировка одного товара (FR-086) |

`CreateWriteOffRequest`:

```json
{ "lines": [ { "batchId": 901, "quantity": "3", "reason": "DAMAGED", "comment": "Помята упаковка" } ] }
```

Валидация: 1–200 строк; `reason` из списка BR-100 (кроме `RETURN_DAMAGED`); `comment` ≥ 3 символов для `OTHER`, `SHORTAGE`; `EXPIRED` только для истёкшей партии (422 `REASON_EXPIRED_NOT_EXPIRED`). STAFF получает ответ без `unitCost`, `costAmount`, `totalCost`.

### 5.7. Касса и продажи (`sales`)

| Метод | Путь | Доступ | Описание |
|---|---|---|---|
| POST | `/sales/quote` | A | Расчёт корзины без проведения: цены, распределение, скидки, предупреждения (истекает сегодня/завтра, FR-145). Ничего не блокирует и не записывает |
| POST | `/sales` | A | **Проведение продажи**, `Idempotency-Key` |
| GET | `/sales` | A (разд.) | STAFF — только свои за сегодня (FR-099); OWNER — фильтры `from`, `to`, `cashierId`, `paymentMethod`, `number` |
| GET | `/sales/{id}` | A (разд.) | Детали с распределением по партиям (FR-137) |
| GET | `/sales/{id}/receipt` | A | Данные нефискального чека для печати (FR-098) |
| GET | `/sales/my-summary` | A | Число и сумма своих чеков за сегодня (FR-158) |

`CreateSaleRequest`:

| Поле | Тип | Валидация |
|---|---|---|
| `lines` | array | 1–100 |
| `lines[].productId` | integer | активный товар магазина |
| `lines[].quantity` | quantity | > 0; PCS — целое; KG/L — ≥ 0,001; ≤ 9 999 |
| `lines[].manualBatchId` | integer? | ручной выбор партии (FR-135) |
| `lines[].discount` | `{ type: PERCENT|AMOUNT, value }`? | PERCENT 0–100; AMOUNT ≥ 0 |
| `lines[].clientUnitPrice` | money | цена, которую видел кассир; сверяется с сервером |
| `receiptDiscountPct` | percent? | 0–100 |
| `paymentMethod` | `CASH`/`CARD`/`TRANSFER` | обязательно |
| `cashReceived` | money? | обязательно для CASH, ≥ итога |
| `clientTotal` | money | итог на экране кассира; расхождение → 409 `PRICE_CHANGED` |

`SaleResponse` (общая часть): `id`, `number`, `postedAt`, `businessDate`, `cashier {id, fullName}`, `paymentMethod`, `grossAmount`, `discountAmount`, `totalAmount`, `cashReceived`, `changeAmount`, `lines[] { lineNo, productId, productName, unit, quantity, basePrice, grossAmount, lineDiscountAmount, receiptDiscountAmount, totalAmount, allocations[] { batchId, batchNumber, expiryDate, quantity, unitPrice } }`.
`SaleOwnerResponse` добавляет: `costAmount`, `grossProfit`, `lines[].unitCost`, `lines[].costAmount`, `warnings[]` (`BELOW_COST`).

Коды ответа: 201 — продажа проведена; 200 — идемпотентный повтор; 409 `STOCK_INSUFFICIENT` / `EXPIRED_ONLY` / `PRICE_CHANGED` / `CONCURRENCY_RETRY`; 422 `DISCOUNT_LIMIT_EXCEEDED` / `DISCOUNT_EXCEEDS_AMOUNT` / `ZERO_TOTAL_NOT_ALLOWED`.

Пример ответа 409 `PRICE_CHANGED`: `params.lines = [{ "lineNo": 1, "unitPrice": "85.00" }]`, `params.total = "425.00"`. Клиент обновляет корзину, показывает изменение и повторяет с **тем же** ключом (цены клиента не входят в хеш, ALGORITHMS 8.2).

### 5.8. Возвраты (`returns`)

| Метод | Путь | Доступ | Описание |
|---|---|---|---|
| GET | `/sales/{id}/returnable` | A | Что можно вернуть: строки, проданное, возвращённое, окно, признак истёкших партий |
| POST | `/returns` | A | **Проведение возврата**, `Idempotency-Key`; STAFF — только продажи сегодняшнего дня |
| GET | `/returns` | A (разд.) | STAFF — только свои за сегодня |
| GET | `/returns/{id}` | A (разд.) | |

`CreateReturnRequest`:

```json
{ "saleId": 1501, "reasonComment": "Не подошёл размер упаковки",
  "lines": [ { "saleItemId": 7731, "quantity": "1", "condition": "SALEABLE" } ] }
```

Валидация: `quantity ≤ sold − returned` (422 `RETURN_QTY_EXCEEDED`); окно (422 `RETURN_WINDOW_EXPIRED`); STAFF и не сегодняшняя продажа → 403 `RETURN_STAFF_TODAY_ONLY`. Ответ содержит `refundAmount`, строки с `forcedDamaged`; OWNER — дополнительно себестоимость.

### 5.9. Сроки годности и риск (`expiration`)

| Метод | Путь | Доступ | Описание |
|---|---|---|---|
| GET | `/expiry/batches` | A (разд.) | Группы `EXPIRED`, `TODAY`, `CRITICAL`, `HIGH`, `MEDIUM` по порогам (FR-141); STAFF — без денег (FR-146) |
| GET | `/expiry/risk` | A (разд.) | Оценка риска по партиям с объяснением (FR-181–FR-185); фильтры `level`, `categoryId` |
| GET | `/expiry/risk/summary` | O | Сумма под риском по уровням (для панели) |
| GET | `/expiry/actions` | A (разд.) | Центр действий: рекомендуемые действия (FR-142); STAFF — «выложить вперёд» и количества |
| POST | `/expiry/actions` | O / P:stock.writeoff | Отметить выполненным: `{ batchId, action, linkedDocType?, linkedDocId? }` (FR-143) |
| GET | `/expiry/saved?from=&to=` | O | «Сохранено от просрочки» (FR-186), с формулой |
| POST | `/expiry/recalculate` | O | Пересчёт по запросу (FR-144), не чаще 1 раза в минуту |

`RiskBatchResponse` (OWNER): `batchId`, `batchNumber`, `product {id, name, unit}`, `expiryDate`, `sellDays`, `qtyOnHand`, `expectedSold`, `riskQty`, `level`, `valueAtCost`, `valueAtRetail`, `recommendedAction`, `explanation { code, params }`. STAFF — без `valueAtCost`, `valueAtRetail`.

### 5.10. Пополнение (`replenishment`)

| Метод | Путь | Доступ | Описание |
|---|---|---|---|
| GET | `/replenishment/recommendations` | O | Рекомендации, сгруппированные по поставщику (FR-191, FR-193); фильтры `supplierId`, `urgentOnly`, `categoryId` |
| GET | `/replenishment/recommendations/{productId}` | O | Одна рекомендация с полным объяснением (FR-192) |
| POST | `/replenishment/orders` | O | Создать черновик заказа из выбранных: `{ supplierId, expectedDate, lines: [{ productId, quantity, recommendedQty }] }` (FR-194, FR-195) |

Рекомендации никогда не создают заказ сами (BR-178): `POST /replenishment/orders` — единственный путь, и он требует действия OWNER.

### 5.11. Расходы (`expenses`)

| Метод | Путь | Доступ | Описание |
|---|---|---|---|
| GET | `/expenses` | O | `from`, `to`, `categoryId`; итоги по категориям в ответе (FR-164) |
| POST | `/expenses` | O | `{ expenseDate, categoryId, amount, paymentMethod, comment }`; сумма > 0 и ≤ 10 000 000 |
| PUT | `/expenses/{id}` | O | С `version`; аудит старого и нового значения (FR-163) |
| DELETE | `/expenses/{id}` | O | Мягкое удаление + аудит |
| GET, POST, PUT | `/expense-categories` | O | Пользовательские категории; системные не удаляются |
| GET, POST, PUT, DELETE | `/recurring-expenses` | O | Шаблоны (FR-166) |

### 5.12. Отчёты, финансы, панель (`reporting`, `finance`)

| Метод | Путь | Доступ | Описание |
|---|---|---|---|
| GET | `/reports/sales` | O | `from`, `to`, `groupBy=DAY|PRODUCT|CATEGORY|CASHIER|PAYMENT_METHOD` (FR-151) |
| GET | `/reports/gross-profit` | O | Валовая прибыль по тем же группировкам (FR-152) |
| GET | `/reports/pnl` | O | Управленческий P&L (FR-153) |
| GET | `/reports/margin` | O | Маржинальность, топ-20 (FR-154) |
| GET | `/reports/discounts` | O | FR-107 |
| GET | `/reports/write-offs` | O | FR-126 |
| GET | `/reports/{name}/export?format=csv|xlsx` | O | Экспорт (FR-155); те же параметры |
| GET | `/dashboard/owner` | O | FR-171, FR-172 |
| GET | `/dashboard/staff` | A | FR-173, без денег, кроме своих чеков |
| GET | `/audit-log` | O | Фильтры `from`, `to`, `userId`, `eventType`; курсорная пагинация (FR-212) |

Каждый ответ отчёта содержит поле `disclaimer`: «Управленческий отчёт. Не является бухгалтерской или налоговой отчётностью» (FR-157); экспорт — ту же строку в первой строке файла.

`PnlResponse`:

```json
{ "from": "2026-11-01", "to": "2026-11-30",
  "grossSales": "...", "discounts": "...", "salesRevenue": "...", "returns": "...", "netRevenue": "...",
  "costOfSales": "...", "grossProfit": "...", "grossMarginPct": "...",
  "losses": { "writeOffs": "...", "stockCountNet": "...", "total": "..." },
  "expenses": { "total": "...", "byCategory": [ { "categoryId": 1, "name": "Аренда", "amount": "..." } ] },
  "operatingProfit": "...", "disclaimer": "..." }
```

Формулы — `ALGORITHMS.md`, раздел 9.

### 5.13. Служебные маршруты

| Метод | Путь | Доступ | Описание |
|---|---|---|---|
| GET | `/actuator/health` | только сеть Docker | Проверка здоровья для Compose и Nginx |
| GET | `/v3/api-docs`, `/swagger-ui.html` | только профиль `dev` | OpenAPI |

Сброс пароля OWNER оператором (FR-008, D-011) — **не HTTP-маршрут**, а команда CLI в контейнере backend (`java -jar app.jar --operator.reset-owner-password=<login>`) с записью в аудит. Так у неё нет поверхности атаки из интернета.

---

## 6. Сводка покрытия требований API

| Группа | Маршрутов | Связь |
|---|---|---|
| Аутентификация | 6 | M01 |
| Магазин и сотрудники | 11 | M02 |
| Каталог | 17 | M03, M04 |
| Поставщики | 7 | M05 |
| Закупки | 15 | M06 |
| Склад, списания, инвентаризация | 22 | M07, M08, M09, M13 |
| Касса | 6 | M10, M11, M14 |
| Возвраты | 4 | M12 |
| Сроки и риск | 7 | M15, M19 |
| Пополнение | 3 | M20 |
| Расходы | 6 групп | M17 |
| Отчёты, панель, аудит | 10 | M16, M18, M22 |

Тест матрицы прав (NFR-032) обходит **все** маршруты из OpenAPI для ролей OWNER, STAFF и STAFF с каждым делегированным правом и сравнивает коды с таблицами этого раздела.
