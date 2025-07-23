# Part 3 – Low-Stock Alert API

## Assumptions

- `inventory_transactions` captures all stock movements with `+/- delta` values.
- "Recent sales activity" means transactions in the last **30 days**.
- **Low stock** is defined as: `quantity < threshold` in `reorder_policies`.
- Supplier link is via `supplier_products`, sorted by `lead_time_days`.

---

## Implementation (Flask + SQL)

```python
@app.route('/api/companies/<int:company_id>/alerts/low-stock', methods=['GET'])
def low_stock_alerts(company_id):
    query = text("""
    WITH recent_sales AS (
        SELECT product_id, warehouse_id, SUM(-delta) AS sales_30d
        FROM inventory_transactions
        WHERE reason = 'sale' AND tx_time >= now() - INTERVAL '30 days'
        GROUP BY product_id, warehouse_id
    ),
    thresholded AS (
        SELECT i.product_id, i.warehouse_id, i.quantity, rp.threshold_qty,
               COALESCE(rs.sales_30d, 0) AS sales_30d
        FROM inventories i
        JOIN reorder_policies rp ON rp.product_id = i.product_id
        LEFT JOIN recent_sales rs ON rs.product_id = i.product_id AND rs.warehouse_id = i.warehouse_id
    ),
    supplier_info AS (
        SELECT sp.product_id,
               sp.supplier_id,
               s.name AS supplier_name,
               s.contact_email,
               ROW_NUMBER() OVER (PARTITION BY sp.product_id ORDER BY sp.lead_time_days) = 1 AS first_choice
        FROM supplier_products sp
        JOIN suppliers s ON s.id = sp.supplier_id
    )
    SELECT p.id AS product_id,
           p.name AS product_name,
           p.sku,
           w.id AS warehouse_id,
           w.name AS warehouse_name,
           t.quantity AS current_stock,
           t.threshold_qty AS threshold,
           CASE WHEN t.sales_30d = 0 THEN NULL
                ELSE CEIL(t.quantity / (t.sales_30d / 30.0)) END AS days_until_stockout,
           jsonb_build_object(
              'id', si.supplier_id,
              'name', si.supplier_name,
              'contact_email', si.contact_email
           ) AS supplier
    FROM thresholded t
    JOIN products p ON p.id = t.product_id
    JOIN warehouses w ON w.id = t.warehouse_id AND w.company_id = :company_id
    LEFT JOIN supplier_info si ON si.product_id = p.id AND si.first_choice
    WHERE t.quantity < t.threshold_qty
    ORDER BY t.quantity ASC
    LIMIT 100
    """)

    res = db.session.execute(query, {"company_id": company_id})
    alerts = [dict(row._mapping) for row in res]

    return jsonify({"alerts": alerts, "total_alerts": len(alerts)})
```

---

## Edge Case Handling

- **Zero sales?** → `days_until_stockout = null` (can’t extrapolate).
- **Missing supplier?** → `supplier = null` handled gracefully.
- No alerts if stock exceeds threshold or no recent sales activity.
- Only returns alerts for the specified `company_id`.

---

## Improvements If Time Allowed

- Paging support via `limit/offset` using `request.args`.
- Caching with Redis or a materialized view to boost performance.
- Precompute daily sales aggregates to reduce query cost.

---

## Final Notes

### Key Assumptions

- Prices are stored in cents (`int`) for financial accuracy.
- 30-day window used for recent sales extrapolation.
- Fastest supplier = one with the **lowest lead_time_days**.
- **Bundles** treated as separate product entries (no real-time BOM unpacking).
- No multi-currency, tax, or unit-of-measure conversions implemented.

### Alternative Considerations

- Use **SQL views** or a **data warehouse** (e.g., BigQuery) for analytics/reporting.
- Logic for bundle alerts could use recursive dependency trees (omitted for simplicity).
