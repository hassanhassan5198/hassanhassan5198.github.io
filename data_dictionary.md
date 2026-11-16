# Data Dictionary: `bigquery-public-data.thelook_ecommerce`

This dataset is a synthetic online retail dataset. All tables use the prefix `bigquery-public-data.thelook_ecommerce.`. Always use fully qualified table names in queries.

---

## Business rules (read first)

These rules define how common business terms map to columns. Follow them exactly.

1. **Revenue** means `SUM(order_items.sale_price)`. Do not use `products.retail_price`, which is the list price, not the amount paid.
2. **Units sold** means `COUNT(*)` of rows in `order_items`, or `COUNT(order_items.id)`. Each row is one unit.
3. **Returns:** an item was returned when `order_items.returned_at IS NOT NULL`. Use this column, not `status`, to count returned items.
4. **Cancelled orders:** unless the question asks otherwise, exclude rows where `status = 'Cancelled'` from revenue and unit counts.
5. **Status values** must match the data exactly, including capitalization. Use only the values shown in the Status values section below.
6. **Category and brand names** come from `products.category` and `products.brand`. Match them to the data as written (for example, `Jeans`, not `jeans`).
7. **Customer country, state, and city** come from `users`, not from `orders`.
8. **Date filters:** use `created_at` for when an order or item was placed. Years are extracted with `EXTRACT(YEAR FROM created_at)`. Questions with relative dates such as "last month" should be treated as ambiguous unless a fixed date is given.
9. **Counting customers:** use `COUNT(DISTINCT user_id)` when a question asks about customers who bought something. Use `COUNT(*)` on `users` only when it asks about all registered accounts.
10. **Order count** means `COUNT(DISTINCT order_id)` from `orders` or `order_items`. An order can contain several items, so `COUNT(*)` on `order_items` counts items, not orders.

---

## Status values

These are the status values in the data. Verify them with the queries at the bottom of this file before you rely on them.

| Table | Column | Values |
|---|---|---|
| `orders` | `status` | `Cancelled`, `Complete`, `Processing`, `Returned`, `Shipped` |
| `order_items` | `status` | `Cancelled`, `Complete`, `Processing`, `Returned`, `Shipped` |

---

## Tables

### 1. `orders`

**One row represents:** one customer order, which can contain several items.

| Column | Type | Meaning |
|---|---|---|
| `order_id` | INTEGER | Unique order identifier. Primary key. |
| `user_id` | INTEGER | Customer who placed the order. Joins to `users.id`. |
| `status` | STRING | Order status, such as `Complete` or `Returned`. |
| `gender` | STRING | Gender recorded on the order. |
| `created_at` | TIMESTAMP | When the order was placed. |
| `shipped_at` | TIMESTAMP | When the order shipped. Null if not shipped. |
| `delivered_at` | TIMESTAMP | When the order was delivered. Null if not delivered. |
| `returned_at` | TIMESTAMP | When the order was returned. Null if not returned. |
| `num_of_item` | INTEGER | Number of items in the order. |

**Key joins**
- `orders.order_id` → `order_items.order_id` (one order has many items)
- `orders.user_id` → `users.id`

---

### 2. `order_items`

**One row represents:** one unit of one product sold in one order. This is the main fact table for sales and returns.

| Column | Type | Meaning |
|---|---|---|
| `id` | INTEGER | Unique identifier for this order line. Primary key. |
| `order_id` | INTEGER | The order this item belongs to. Joins to `orders.order_id`. |
| `user_id` | INTEGER | Customer who bought the item. Joins to `users.id`. |
| `product_id` | INTEGER | The product sold. Joins to `products.id`. |
| `inventory_item_id` | INTEGER | The specific physical unit sold. Joins to `inventory_items.id`. |
| `status` | STRING | Item status, such as `Complete`, `Returned`, or `Cancelled`. |
| `created_at` | TIMESTAMP | When the item was ordered. |
| `shipped_at` | TIMESTAMP | When the item shipped. Null if not shipped. |
| `delivered_at` | TIMESTAMP | When the item was delivered. Null if not delivered. |
| `returned_at` | TIMESTAMP | When the item was returned. **Non-null means the item was returned.** |
| `sale_price` | FLOAT | Amount the customer paid for this item. Use this for revenue. |

**Key joins**
- `order_items.order_id` → `orders.order_id`
- `order_items.user_id` → `users.id`
- `order_items.product_id` → `products.id`
- `order_items.inventory_item_id` → `inventory_items.id`

---

### 3. `products`

**One row represents:** one product in the catalog (a product type, not a single physical unit).

| Column | Type | Meaning |
|---|---|---|
| `id` | INTEGER | Unique product identifier. Primary key. |
| `name` | STRING | Product name. |
| `brand` | STRING | Brand name. |
| `category` | STRING | Product category, such as `Jeans` or `Outerwear & Coats`. |
| `department` | STRING | Department the category belongs to, such as `Men` or `Women`. |
| `sku` | STRING | Stock keeping unit code. |
| `cost` | FLOAT | What the retailer paid for the product. |
| `retail_price` | FLOAT | List price. Not the amount paid. Do not use for revenue. |
| `distribution_center_id` | INTEGER | Warehouse that stocks the product. Joins to `distribution_centers.id`. |

