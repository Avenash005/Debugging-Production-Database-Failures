# DEBUG-REPORT.md

## 🐞 Bug 1 — Orders with Missing Customer Data

### 🔍 Symptom

Some orders return `NULL` for `customer_name` when fetching all orders.

### 🧪 Reproduction Query

```sql
SELECT o.id, o.customer_id, c.name
FROM orders o
LEFT JOIN customers c ON o.customer_id = c.id
WHERE c.id IS NULL;
```

---

### 🔎 Data Flow Trace

* API endpoint: `/orders`
* Query joins `orders` with `customers` using `LEFT JOIN`
* Orders are inserted via backend without validating `customer_id`
* Since no constraint exists, invalid `customer_id` values are stored

---

### 💥 Root Cause

The `orders.customer_id` column:

* Has **no FOREIGN KEY constraint**
* Is **not enforced as NOT NULL**

This allows orphaned records referencing non-existent customers.

---

### 🛠 Fix Applied

```sql
ALTER TABLE orders
ADD CONSTRAINT fk_orders_customer
FOREIGN KEY (customer_id)
REFERENCES customers(id)
ON DELETE CASCADE;

ALTER TABLE orders
ALTER COLUMN customer_id SET NOT NULL;
```

---

### ✅ Validation

**Re-run reproduction query:**

```sql
SELECT o.id
FROM orders o
LEFT JOIN customers c ON o.customer_id = c.id
WHERE c.id IS NULL;
```

→ Returns **0 rows**

**Attempt invalid insert:**

```sql
INSERT INTO orders (customer_id, total, status)
VALUES (9999, 100, 'pending');
```

→ Fails with foreign key violation

---

## 🐞 Bug 2 — Negative Product Inventory

### 🔍 Symptom

Some products have `inventory_count < 0`.

### 🧪 Reproduction Query

```sql
SELECT * FROM products
WHERE inventory_count < 0;
```

---

### 🔎 Data Flow Trace

* API endpoint: `/order_items`
* When adding items:

  * Inserts order item
  * Decrements inventory directly
* No validation or constraint prevents negative values

---

### 💥 Root Cause

The `products.inventory_count` column:

* Has **no CHECK constraint**
* Allows values below zero

---

### 🛠 Fix Applied

```sql
ALTER TABLE products
ADD CONSTRAINT check_inventory_non_negative
CHECK (inventory_count >= 0);
```

---

### ✅ Validation

**Attempt invalid update:**

```sql
UPDATE products
SET inventory_count = -10
WHERE id = 1;
```

→ Fails due to CHECK constraint

---

## 🐞 Bug 3 — Duplicate Payments for a Single Order

### 🔍 Symptom

Some orders show multiple payment records with conflicting statuses.

### 🧪 Reproduction Query

```sql
SELECT order_id, COUNT(*)
FROM payments
GROUP BY order_id
HAVING COUNT(*) > 1;
```

---

### 🔎 Data Flow Trace

* API endpoint: `/payments`
* Payments are inserted without checking if one already exists
* Retrieval query returns all matching rows

---

### 💥 Root Cause

The `payments.order_id` column:

* Has **no UNIQUE constraint**
* Allows multiple payment records for the same order

---

### 🛠 Fix Applied

```sql
ALTER TABLE payments
ADD CONSTRAINT unique_order_payment UNIQUE (order_id);
```

---

### ✅ Validation

**Attempt duplicate insert:**

```sql
INSERT INTO payments (order_id, amount, status)
VALUES (1, 100, 'completed');
```

→ Fails due to UNIQUE constraint

---

# 🧠 Final Summary

| Bug                     | Root Cause                | Fix                     |
| ----------------------- | ------------------------- | ----------------------- |
| Orders missing customer | Missing FK + NOT NULL     | Added FK + NOT NULL     |
| Negative inventory      | Missing CHECK constraint  | Added CHECK constraint  |
| Duplicate payments      | Missing UNIQUE constraint | Added UNIQUE constraint |

---

# ✅ Key Takeaway

All issues were caused by **missing schema-level constraints**.

Fixing them at the database level ensures:

* Invalid data cannot be inserted
* Application bugs cannot corrupt data
* System remains consistent under all conditions

---
