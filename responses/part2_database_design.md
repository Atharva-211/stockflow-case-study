# Part 2 – Database Design

## Schema (PostgreSQL DDL Style)

```sql
CREATE TABLE companies (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL
);

CREATE TABLE warehouses (
    id SERIAL PRIMARY KEY,
    company_id INTEGER NOT NULL REFERENCES companies(id),
    name TEXT NOT NULL,
    UNIQUE(company_id, name)
);

CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    sku TEXT NOT NULL UNIQUE,
    price_cents INTEGER CHECK (price_cents >= 0),
    is_bundle BOOLEAN DEFAULT FALSE
);

CREATE TABLE inventories (
    product_id INTEGER REFERENCES products(id),
    warehouse_id INTEGER REFERENCES warehouses(id),
    quantity INTEGER DEFAULT 0,
    PRIMARY KEY (product_id, warehouse_id)
);

CREATE TABLE inventory_transactions (
    id BIGSERIAL PRIMARY KEY,
    product_id INTEGER NOT NULL,
    warehouse_id INTEGER NOT NULL,
    delta INTEGER NOT NULL,
    reason TEXT,
    tx_time TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE suppliers (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    contact_email TEXT
);

CREATE TABLE supplier_products (
    supplier_id INTEGER NOT NULL REFERENCES suppliers(id),
    product_id INTEGER NOT NULL REFERENCES products(id),
    lead_time_days INTEGER,
    PRIMARY KEY (supplier_id, product_id)
);

CREATE TABLE reorder_policies (
    product_id INTEGER PRIMARY KEY REFERENCES products(id),
    threshold_qty INTEGER NOT NULL
);

-- Bundle products (BOM)
CREATE TABLE product_components (
    parent_product_id INTEGER REFERENCES products(id),
    component_id INTEGER REFERENCES products(id),
    qty INTEGER NOT NULL,
    PRIMARY KEY (parent_product_id, component_id)
);
```

---

## Design Justification

- **Composite PK in `inventories`** provides O(1) lookup by product-warehouse.
- **Normalized schema** offers flexibility by separating products from inventory.
- **`product_components`** table supports bundles as multi-product compositions (BOM structure).
- **`inventory_transactions`** mirrors accounting ledgers to preserve audit trails.
- **Indexes** expected on `(product_id)`, `(warehouse_id)`, and `sku`.

---

## Questions to Product Team (Gaps)

1. Do we need **lot/serial tracking** (e.g., for perishable goods or recalls)?
2. Should we support **multi-unit measures** (e.g., 1 case = 12 items)?
3. Do products ever have **multiple suppliers or tiered costs**?
4. Should `inventory_transactions` **track users/actions** responsible for changes?

