# Последовательность: распределение FEFO

Внутри транзакции продажи (шаг `allocateAndIssue` в [sequence-pos-sale.md](sequence-pos-sale.md)). Физическое распределение (какая партия) и учётная оценка (по какой себестоимости) разделены: цена партии в себестоимость не входит (`ALGORITHMS.md`, раздел 1).

```mermaid
sequenceDiagram
    autonumber
    participant SL as sales: PostSaleService
    participant L as inventory: LockService
    participant F as inventory: FefoAllocator (чистая функция)
    participant C as inventory: WeightedAverageCost
    participant DB as PostgreSQL

    SL->>L: lockProducts(storeId, productIds по возрастанию)
    L->>DB: SELECT product_stock ... ORDER BY product_id FOR UPDATE
    alt ждали > 5 с (55P03) или deadlock (40P01)
        DB-->>SL: ошибка
        SL-->>SL: откат → 409 CONCURRENCY_RETRY
    end
    L->>DB: SELECT batch WHERE product_id IN (...) AND qty_on_hand > 0<br/>ORDER BY id FOR UPDATE
    DB-->>L: партии (заблокированы)
    loop каждая строка чека
        SL->>F: allocate(партии товара, qty, today, blockOnExpiryDay, manualBatchId)
        F->>F: кандидаты: ACTIVE, остаток > 0,<br/>срок > сегодня (или = сегодня, если разрешено)
        opt ручной выбор партии
            F->>F: годна? иначе EXPIRED_BATCH / BATCH_BLOCKED
        end
        F->>F: сортировка: срок ↑ (пустой — последним),<br/>дата приёмки ↑, id ↑
        F->>F: берём min(остаток, нужно) по порядку
        alt не хватило
            F-->>SL: EXPIRED_ONLY или STOCK_INSUFFICIENT { available }
            SL-->>SL: откат всей продажи (BR-011)
        else хватило
            F-->>SL: [(batchId, qty), ...], Σ qty = qty строки
        end
        SL->>C: unitCost(product_stock.avg_cost)
        C-->>SL: avg (расход среднюю не меняет)
    end
    SL->>L: issue(распределения, DocumentRef)
    L->>DB: UPDATE batch SET qty_on_hand = qty_on_hand − q (CHECK ≥ 0)
    L->>DB: UPDATE product_stock SET qty_on_hand = qty_on_hand − Σq
    L->>DB: INSERT stock_movement SALE (по одному на распределение, unit_cost = avg)
    L->>DB: batch.status = DEPLETED, если остаток 0
```
