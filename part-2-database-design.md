# Part 2: Database Design

## Proposed Schema

```sql
Company (
  id BIGINT PRIMARY KEY,
  name VARCHAR,
  created_at TIMESTAMP
);

Warehouse (
  id BIGINT PRIMARY KEY,
  company_id BIGINT,
  name VARCHAR,
  location VARCHAR
);

Product (
  id BIGINT PRIMARY KEY,
  sku VARCHAR UNIQUE,
  name VARCHAR,
  price DECIMAL(10,2),
  product_type VARCHAR,
  is_bundle BOOLEAN
);

Inventory (
  id BIGINT PRIMARY KEY,
  product_id BIGINT,
  warehouse_id BIGINT,
  quantity INT,
  UNIQUE(product_id, warehouse_id)
);

InventoryHistory (
  id BIGINT PRIMARY KEY,
  inventory_id BIGINT,
  change INT,
  reason VARCHAR,
  created_at TIMESTAMP
);

Supplier (
  id BIGINT PRIMARY KEY,
  name VARCHAR,
  contact_email VARCHAR
);

ProductSupplier (
  product_id BIGINT,
  supplier_id BIGINT,
  PRIMARY KEY(product_id, supplier_id)
);

BundleItem (
  bundle_id BIGINT,
  component_product_id BIGINT,
  quantity INT,
  PRIMARY KEY(bundle_id, component_product_id)
);

Sales (
  id BIGINT PRIMARY KEY,
  product_id BIGINT,
  warehouse_id BIGINT,
  quantity INT,
  sold_at TIMESTAMP
);

Missing Requirements / Questions
  1. Is SKU unique globally or per company?
  2. Definition of “recent sales” (time window)
  3. Can products have multiple preferred suppliers?
  4. Can bundles be sold independently?
  5. Are inventory reservations required?
  7. Soft delete vs hard delete?

Design Decisions
  1. Inventory separated from Product to support multi-warehouse
  2. Inventory History added for auditing and analytics
  3. Unique constraints prevent data corruption
  4. BundleItem allows flexible product composition
  5. Indexes recommended on SKU, product_id, warehouse_id
