# Проектирование базы данных

**Фаза 3 · Grocery ERP** · 27.11.2026 · статус: черновик на утверждение

СУБД — PostgreSQL 16. Схема создаётся только миграциями Flyway (`backend/src/main/resources/db/migration`). ER-диаграмма — [`diagrams/er.md`](diagrams/er.md). Алгоритмы, которые опираются на эти таблицы, — [`ALGORITHMS.md`](ALGORITHMS.md).

---

## 1. Общие соглашения

| Тема | Решение | Обоснование |
|---|---|---|
| Имена | `snake_case`, таблицы в единственном числе (`product`, `sale_item`) | Совпадает с соглашениями Hibernate `CamelCaseToUnderscoresNamingStrategy` |
| Первичные ключи | `bigint GENERATED ALWAYS AS IDENTITY` | Компактные индексы и составные FK; перебор id бесполезен, потому что каждый запрос фильтруется по `store_id` (ADR-0007) |
| Номера документов | Отдельное поле `number` (`integer`), сквозное в пределах магазина и типа документа | Пользователь видит «Чек № 152», а не внутренний id |
| Изоляция магазинов | Колонка `store_id NOT NULL` в каждой бизнес-таблице; `UNIQUE (store_id, id)` в каждой родительской таблице; **составные внешние ключи** `(store_id, x_id) → x(store_id, id)` | БД физически не даёт сослаться на объект другого магазина (DIR-009) |
| Деньги | `numeric(14,2)` для сумм и цен; `numeric(14,4)` для себестоимости единицы | BR-005, BR-070, BR-071; запас до 999 999 999 999,99 для агрегатов |
| Количества | `numeric(12,3)` | 1 г / 1 мл (BR-061); до 999 999 999,999 |
| Проценты | `numeric(5,2)` (0,00–100,00) | Скидки, лимиты |
| Время | `timestamptz` (UTC) для моментов, `date` для бизнес-дат и сроков годности | BR-180, BR-181, DIR-010 |
| Бизнес-дата | Колонка `business_date date` в документах, вычисляется сервером как дата в Asia/Bishkek на момент проведения | Отчёты группируют по ней без пересчёта часовых поясов (AC-REP-06) |
| Перечисления | `text` + `CHECK (col IN (...))` | Проще миграций `ENUM`; Hibernate хранит `@Enumerated(STRING)` |
| Блокировка версий | `version bigint` в изменяемых справочниках (`@Version`) | Оптимистическая защита от «потерянных обновлений» в формах |
| Аудит полей | `created_at`, `created_by`, `updated_at`, `updated_by` в справочниках | Кто и когда изменил; подробная история — в `audit_log` |
| Удаление | Физически удаляются только черновики документов и истёкшие технические записи; остальное архивируется (раздел 7) | BR-112 |

Расширения: `pg_trgm` (поиск по части названия, NFR-003), `citext` не нужен (логин хранится в нижнем регистре).

---

## 2. Модули и таблицы

| Модуль | Таблицы |
|---|---|
| Stores | `store`, `store_settings`, `document_sequence` |
| Users | `app_user`, `user_permission` |
| Authentication | `refresh_token` |
| Catalog | `category`, `product`, `product_barcode`, `product_price_history` |
| Suppliers | `supplier`, `product_supplier` |
| Purchases | `purchase`, `purchase_item`, `purchase_order`, `purchase_order_item` |
| Inventory | `product_stock`, `batch`, `stock_movement`, `write_off`, `write_off_item`, `stock_count`, `stock_count_item` |
| Sales | `sale`, `sale_item`, `sale_batch_allocation` |
| Returns | `customer_return`, `return_item`, `return_allocation` |
| Expenses | `expense_category`, `expense`, `recurring_expense` |
| Expiration | `batch_risk_event`, `expiry_action` |
| Платформа (shared) | `idempotency_key`, `audit_log`, `job_run` |

Finance, Replenishment и Reporting **не владеют таблицами** с бизнес-данными: они вычисляют результаты из документов (раздел 9). Единственное, что сохраняет пополнение, — снимок рекомендации в строке заказа (`purchase_order_item.recommended_qty`, `explanation`).

---

## 3. Таблицы

Ниже для краткости опущены общие колонки `created_at`, `created_by`, `updated_at`, `updated_by`, `version` там, где они есть по правилу раздела 1. «FK*» — составной внешний ключ `(store_id, col)`.

### 3.1. Stores

**`store`** — арендатор.

