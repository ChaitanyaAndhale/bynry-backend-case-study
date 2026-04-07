# Backend Engineering Intern Case Study - Bynry

## Approach

I tried to keep the solution simple and practical based on my current understanding.
I made some assumptions where requirements were not clearly defined.

I focused on:

* Keeping data safe (no wrong or partial data)
* Making database easy to scale
* Writing clean and understandable API code

---

## Part 1: Code Review & Debugging

### Issues Identified

1. No input validation
2. No transaction handling
3. SKU uniqueness not checked
4. No error handling
5. Product linked to only one warehouse
6. Price not validated
7. Optional fields not handled
8. No proper status codes

---

### Impact

* API can crash if data is missing
* Inconsistent data (product created but inventory fails)
* Duplicate SKUs create confusion
* Client cannot understand response properly

---

### Fixed Code (Improved Version)

```python
from decimal import Decimal
from flask import request, jsonify

@app.route('/api/products', methods=['POST'])
def create_product():
    data = request.json

    try:
        # Validate required fields
        if 'name' not in data or 'sku' not in data or 'price' not in data:
            return jsonify({"error": "Missing required fields"}), 400

        # Check SKU uniqueness
        existing_product = Product.query.filter_by(sku=data['sku']).first()
        if existing_product:
            return jsonify({"error": "SKU already exists"}), 409

        # Convert price safely
        price = Decimal(str(data['price']))

        # Start transaction
        db.session.begin()

        # Create product
        product = Product(
            name=data['name'],
            sku=data['sku'],
            price=price
        )
        db.session.add(product)
        db.session.flush()

        # Optional inventory creation
        if 'warehouse_id' in data and 'initial_quantity' in data:
            inventory = Inventory(
                product_id=product.id,
                warehouse_id=data['warehouse_id'],
                quantity=data.get('initial_quantity', 0)
            )
            db.session.add(inventory)

        db.session.commit()

        return jsonify({
            "message": "Product created successfully",
            "product_id": product.id
        }), 201

    except Exception as e:
        db.session.rollback()
        return jsonify({"error": str(e)}), 500
```

---

## Part 2: Database Design

### Schema (Simplified)

Company(id, name)

Warehouse(id, company_id, name)

Product(id, company_id, name, sku UNIQUE, price)

Inventory(id, product_id, warehouse_id, quantity)

Inventory_Log(id, product_id, warehouse_id, change, created_at)

Supplier(id, name, contact_email)

Product_Supplier(product_id, supplier_id)

Bundle(id, product_id)

Bundle_Items(bundle_id, child_product_id, quantity)

---

### Missing Requirements

* What is recent sales?
* How is threshold decided?
* Can a product have multiple suppliers?
* Can bundles contain other bundles?
* Do we track reserved stock?

---

### Design Decisions

* Used separate inventory table for multiple warehouses
* Made SKU unique to avoid duplicates
* Added inventory log for tracking changes
* Used many-to-many relation for product and supplier

---

## Part 3: API Implementation

### Approach

* Loop through products
* Check inventory for each warehouse
* Apply threshold logic
* Filter products with recent sales
* Attach supplier information

---

### Assumptions

* Threshold = 20
* Recent sales = last 30 days
* One main supplier per product
* Stockout calculated using average sales

---

### Edge Cases

* No supplier available
* No recent sales
* Multiple warehouses handled
* Missing data handled safely

---

## Conclusion

I tried to keep the solution simple, clear and practical.
I focused on correct data handling, basic scalability and clean code.