**Key joins**
- `products.id` → `order_items.product_id`
- `products.id` → `inventory_items.product_id`
- `products.distribution_center_id` → `distribution_centers.id`

---

### 4. `inventory_items`

**One row represents:** one physical unit of a product in stock, from the moment it was received until it was sold or is still on hand.

| Column | Type | Meaning |
|---|---|---|
| `id` | INTEGER | Unique unit identifier. Primary key. |
| `product_id` | INTEGER | The product this unit belongs to. Joins to `products.id`. |
| `created_at` | TIMESTAMP | When the unit entered inventory. |
| `sold_at` | TIMESTAMP | When the unit sold. Null if still in stock. |
| `cost` | FLOAT | Cost of this specific unit. |
| `product_category` | STRING | Category, copied from the product when the unit was stocked. |
| `product_brand` | STRING | Brand, copied from the product. |
| `product_retail_price` | FLOAT | List price at the time of stocking. |
| `product_distribution_center_id` | INTEGER | Warehouse holding the unit. Joins to `distribution_centers.id`. |

**Key joins**
- `inventory_items.id` → `order_items.inventory_item_id`
- `inventory_items.product_id` → `products.id`
- `inventory_items.product_distribution_center_id` → `distribution_centers.id`

**Note:** a unit that has not sold has a null `sold_at`. A unit that sold has a non-null `sold_at`, and it should match one row in `order_items`.

---

### 5. `users`

**One row represents:** one registered customer account.

| Column | Type | Meaning |
|---|---|---|
| `id` | INTEGER | Unique customer identifier. Primary key. |
| `first_name` | STRING | Customer first name. |
| `last_name` | STRING | Customer last name. |
| `email` | STRING | Customer email. Do not expose in outputs. |
| `age` | INTEGER | Customer age. |
| `gender` | STRING | Customer gender. |
| `street_address` | STRING | Street address. |
| `city` | STRING | City. |
| `state` | STRING | State or region. |
| `postal_code` | STRING | Postal code. |
| `country` | STRING | Country, such as `United States`. |
| `latitude` | FLOAT | Latitude of the address. |
| `longitude` | FLOAT | Longitude of the address. |
| `traffic_source` | STRING | How the customer found the site, such as `Search`, `Organic`, `Facebook`, `Email`, or `Display`. |
| `created_at` | TIMESTAMP | When the account was created. |

**Key joins**
- `users.id` → `orders.user_id`
- `users.id` → `order_items.user_id`

---

### 6. `distribution_centers`

**One row represents:** one warehouse or distribution center.

| Column | Type | Meaning |
|---|---|---|
| `id` | INTEGER | Unique warehouse identifier. Primary key. |
| `name` | STRING | Warehouse name. |
| `latitude` | FLOAT | Latitude of the warehouse. |
| `longitude` | FLOAT | Longitude of the warehouse. |

**Key joins**
- `distribution_centers.id` → `products.distribution_center_id`
- `distribution_centers.id` → `inventory_items.product_distribution_center_id`

---

## Join map

```
orders.order_id                      -> order_items.order_id
orders.user_id                       -> users.id
order_items.user_id                  -> users.id
order_items.product_id               -> products.id
order_items.inventory_item_id        -> inventory_items.id
products.distribution_center_id      -> distribution_centers.id
inventory_items.product_id           -> products.id
inventory_items.product_distribution_center_id -> distribution_centers.id
```

**Join warnings**
- `orders` and `order_items` are a one-to-many relationship. Joining them multiplies rows. Aggregate `order_items` first when summing revenue, or use `COUNT(DISTINCT order_id)` for order counts.
- `products` and `inventory_items` are one-to-many. Join through `order_items` when you need what was actually sold.
- To find the customer for an item, use `order_items.user_id`. You do not need to go through `orders`.

---

## Common mistakes to avoid

- Using `products.retail_price` as revenue. Use `order_items.sale_price` instead.
- Counting orders with `COUNT(*)` on `order_items`. This counts items, not orders.
- Filtering returns with `status = 'Returned'` only. Use `returned_at IS NOT NULL` to be safe.
- Joining `orders` to `products` directly. Always go through `order_items`.
- Using the word "last quarter" or "this year" without a fixed date. Ask for a specific period instead.

---

## Verification queries

Run these in BigQuery before you finalize the dictionary. Replace any value in this file that doesn't match.

```sql
-- Confirm status values in orders
SELECT DISTINCT status
FROM `bigquery-public-data.thelook_ecommerce.orders`
ORDER BY status;

-- Confirm status values in order_items
SELECT DISTINCT status
FROM `bigquery-public-data.thelook_ecommerce.order_items`
ORDER BY status;

-- Confirm returned items are identified by returned_at
SELECT
  COUNT(*) AS total_items,
  COUNTIF(returned_at IS NOT NULL) AS returned_by_timestamp,
  COUNTIF(status = 'Returned') AS returned_by_status
FROM `bigquery-public-data.thelook_ecommerce.order_items`;

-- Confirm the categories and brands you'll reference
SELECT category, COUNT(*) AS products
FROM `bigquery-public-data.thelook_ecommerce.products`
GROUP BY category
ORDER BY products DESC;
```

If `returned_by_timestamp` and `returned_by_status` return different counts, record that in the business rules, because it changes how "returned" should be defined.