| Колонка | Тип | Ограничения |
|---|---|---|
| `id` | bigint | PK |
| `name` | text | NOT NULL, длина 1–120 |
| `address` | text | ≤ 300 |
| `phone` | text | ≤ 30 |
| `currency` | char(3) | NOT NULL DEFAULT 'KGS', `CHECK (currency = 'KGS')` (FR-012) |
| `time_zone` | text | NOT NULL DEFAULT 'Asia/Bishkek', `CHECK (time_zone = 'Asia/Bishkek')` |
| `status` | text | `ACTIVE`, `SUSPENDED` |
| `created_at` | timestamptz | NOT NULL |

**`store_settings`** — 1 : 1 с `store` (PK = `store_id`).

| Колонка | Тип | По умолчанию | CHECK |
|---|---|---|---|
| `block_on_expiry_day` | boolean | false | — (BR-020, D-005) |
| `expiry_t1_days`, `expiry_t2_days`, `expiry_t3_days` | smallint | 3, 7, 14 | `1 ≤ t1 < t2 < t3 ≤ 60` |
| `risk_window_days` | smallint | 28 | 7–90 (BR-161) |
| `staff_discount_limit_pct` | numeric(5,2) | 10,00 | 0–50 (BR-081) |
| `return_window_days` | smallint | 14 | 0–30 (BR-092) |
| `default_lead_time_days` | smallint | 2 | 0–60 (BR-130) |
| `default_order_cycle_days` | smallint | 7 | 1–30 (BR-133) |
| `default_safety_days` | smallint | 2 | 0–30 (BR-141) |
| `overdue_order_ignore_days` | smallint | 14 | 1–60 (BR-152) |

Поле «язык магазина» удалено (D-026).

**`document_sequence`** — `(store_id, doc_type)` PK, `next_value integer NOT NULL`. Номер берётся `UPDATE ... SET next_value = next_value + 1 RETURNING next_value - 1` в конце транзакции документа (короткая блокировка строки). Типы: `SALE`, `RETURN`, `PURCHASE`, `PURCHASE_ORDER`, `WRITE_OFF`, `STOCK_COUNT`, `BATCH`.

### 3.2. Users и Authentication

**`app_user`**

| Колонка | Тип | Ограничения |
|---|---|---|
| `id` | bigint | PK; `UNIQUE (store_id, id)` |
| `store_id` | bigint | FK → store, NOT NULL (BR-124) |
| `login` | text | NOT NULL, `UNIQUE (login)` во всей системе (BR-123), `CHECK (login = lower(login))`, 3–40 символов `[a-z0-9._-]` |
| `full_name` | text | NOT NULL, 1–100 |
| `password_hash` | text | NOT NULL (Argon2id, формат PHC) |
| `role` | text | `OWNER`, `STAFF` |
| `active` | boolean | NOT NULL DEFAULT true |
| `token_version` | integer | NOT NULL DEFAULT 0; увеличивается при деактивации, смене и сбросе пароля, изменении прав (FR-006) |
| `must_change_password` | boolean | true для временного пароля |
| `failed_login_count` | smallint | для блокировки (FR-003) |
| `locked_until` | timestamptz | NULL |
| `last_login_at` | timestamptz | FR-017 |

Индекс: частичный `(store_id) WHERE active` для подсчёта лимита 10 учётных записей (FR-016). Правило «в магазине есть хотя бы один активный OWNER» (FR-018) проверяется в сервисе под блокировкой строки `store`.

Колонка языка пользователя удалена (D-026).

**`user_permission`** — `(store_id, user_id, permission)` PK; `permission IN ('purchase.receive','stock.count','stock.writeoff')`; `granted_by`, `granted_at`. FK* на `app_user`.

**`refresh_token`**

| Колонка | Тип | Смысл |
|---|---|---|
| `id` | bigint | PK |
| `user_id` | bigint | FK → app_user |
| `family_id` | uuid | Цепочка ротации одного входа |
| `token_hash` | bytea | SHA-256 от случайного токена (сам токен не хранится), `UNIQUE` |
| `issued_at`, `last_used_at` | timestamptz | Для бездействия 8 ч (FR-005) |
| `family_expires_at` | timestamptz | Вход + 24 ч, абсолютный срок |
| `replaced_by_id` | bigint | Следующий токен цепочки |
| `revoked_at`, `revoke_reason` | | `LOGOUT`, `ROTATED`, `REUSE_DETECTED`, `USER_CHANGED` |
| `user_agent`, `ip` | text, inet | Для журнала |

Индексы: `(user_id) WHERE revoked_at IS NULL`, `(family_id)`.

### 3.3. Catalog

