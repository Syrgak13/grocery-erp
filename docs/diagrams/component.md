# Компоненты backend: модули и зависимости (C4, уровень 3)

Стрелка «A → B» означает: модуль A вызывает публичный `api` модуля B. Граф ацикличен; проверяется ArchUnit в CI. Модуль `shared` (деньги, ошибки, контекст магазина, время, идемпотентность, аудит, нумерация) используют все модули, на схеме его стрелки опущены.

```mermaid
flowchart TB
    subgraph web["REST-контроллеры /api/v1 (слой web каждого модуля)"]
    end

    auth["auth<br/>вход, токены, блокировка"]
    users["users<br/>сотрудники, права"]
    stores["stores<br/>магазин, настройки"]
    catalog["catalog<br/>товары, категории, штрихкоды"]
    suppliers["suppliers<br/>поставщики, сроки поставки"]
    inventory["inventory<br/>партии, остатки, avg, движения,<br/>FEFO, списания, инвентаризация"]
    purchases["purchases<br/>приёмки, заказы поставщику"]
    sales["sales<br/>касса, чек, скидки, статистика D"]
    returns["returns<br/>возвраты покупателей"]
    expenses["expenses<br/>расходы"]
    finance["finance<br/>валовая прибыль, P&L,<br/>стоимость склада"]
    expiration["expiration<br/>сроки, риск, центр действий"]
    replenishment["replenishment<br/>объяснимые рекомендации"]
    reporting["reporting<br/>отчёты, панели, экспорт, аудит"]
    shared["shared<br/>Money, Quantity, ошибки, CurrentUser,<br/>BusinessClock, идемпотентность, аудит"]

    web --> auth & users & stores & catalog & suppliers & inventory & purchases & sales & returns & expenses & expiration & replenishment & reporting

    auth --> users
    users --> stores
    catalog --> stores
    suppliers --> catalog
    inventory --> catalog
    purchases --> inventory
    purchases --> suppliers
    sales --> inventory
    sales --> catalog
    returns --> sales
    returns --> inventory
    expenses --> stores
    finance --> sales
    finance --> returns
    finance --> inventory
    finance --> expenses
    expiration --> inventory
    expiration --> sales
    replenishment --> expiration
    replenishment --> purchases
    replenishment --> suppliers
    replenishment --> sales
    reporting --> finance
    reporting --> expiration
    reporting --> replenishment
    reporting --> users

    classDef core fill:#fde68a,stroke:#b45309,color:#111
    classDef read fill:#bfdbfe,stroke:#1d4ed8,color:#111
    class inventory,sales,purchases,returns core
    class finance,reporting,replenishment,expiration read
```

Жёлтым — модули, меняющие остатки под блокировками; синим — модули, которые только читают и вычисляют.

## Компоненты модуля inventory (пример внутреннего устройства)

```mermaid
flowchart LR
    subgraph inventory["inventory"]
        direction TB
        api["api<br/>StockApi · WriteOffApi · BatchQueryApi · LockService"]
        app["application<br/>PostWriteOffService · PostStockCountService<br/>IssueStockService · ReceiveStockService<br/>ReconciliationJob"]
        dom["domain (чистые функции и сущности)<br/>FefoAllocator · WeightedAverageCost<br/>Batch · ProductStock · StockMovement"]
        per["persistence<br/>BatchRepository (FOR UPDATE)<br/>ProductStockRepository · MovementRepository"]
        ctl["web<br/>BatchController · WriteOffController<br/>StockCountController · StockController"]
    end
    ctl --> app
    api --> app
    app --> dom
    app --> per
    per --> pg[("PostgreSQL")]
```
