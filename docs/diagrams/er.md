# ER-диаграмма

Основные сущности и связи. Типы упрощены для читаемости: `money` = `numeric(14,2)`, `cost` = `numeric(14,4)`, `qty` = `numeric(12,3)`. Почти у каждой таблицы есть `store_id` и составной внешний ключ `(store_id, x_id)`; на диаграмме `store_id` показан только у корневых сущностей. Полное описание колонок и ограничений — [`../DATABASE.md`](../DATABASE.md).

## Справочники, пользователи, закупки

```mermaid
erDiagram
    STORE ||--|| STORE_SETTINGS : "настройки"
    STORE ||--o{ APP_USER : "сотрудники"
    APP_USER ||--o{ USER_PERMISSION : "делегированные права"
    APP_USER ||--o{ REFRESH_TOKEN : "сеансы"
    STORE ||--o{ CATEGORY : ""
    CATEGORY |o--o{ CATEGORY : "родитель"
    CATEGORY |o--o{ PRODUCT : ""
    PRODUCT ||--o{ PRODUCT_BARCODE : ""
    PRODUCT ||--|| PRODUCT_STOCK : "остаток и avg"
    STORE ||--o{ SUPPLIER : ""
    SUPPLIER |o--o{ PRODUCT : "предпочтительный"
    PRODUCT ||--o{ PRODUCT_SUPPLIER : ""
    SUPPLIER ||--o{ PRODUCT_SUPPLIER : ""
    SUPPLIER |o--o{ PURCHASE : ""
    SUPPLIER ||--o{ PURCHASE_ORDER : ""
    PURCHASE_ORDER ||--|{ PURCHASE_ORDER_ITEM : ""
    PURCHASE_ORDER |o--o{ PURCHASE : "приёмка по заказу"
    PURCHASE ||--|{ PURCHASE_ITEM : ""
    PURCHASE_ORDER_ITEM |o--o{ PURCHASE_ITEM : ""
    PURCHASE_ITEM |o--|| BATCH : "создаёт при проведении"
    PRODUCT ||--o{ BATCH : ""

    STORE {
        bigint id PK
        text name
        char currency "KGS"
        text time_zone "Asia/Bishkek"
    }
    STORE_SETTINGS {
        bigint store_id PK, FK
        boolean block_on_expiry_day
        smallint expiry_t1_days
        numeric staff_discount_limit_pct
        smallint return_window_days
    }
    APP_USER {
        bigint id PK
        bigint store_id FK
        text login UK "уникален в системе"
        text password_hash "Argon2id"
        text role "OWNER или STAFF"
        boolean active
        integer token_version
    }
    USER_PERMISSION {
        bigint user_id PK, FK
        text permission PK
    }
    REFRESH_TOKEN {
        bigint id PK
        bigint user_id FK
        uuid family_id
        bytea token_hash UK
        timestamptz family_expires_at
        timestamptz revoked_at
    }
    CATEGORY {
        bigint id PK
        bigint parent_id FK
        text name
        boolean active
    }
    PRODUCT {
        bigint id PK
        bigint store_id FK
        integer code UK
        text name
        text unit "PCS KG L"
        money retail_price
        boolean track_expiry
        smallint shelf_life_days
        bigint preferred_supplier_id FK
        smallint safety_days
        qty order_multiple
        qty min_order_qty
        boolean active
    }
    PRODUCT_BARCODE {
        bigint id PK
        bigint product_id FK
        text barcode "UK среди активных"
    }
    PRODUCT_STOCK {
        bigint product_id PK, FK
        qty qty_on_hand "CHECK >= 0"
        cost avg_cost
        cost last_purchase_price
    }
    SUPPLIER {
        bigint id PK
        text name
        smallint lead_time_days
        smallint order_cycle_days
        boolean active
    }
    PRODUCT_SUPPLIER {
        bigint product_id PK, FK
        bigint supplier_id PK, FK
        smallint lead_time_days "переопределение"
        cost last_purchase_price
    }
    PURCHASE {
        bigint id PK
        integer number UK
        text status "DRAFT POSTED CANCELLED"
        text payment_status
        money total_amount
        date business_date
    }
    PURCHASE_ITEM {
        bigint id PK
        bigint product_id FK
        qty quantity
        cost unit_cost
        date expiry_date
        bigint batch_id FK
    }
    PURCHASE_ORDER {
        bigint id PK
        text status
        date expected_date
    }
    PURCHASE_ORDER_ITEM {
        bigint id PK
        bigint product_id FK
        qty ordered_qty
        qty received_qty
        qty recommended_qty
        jsonb explanation
    }
    BATCH {
        bigint id PK
        integer number UK
        bigint product_id FK
        date expiry_date
        timestamptz received_at
        qty qty_on_hand "CHECK >= 0"
        cost unit_cost "информационная"
        text status "ACTIVE BLOCKED DEPLETED"
        money markdown_price
    }
```

## Движения, продажи, возвраты, списания, расходы