**`category`** — `id`, `store_id`, `parent_id` (FK*, NULL для корня), `name` (1–80), `active`. `UNIQUE (store_id, parent_id, lower(name))` через выражение и `COALESCE(parent_id, 0)`. Глубина ≤ 2 (FR-031) проверяется сервисом: у родителя не может быть своего родителя.

**`product`**

| Колонка | Тип | Ограничения / смысл |
|---|---|---|
| `id`, `store_id` | bigint | PK; `UNIQUE (store_id, id)` |
| `code` | integer | Внутренний код, `UNIQUE (store_id, code)` (FR-026) |
| `name` | text | NOT NULL, 1–200 (было `name_ru` / `name_ky`, D-026) |
| `category_id` | bigint | FK*, NULL допустим («Без категории») |
| `unit` | text | `PCS`, `KG`, `L`; не меняется после первого движения (FR-030) |
| `retail_price` | numeric(14,2) | `CHECK (retail_price >= 0 AND retail_price <= 1000000)` (BR-074) |
| `track_expiry` | boolean | DEFAULT true |
| `shelf_life_days` | smallint | Срок годности по умолчанию; `CHECK (track_expiry OR shelf_life_days IS NULL)`, `CHECK (shelf_life_days BETWEEN 1 AND 3650)` |
| `preferred_supplier_id` | bigint | FK* → supplier, NULL (BR-132) |
| `safety_days` | smallint | NULL → значение магазина; 0–30 (BR-141) |
| `order_multiple` | numeric(12,3) | NULL → 1; `> 0` |
| `min_order_qty` | numeric(12,3) | NULL → 0; `>= 0` |
| `active` | boolean | Архив = false (FR-023) |

Индексы: `GIN (lower(name) gin_trgm_ops)` для поиска; `(store_id, category_id) WHERE active`; `(store_id, active)`.

**`product_barcode`** — `id`, `store_id`, `product_id` (FK*), `barcode` (text, 1–32). Уникальность среди **активных** товаров магазина (FR-034): колонка `product_active boolean` поддерживается сервисом при архивировании, индекс `UNIQUE (store_id, barcode) WHERE product_active`. Архивный товар освобождает штрихкод.

**`product_price_history`** — `product_id`, `old_price`, `new_price`, `changed_by`, `changed_at` (FR-029). Только добавление.

### 3.4. Suppliers

**`supplier`** — `id`, `store_id`, `name` (1–120, `UNIQUE (store_id, lower(name)) WHERE active`), `contact_person`, `phone`, `note`, `lead_time_days` (smallint, 0–60, NULL → магазин), `order_cycle_days` (smallint, 1–30, NULL → магазин), `active`.

**`product_supplier`** — связь многие-ко-многим (FR-044).

| Колонка | Тип | Смысл |
|---|---|---|
| `store_id`, `product_id`, `supplier_id` | bigint | PK `(store_id, product_id, supplier_id)`; FK* на product и supplier |
| `lead_time_days` | smallint | Переопределение срока поставки (BR-131), NULL → поставщик |
| `last_purchase_price` | numeric(14,4) | Обновляется при проведении приёмки |
| `last_purchase_at` | timestamptz | Для правила «поставщик последней приёмки» (BR-132) |

### 3.5. Purchases

**`purchase`** — приёмка (накладная поставщика).

| Колонка | Тип | Ограничения |
|---|---|---|
| `id`, `store_id`, `number` | | `UNIQUE (store_id, number)` после проведения (NULL у черновика) |
| `supplier_id` | bigint | FK*, NULL допустим («Без поставщика») |
| `purchase_order_id` | bigint | FK*, NULL |
| `invoice_number` | text | ≤ 50 |
| `invoice_date` | date | |
| `status` | text | `DRAFT`, `POSTED`, `CANCELLED` |
| `payment_status` | text | `UNPAID`, `PAID` (FR-046) |
| `total_amount` | numeric(14,2) | Σ строк, фиксируется при проведении |
| `business_date` | date | Дата проведения в Asia/Bishkek |
| `posted_at`, `posted_by`, `cancelled_at`, `cancelled_by` | | |

**`purchase_item`** — `id`, `store_id`, `purchase_id` (FK*, ON DELETE CASCADE только для DRAFT — удаление черновика), `line_no`, `product_id` (FK*), `quantity` (`> 0`), `unit_cost` numeric(14,4) (`>= 0`), `line_amount` numeric(14,2), `expiry_date`, `production_date` (`CHECK (production_date IS NULL OR expiry_date IS NULL OR production_date <= expiry_date)`), `purchase_order_item_id` (FK*, NULL), `batch_id` (FK*, заполняется при проведении), `product_name`, `unit` (снимок, BR-111).

