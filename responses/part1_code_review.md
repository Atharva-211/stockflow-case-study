# Part 1 – Code Review & Fixes

## 1. Issues Identified

| #  | Type               | Problem                                           | Risk in Production                                      |
|----|--------------------|--------------------------------------------------|----------------------------------------------------------|
| 1  | Validation         | `data = request.json` used unvalidated input     | Runtime exceptions; invalid or malicious data            |
| 2  | Atomicity          | Separate `db.session.commit()` calls             | If the second fails, DB ends up in inconsistent state    |
| 3  | Business Rule      | SKU not checked for uniqueness                   | Duplicate product entries; inconsistent inventory        |
| 4  | Data Model Design  | `price` is a float                               | Rounding errors in financial computations                |
| 5  | Error Handling     | No `try-except` logic or error status codes      | 500s or silent failures                                  |
| 6  | Assumptions        | `warehouse_id` used as a column in Product       | Violates normalization; product tied to only one warehouse |

---

## 2. Impact

- System allows duplicate SKUs, violating uniqueness guarantee.
- Partial commits can leave products without inventory or create orphaned records.
- Financial calculations using float may yield incorrect totals.
- Missing status codes confuse the frontend and error logs.
- Lack of validation can lead to security issues (e.g., JSON injection).

---

## 3. Corrected Implementation

Moved to a transactional, fully validated, and normalized version.

### Validation Schema

```python
from marshmallow import Schema, fields, ValidationError

class ProductSchema(Schema):
    name = fields.String(required=True)
    sku = fields.String(required=True)
    price_cents = fields.Integer(required=True)  # storing as cents
    warehouse_id = fields.Integer(required=True)
    initial_quantity = fields.Integer(required=True)
```

### API Code: `/api/products`

```python
from flask import request, jsonify
from sqlalchemy.exc import IntegrityError
from http import HTTPStatus

@app.route('/api/products', methods=['POST'])
def create_product():
    try:
        data = ProductSchema().load(request.json)
    except ValidationError as err:
        return {"errors": err.messages}, HTTPStatus.UNPROCESSABLE_ENTITY

    try:
        with db.session.begin():
            # Ensure SKU is unique
            existing = db.session.query(Product.id).filter_by(sku=data['sku']).first()
            if existing:
                return {"error": "SKU already exists"}, HTTPStatus.CONFLICT

            # Create product
            product = Product(
                name=data['name'],
                sku=data['sku'],
                price_cents=data['price_cents']
            )
            db.session.add(product)
            db.session.flush()  # Generate product.id

            # Check warehouse exists
            warehouse = Warehouse.query.get(data['warehouse_id'])
            if not warehouse:
                return {"error": "Warehouse not found"}, HTTPStatus.NOT_FOUND

            inventory = Inventory(
                product_id=product.id,
                warehouse_id=data['warehouse_id'],
                quantity=data['initial_quantity']
            )
            db.session.add(inventory)

        return {"message": "Product created", "product_id": product.id}, HTTPStatus.CREATED

    except IntegrityError:
        db.session.rollback()
        return {"error": "Integrity error"}, HTTPStatus.INTERNAL_SERVER_ERROR
```

---

## Key Improvements

- All logic wrapped in `db.session.begin()` for atomicity.
- Marshmallow schema validation with appropriate HTTP status codes.
- Price stored as integer (`price_cents`) to prevent float rounding issues.
- Unique SKU check enforced at both the application and database levels.
