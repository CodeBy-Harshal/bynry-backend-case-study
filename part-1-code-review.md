# Part 1: Code Review & Debugging

## Original Code (Given)

```python
@app.route('/api/products', methods=['POST'])
def create_product():
    data = request.json

    product = Product(
        name=data['name'],
        sku=data['sku'],
        price=data['price'],
        warehouse_id=data['warehouse_id']
    )

    db.session.add(product)
    db.session.commit()

    inventory = Inventory(
        product_id=product.id,
        warehouse_id=data['warehouse_id'],
        quantity=data['initial_quantity']
    )

    db.session.add(inventory)
    db.session.commit()

    return {"message": "Product created", "product_id": product.id}

Issues Identified:
  1. request.json used instead of request.get_json()
  2. No request payload validation
  3. SKU uniqueness not enforced
  4.Product incorrectly linked to a warehouse
  5. Price handled as float (precision issue)
  6. Multiple commits (non-atomic operation)
  7. No rollback on failure
  8. Inventory duplication possible
  9. Optional fields assumed mandatory
  10. Negative inventory allowed
  11. No proper HTTP status codes

Production Impact:
  1. Duplicate SKUs break reporting and integrations
  2. Partial commits cause orphaned records
  3. Floating-point pricing leads to billing errors
  4. Multi-warehouse support fails
  5. Inventory inconsistencies can cause overselling
  6. API becomes unreliable under concurrency

Corrected Code 
python

@app.route('/api/products', methods=['POST'])
def create_product():

    # request.json was used earlier
    # get_json() safely parses request body
    data = request.get_json()

    # No validation earlier
    if not data:
        return jsonify({"error": "Invalid JSON body"}), 400

    # Required fields not validated earlier
    required_fields = ['name', 'sku', 'price']
    for field in required_fields:
        if field not in data:
            return jsonify({"error": f"{field} is required"}), 400

    # Price used directly (float issue)
    # Decimal used for monetary values
    try:
        price = Decimal(str(data['price']))
        if price < 0:
            raise ValueError
    except:
        return jsonify({"error": "Invalid price"}), 400

    try:
        # Multiple commits earlier
        # Single atomic transaction
        with db.session.begin():

            # Product tied to warehouse earlier
            # Product is warehouse-independent
            product = Product(
                name=data['name'],
                sku=data['sku'],  # SKU uniqueness not enforced earlier
                price=price
            )
            db.session.add(product)
            db.session.flush()

            # Inventory assumed mandatory earlier
            if 'warehouse_id' in data and 'initial_quantity' in data:
                inventory = Inventory(
                    product_id=product.id,
                    warehouse_id=data['warehouse_id'],
                    quantity=max(0, data['initial_quantity'])  # prevent negative
                )
                db.session.add(inventory)

        return jsonify({
            "message": "Product created successfully",
            "product_id": product.id
        }), 201

    except IntegrityError:
        db.session.rollback()
        return jsonify({"error": "SKU already exists"}), 409

    except Exception:
        db.session.rollback()
        return jsonify({"error": "Internal server error"}), 500