**`purchase_order`** — заказ поставщику (FR-057, R2). Нужен, потому что ожидаемые поставки (BR-150) входят в формулу пополнения как `E`.

| Колонка | Тип | Ограничения |
|---|---|---|
| `id`, `store_id`, `number` | | |
| `supplier_id` | bigint | FK*, NOT NULL |
| `status` | text | `DRAFT`, `ORDERED`, `PARTIALLY_RECEIVED`, `RECEIVED`, `CANCELLED` |
| `expected_date` | date | NOT NULL для статуса ORDERED и далее (`CHECK (status = 'DRAFT' OR expected_date IS NOT NULL)`) |
| `ordered_at` | timestamptz | |

Индекс: `(store_id, status, expected_date) WHERE status IN ('ORDERED','PARTIALLY_RECEIVED')`.

**`purchase_order_item`** — `id`, `store_id`, `purchase_order_id`, `product_id`, `ordered_qty` (`> 0`), `received_qty` (`>= 0`, `CHECK (received_qty <= ordered_qty)`), `cancelled_qty`, `recommended_qty` (снимок рекомендации, FR-194), `explanation jsonb` (снимок объяснения).

### 3.6. Inventory

**`product_stock`** — состояние запаса и средней себестоимости товара; 1 : 1 с `product` (PK `(store_id, product_id)`). Строка — объект блокировки (ALGORITHMS, раздел 7).

| Колонка | Тип | Ограничения |
|---|---|---|
| `qty_on_hand` | numeric(12,3) | `CHECK (qty_on_hand >= 0)` |
| `avg_cost` | numeric(14,4) | NULL до первого прихода, `CHECK (avg_cost >= 0)` |
| `last_purchase_price` | numeric(14,4) | |
| `first_receipt_date` | date | Для N в формуле D (BR-161) |
| `has_movements` | boolean | Блокирует смену единицы (FR-030) |

**`batch`** — партия.

| Колонка | Тип | Ограничения |
|---|---|---|
| `id`, `store_id`, `number` | | `UNIQUE (store_id, number)` — короткий номер (FR-065) |
| `product_id` | bigint | FK* |
| `source_type` | text | `PURCHASE`, `STOCK_COUNT`, `OPENING_BALANCE` (импорт) |
| `source_id` | bigint | id строки-источника |
| `received_at` | timestamptz | Для сортировки FEFO (BR-030) |
| `expiry_date` | date | NULL для товаров без учёта сроков (FR-067) |
| `production_date` | date | |
| `initial_qty` | numeric(12,3) | `> 0` |
| `qty_on_hand` | numeric(12,3) | `CHECK (qty_on_hand >= 0)` (BR-013). Верхней границы нет: после возвратов и отмены списаний остаток может превысить `initial_qty` |
| `unit_cost` | numeric(14,4) | Информационная (BR-006) |
| `status` | text | `ACTIVE`, `BLOCKED`, `DEPLETED` |
| `markdown_price` | numeric(14,2) | NULL; уценка (FR-064, BR-084), `CHECK (markdown_price > 0)` |
| `markdown_set_by`, `markdown_set_at` | | |

Индексы: **FEFO** — `(store_id, product_id, expiry_date NULLS LAST, received_at, id) WHERE qty_on_hand > 0 AND status = 'ACTIVE'`; **сроки** — `(store_id, expiry_date) WHERE qty_on_hand > 0`.

**`stock_movement`** — журнал движений, только добавление.

| Колонка | Тип | Ограничения |
|---|---|---|
| `id`, `store_id` | bigint | |
| `product_id`, `batch_id` | bigint | FK* |
| `type` | text | 8 типов (ALGORITHMS, 4.1) |
| `qty_delta` | numeric(12,3) | `CHECK (qty_delta <> 0)`; знак согласован с типом: `CHECK ((type IN ('RECEIPT','CUSTOMER_RETURN','WRITE_OFF_REVERSAL','ADJUSTMENT_PLUS') AND qty_delta > 0) OR (type IN ('RECEIPT_REVERSAL','SALE','WRITE_OFF','ADJUSTMENT_MINUS') AND qty_delta < 0))` |
| `unit_cost` | numeric(14,4) | NOT NULL |
| `cost_amount` | numeric(14,2) | `round2(|qty_delta| × unit_cost)`, со знаком движения |
| `avg_cost_after` | numeric(14,4) | |
| `qty_after_product` | numeric(12,3) | Остаток товара после движения (для отчёта «остатки на дату», FR-077) |
| `doc_type`, `doc_id`, `doc_line_id` | text, bigint, bigint | Ссылка на документ |
| `business_date` | date | |
| `created_at`, `created_by` | | |