```mermaid
erDiagram
    BATCH ||--o{ STOCK_MOVEMENT : ""
    PRODUCT ||--o{ STOCK_MOVEMENT : ""
    APP_USER ||--o{ SALE : "кассир"
    SALE ||--|{ SALE_ITEM : ""
    SALE_ITEM ||--|{ SALE_BATCH_ALLOCATION : "FEFO"
    BATCH ||--o{ SALE_BATCH_ALLOCATION : ""
    SALE ||--o{ CUSTOMER_RETURN : ""
    CUSTOMER_RETURN ||--|{ RETURN_ITEM : ""
    SALE_ITEM ||--o{ RETURN_ITEM : ""
    RETURN_ITEM ||--|{ RETURN_ALLOCATION : ""
    SALE_BATCH_ALLOCATION ||--o{ RETURN_ALLOCATION : "в ту же партию"
    WRITE_OFF ||--|{ WRITE_OFF_ITEM : ""
    BATCH ||--o{ WRITE_OFF_ITEM : ""
    WRITE_OFF |o--o| WRITE_OFF : "сторно"
    CUSTOMER_RETURN |o--o| WRITE_OFF : "RETURN_DAMAGED"
    STOCK_COUNT ||--|{ STOCK_COUNT_ITEM : ""
    BATCH |o--o{ STOCK_COUNT_ITEM : ""
    EXPENSE_CATEGORY ||--o{ EXPENSE : ""
    RECURRING_EXPENSE |o--o{ EXPENSE : "создаёт"
    BATCH ||--o{ BATCH_RISK_EVENT : ""
    BATCH ||--o{ EXPIRY_ACTION : ""

    STOCK_MOVEMENT {
        bigint id PK
        text type "8 типов"
        qty qty_delta "знак по типу"
        cost unit_cost
        money cost_amount
        cost avg_cost_after
        text doc_type
        bigint doc_id
        date business_date
    }
    SALE {
        bigint id PK
        integer number UK
        uuid idempotency_key UK
        text payment_method "CASH CARD TRANSFER"
        money gross_amount
        money discount_amount
        money total_amount
        money cost_amount
        date business_date
    }
    SALE_ITEM {
        bigint id PK
        bigint product_id FK
        text product_name "снимок"
        qty quantity
        money base_price
        money line_discount_amount
        money receipt_discount_amount
        money total_amount
        cost unit_cost "avg при продаже"
        money cost_amount
        qty returned_qty
    }
    SALE_BATCH_ALLOCATION {
        bigint id PK
        bigint batch_id FK
        qty quantity
        money unit_price
        boolean manual
        qty returned_qty
    }
    CUSTOMER_RETURN {
        bigint id PK
        integer number UK
        uuid idempotency_key UK
        money refund_amount
        money cost_amount
        date business_date
    }
    RETURN_ITEM {
        bigint id PK
        bigint sale_item_id FK
        qty quantity
        text condition "SALEABLE DAMAGED"
        money refund_amount
        cost unit_cost
    }
    RETURN_ALLOCATION {
        bigint id PK
        bigint batch_id FK
        qty quantity
    }
    WRITE_OFF {
        bigint id PK
        integer number UK
        text kind
        bigint reversed_write_off_id FK
        money total_cost
    }
    WRITE_OFF_ITEM {
        bigint id PK
        bigint batch_id FK
        qty quantity
        text reason
        cost unit_cost
        money cost_amount
    }
    STOCK_COUNT {
        bigint id PK
        text status
        text scope_type
    }
    STOCK_COUNT_ITEM {
        bigint id PK
        bigint batch_id FK
        qty counted_qty
        qty diff_qty
        text reason
    }
    EXPENSE_CATEGORY {
        bigint id PK
        text name
        boolean system
    }
    EXPENSE {
        bigint id PK
        date expense_date
        money amount
        text payment_method
        timestamptz deleted_at
    }
    RECURRING_EXPENSE {
        bigint id PK
        money amount
        smallint day_of_month
    }
    BATCH_RISK_EVENT {
        bigint id PK
        text level
        qty risk_qty
        date event_date
    }
    EXPIRY_ACTION {
        bigint id PK
        text action
        text linked_doc_type
        bigint linked_doc_id
    }
```

## Платформа

```mermaid
erDiagram
    STORE ||--o{ IDEMPOTENCY_KEY : ""
    STORE ||--o{ AUDIT_LOG : ""
    STORE ||--o{ DOCUMENT_SEQUENCE : ""
    IDEMPOTENCY_KEY {
        bigint store_id PK
        uuid key PK
        text operation
        bytea request_hash
        text document_type
        bigint document_id
    }
    AUDIT_LOG {
        bigint id PK
        bigint store_id FK
        bigint user_id FK
        text event_type
        jsonb old_value
        jsonb new_value
        timestamptz created_at
    }
    DOCUMENT_SEQUENCE {
        bigint store_id PK
        text doc_type PK
        integer next_value
    }
```