Индексы: `(store_id, product_id, created_at)`, `(store_id, batch_id)`, `(store_id, business_date, type)`, `(store_id, doc_type, doc_id)`.

**`write_off`** — `id`, `store_id`, `number`, `kind` (`REGULAR`, `BULK_EXPIRED`, `RETURN_DAMAGED`, `REVERSAL`), `reversed_write_off_id` (FK*, для сторно; `UNIQUE` — отменить можно один раз), `status` (`POSTED`, `REVERSED`), `total_cost` numeric(14,2), `business_date`, `posted_at`, `posted_by`, `customer_return_id` (для RETURN_DAMAGED).

**`write_off_item`** — `id`, `store_id`, `write_off_id`, `batch_id`, `product_id`, `quantity` (`> 0`), `reason` (`EXPIRED`, `DAMAGED`, `SPOILED`, `SHORTAGE`, `RETURN_TO_SUPPLIER`, `RETURN_DAMAGED`, `OTHER`), `comment` (`CHECK (reason NOT IN ('OTHER','SHORTAGE') OR length(trim(comment)) >= 3)`, BR-105), `unit_cost`, `cost_amount`, `product_name` (снимок).

**`stock_count`** — `id`, `store_id`, `number`, `scope_type` (`ALL`, `CATEGORY`, `LIST`, `QUICK`), `scope_category_id`, `status` (`DRAFT`, `POSTED`), `posted_at`, `posted_by`, `business_date`, `shortage_cost`, `surplus_cost`.

**`stock_count_item`** — `id`, `store_id`, `stock_count_id`, `product_id`, `batch_id` (NULL = без партии), `expected_qty_at_entry`, `counted_qty` (`>= 0`), `qty_at_posting`, `diff_qty`, `reason` (`RECOUNT`, `SHORTAGE`, `SURPLUS`, `INPUT_ERROR`, `OTHER`; обязательна при `diff_qty <> 0`, BR-202), `new_batch_expiry_date` (для излишка без партии, BR-201), `owner_unit_cost` (для излишка при `avg_cost IS NULL`), `cost_amount`.

### 3.7. Sales

**`sale`**

| Колонка | Тип | Ограничения |
|---|---|---|
| `id`, `store_id`, `number` | | `UNIQUE (store_id, number)` |
| `idempotency_key` | uuid | `UNIQUE (store_id, idempotency_key)` |
| `cashier_id` | bigint | FK* → app_user |
| `payment_method` | text | `CASH`, `CARD`, `TRANSFER` (FR-094). Один способ на чек: смешанная оплата не требуется в MVP |
| `gross_amount`, `discount_amount`, `total_amount` | numeric(14,2) | `CHECK (total_amount >= 0 AND total_amount <= 10000000)`; `CHECK (total_amount = gross_amount - discount_amount)` |
| `receipt_discount_pct` | numeric(5,2) | NULL |
| `cash_received`, `change_amount` | numeric(14,2) | Только для CASH; `CHECK (payment_method <> 'CASH' OR cash_received >= total_amount)` |
| `cost_amount` | numeric(14,2) | Σ себестоимости строк |
| `business_date` | date | NOT NULL |
| `posted_at` | timestamptz | NOT NULL |

Статуса у продажи нет: она существует только проведённой. Отмены продажи нет — исправление возвратом (BR-110). Индексы: `(store_id, business_date)`, `(store_id, cashier_id, business_date)`.

**`sale_item`**

| Колонка | Тип | Смысл |
|---|---|---|
| `id`, `store_id`, `sale_id`, `line_no` | | |
| `product_id` | bigint | FK* |
| `product_name`, `unit` | text | Снимок (BR-111) |
| `quantity` | numeric(12,3) | `> 0`, `<= 9999` |
| `base_price` | numeric(14,2) | Розничная цена на момент продажи |
| `gross_amount` | numeric(14,2) | Σ по распределениям `round2(qty × price)` |
| `line_discount_type` | text | `PERCENT`, `AMOUNT`, NULL |
| `line_discount_value` | numeric(14,2) | |
| `line_discount_amount` | numeric(14,2) | `>= 0` |
| `receipt_discount_amount` | numeric(14,2) | Доля скидки чека (BR-082) |
| `total_amount` | numeric(14,2) | `CHECK (total_amount >= 0)`, `CHECK (total_amount = gross_amount - line_discount_amount - receipt_discount_amount)` |
| `discount_applied_by` | bigint | Кто дал ручную скидку (BR-086) |
| `unit_cost` | numeric(14,4) | avg на момент продажи (BR-007) |
| `cost_amount` | numeric(14,2) | `round2(quantity × unit_cost)` |
| `returned_qty` | numeric(12,3) | `CHECK (returned_qty >= 0 AND returned_qty <= quantity)` (BR-091) |
| `returned_amount` | numeric(14,2) | Σ сумм возвратов по строке |

Индекс для D: `(store_id, product_id)` + `sale.business_date` через join; при росте объёма — денормализованная `business_date` в `sale_item` (уже предусмотрена колонкой).

**`sale_batch_allocation`** — физическое распределение строки по партиям (BR-031).

| Колонка | Тип | Ограничения |
|---|---|---|
| `id`, `store_id`, `sale_item_id`, `batch_id` | bigint | FK* |
| `quantity` | numeric(12,3) | `> 0` |
| `unit_price` | numeric(14,2) | Розничная или уценка партии |
| `amount` | numeric(14,2) | `round2(quantity × unit_price)` |
| `manual` | boolean | Ручной выбор партии (BR-033) |
| `returned_qty` | numeric(12,3) | `CHECK (returned_qty >= 0 AND returned_qty <= quantity)` |

Сумма `quantity` распределений = `sale_item.quantity` — проверяется сервисом и тестом-инвариантом (CHECK между строками в PostgreSQL невозможен без триггера; триггер сознательно не используется, чтобы логика оставалась в одном месте, ADR-0006).

### 3.8. Returns

**`customer_return`** — `id`, `store_id`, `number`, `sale_id` (FK*), `idempotency_key` (`UNIQUE (store_id, idempotency_key)`), `refund_amount`, `cost_amount`, `refund_method` (= способ оплаты продажи), `business_date`, `posted_at`, `posted_by`, `reason_comment`.

**`return_item`** — `id`, `store_id`, `customer_return_id`, `sale_item_id` (FK*), `quantity` (`> 0`), `condition` (`SALEABLE`, `DAMAGED`), `forced_damaged` (истёкшая партия, BR-094), `refund_amount`, `unit_cost` (= `sale_item.unit_cost`), `cost_amount`.

**`return_allocation`** — `id`, `store_id`, `return_item_id`, `sale_batch_allocation_id`, `batch_id`, `quantity`. Сохраняет, в какую партию вернулся товар (BR-035).

Название `customer_return`, а не `return`: `RETURN` — ключевое слово SQL и Java.

### 3.9. Expenses

**`expense_category`** — `id`, `store_id`, `name`, `system` (стандартные категории создаются при регистрации магазина и не удаляются, AC-EXPN-05), `active`. `UNIQUE (store_id, lower(name))`.

**`expense`** — `id`, `store_id`, `expense_date` (date), `category_id` (FK*), `amount` (`CHECK (amount > 0 AND amount <= 10000000)`), `payment_method` (`CASH`, `CARD`, `TRANSFER`, `OTHER`), `comment`, `recurring_expense_id`, `deleted_at`, `deleted_by`. Расход — изменяемая запись (BR-114): изменения и удаление пишутся в аудит, удаление мягкое (`deleted_at`), чтобы история сохранялась.

**`recurring_expense`** — шаблон (FR-166): `category_id`, `amount`, `day_of_month` (1–28), `active`, `last_generated_for` (date). Индекс уникальности `(store_id, recurring_expense_id, date_trunc('month', expense_date))` на `expense` не даёт ночной задаче создать дубль.

### 3.10. Expiration

**`batch_risk_event`** — `id`, `store_id`, `batch_id`, `level` (`CRITICAL`, `HIGH`), `risk_qty`, `unit_cost_at_event`, `event_date`. Пишется ночным пересчётом при повышении уровня; нужен для «сохранено от просрочки» (BR-167). `UNIQUE (store_id, batch_id, level)`.

**`expiry_action`** — отметка выполнения действия центра действий (FR-143): `id`, `store_id`, `batch_id`, `action` (`MARKDOWN`, `MOVE_FRONT`, `RETURN_TO_SUPPLIER`, `WRITE_OFF`), `level_at_action`, `linked_doc_type`, `linked_doc_id`, `done_by`, `done_at`.

Текущий уровень риска **не хранится**: он вычисляется по запросу за ≤ 30 с для 10 000 партий (NFR-007), а ночная задача только фиксирует события. Так уровень не может «устареть».

### 3.11. Платформа

**`idempotency_key`** (BR-050–BR-054, ALGORITHMS, раздел 8)

| Колонка | Тип | Ограничения |
|---|---|---|
| `store_id`, `key` | bigint, uuid | PK `(store_id, key)` |
| `operation` | text | `SALE`, `RETURN`, `PURCHASE_POST`, `WRITE_OFF`, `STOCK_COUNT_POST`, `PURCHASE_CANCEL`, `WRITE_OFF_REVERSAL` |
| `request_hash` | bytea | SHA-256 |
| `document_type`, `document_id` | text, bigint | Результат |
| `created_by`, `created_at` | | |

Хранится бессрочно (BR-053): ~300 строк в день на магазин, около 3 МБ в год.

**`audit_log`** (FR-211–FR-214)

| Колонка | Тип | Смысл |
|---|---|---|
| `id` | bigint | PK |
| `store_id` | bigint | NULL только для событий до определения магазина (неудачный вход с неизвестным логином) |
| `user_id` | bigint | NULL для анонимных событий |
| `event_type` | text | `LOGIN_SUCCESS`, `LOGIN_FAILED`, `PRICE_CHANGED`, `MANUAL_DISCOUNT`, `MANUAL_BATCH`, `RETURN_POSTED`, `WRITE_OFF_POSTED`, `STOCK_ADJUSTED`, `DOCUMENT_CANCELLED`, `PERMISSION_CHANGED`, `USER_STATUS_CHANGED`, `EXPENSE_CHANGED`, `ACCESS_DENIED`, … |
| `entity_type`, `entity_id` | text, bigint | |
| `old_value`, `new_value` | jsonb | Без паролей и токенов |
| `ip` | inet | |
| `request_id` | uuid | Связь с журналом приложения |
| `created_at` | timestamptz | |

Индексы: `(store_id, created_at DESC)`, `(store_id, user_id, created_at DESC)`, `(store_id, event_type, created_at DESC)`. Приложение имеет на таблицу только `INSERT, SELECT` (раздел 6). При росте — партиционирование по месяцам (не нужно в MVP).

**`job_run`** — `job_name`, `store_id`, `run_date` PK; отметка выполненных ночных задач, чтобы повторный запуск после рестарта не делал работу дважды.

---

## 4. Размещение спорных полей

| Поле | Где | Почему не в другом месте |
|---|---|---|
| `shelf_life_days` | `product` | Свойство товара; используется для подсказки срока при приёмке, ограничения пополнения (BR-175) и `safety_days ≤ shelf/2` (BR-142). У партии свой фактический `expiry_date` |
| `order_period_days` (R) | `supplier.order_cycle_days` (+ `store_settings.default_order_cycle_days`) | BR-133 определяет R у поставщика: это его график визитов. Переопределение на товар не требуется |
| `default_supplier_id` | `product.preferred_supplier_id` | Один предпочтительный поставщик на товар (FR-028, BR-132); флаг в `product_supplier` потребовал бы частичного уникального индекса и не проще |
| `safety_stock_qty` | **Не хранится** | По D-012 страховой запас задаётся в днях (`product.safety_days`), а `SS = D × safety_days` вычисляется. Хранимое количество устаревало бы при изменении спроса. Значение SS сохраняется только в снимке объяснения заказа |
| `lead_time_days` | `supplier.lead_time_days`, переопределение `product_supplier.lead_time_days`, по умолчанию `store_settings` | Точно по BR-130, BR-131 |
| `min_order_qty` | `product.min_order_qty` (вместе с `order_multiple`) | FR-028 относит их к товару. Если интервью покажут, что фасовка зависит от поставщика, поля переносятся в `product_supplier` миграцией |
| `payment_method` | `sale`, `customer_return.refund_method`, `expense` | Один способ на чек; у приёмки вместо способа — `payment_status` (FR-046) |
| `expected_delivery_date` | `purchase_order.expected_date` | Ожидаемая дата — свойство заказа (BR-150). У приёмки дата фактическая (`posted_at`) |

---

## 5. Ограничения целостности: сводка

| Правило | Механизм |
|---|---|
| Остаток ≥ 0 (BR-010, BR-013) | `CHECK` на `batch.qty_on_hand`, `product_stock.qty_on_hand` |
| Знак движения по типу | `CHECK` на `stock_movement` |
| Возвращено ≤ продано (BR-091) | `CHECK` на `sale_item.returned_qty`, `sale_batch_allocation.returned_qty` |
| Итог строки ≥ 0 и арифметика итогов (BR-083, BR-073) | `CHECK` на `sale_item`, `sale` |
| Уникальность штрихкода среди активных (FR-034) | Частичный `UNIQUE` |
| Уникальность ключа идемпотентности (DIR-007) | PK `idempotency_key`, `UNIQUE` в документах |
| Ссылки внутри одного магазина (DIR-009) | Составные FK |
| Нельзя удалить справочник со ссылками (DIR-008) | FK без `ON DELETE CASCADE` (кроме строк черновиков) |
| Валюта и часовой пояс фиксированы (FR-012) | `CHECK` на `store` |
| Лимиты сумм (BR-074) | `CHECK` на ценах и суммах |
| Неизменяемость журналов | Права БД (раздел 6) |

---

## 6. Роли базы данных

| Роль | Права | Кто использует |
|---|---|---|
| `grocery_owner` | Владелец схемы, DDL | Только Flyway при развёртывании |
| `grocery_app` | `SELECT, INSERT, UPDATE, DELETE` на рабочие таблицы; **`SELECT, INSERT` без `UPDATE, DELETE`** на `audit_log`, `stock_movement`, `product_price_history`, `idempotency_key`, `batch_risk_event` | Backend |
| `grocery_backup` | `SELECT` на всё, `pg_dump` | Контейнер резервного копирования |
| `grocery_readonly` | `SELECT` | Ручная диагностика оператором |

Миграции Flyway выполняются под `grocery_owner` (отдельный URL в переменных окружения), приложение подключается как `grocery_app`. Это выполняет AC-AUD-02: у приложения нет прав изменить журнал.

---

## 7. История и архивирование

| Данные | Политика |
|---|---|
| Проведённые документы и их строки | Неизменяемы, не удаляются (BR-110). Исправление — новым документом |
| Черновики (`purchase`, `stock_count`, `purchase_order` в DRAFT) | Удаляются физически пользователем; строки — каскадно |
| Справочники (товар, категория, поставщик, пользователь) | Архивируются флагом `active = false` (BR-112, BR-115). Документы хранят снимок названия (BR-111) |
| Расходы | Мягкое удаление `deleted_at` + аудит (BR-114) |
| `stock_movement`, `audit_log` | Только добавление; хранение не менее 1 года (FR-213), в MVP — бессрочно |
| `refresh_token` | Удаляются ночной задачей через 30 дней после истечения цепочки |
| `idempotency_key` | Бессрочно (BR-053) |
| Магазин | Удаление магазина вне MVP; оператор может перевести в `SUSPENDED` |

**Персональные данные** минимальны: имя и логин сотрудника, телефон контактного лица поставщика, IP в журнале (SEC-011). Данные покупателей не собираются.

---

## 8. Объём данных и индексы

Оценка для одного магазина за 3 года (NFR-060): 5 000 товаров, 60 000 партий, 330 000 чеков, ~1,6 млн строк продаж, ~1,8 млн движений, ~0,5 млн записей аудита. Для 10 магазинов — около 40 млн строк, 15–20 ГБ с индексами. Это укладывается в один сервер 2 vCPU / 4 ГБ при индексах выше. Партиционирование не нужно; при необходимости первыми партиционируются `stock_movement` и `audit_log` по `created_at`.

Критичные запросы и их индексы:

| Запрос | Индекс |
|---|---|
| Касса: товар по штрихкоду (NFR-001) | `UNIQUE (store_id, barcode) WHERE product_active` |
| Поиск по названию (NFR-003) | `GIN (lower(name) gin_trgm_ops)` + фильтр `store_id` |
| FEFO-кандидаты | `(store_id, product_id, expiry_date NULLS LAST, received_at, id) WHERE qty_on_hand > 0 AND status = 'ACTIVE'` |
| Списки сроков | `(store_id, expiry_date) WHERE qty_on_hand > 0` |
| D за 28 дней | `sale_item (store_id, product_id, business_date)`, `return_item` аналогично |
| Отчёт за месяц (NFR-004) | `sale (store_id, business_date)` |
| Ожидаемые поставки | частичный индекс на `purchase_order` |

---

## 9. Что вычисляется, а не хранится

| Значение | Где вычисляется | Почему не хранится |
|---|---|---|
| Уровень риска партии | Expiration | Зависит от даты и спроса; хранимое значение устаревает |
| D (среднесуточные продажи) | Sales (`SalesStatistics`) | Простая агрегация за 28 дней |
| Рекомендация пополнения | Replenishment | Совет, а не документ (BR-178) |
| Прибыль, P&L | Finance | Сумма зафиксированных в документах значений |
| «Истёкшая» партия | Inventory | Сравнение даты, а не статус |
| Стоимость склада | Finance | `Σ product_stock.qty_on_hand × avg_cost` |
