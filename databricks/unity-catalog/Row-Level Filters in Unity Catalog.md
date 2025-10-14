# Row-Level Filters in Unity Catalog

## Overview

Row-Level Filters (RLF), also known as **Row-Level Security (RLS)**, enable fine-grained access control in Databricks Unity Catalog. They allow administrators to **restrict or filter sensitive data** within the same table so that different users see only the data relevant to them.

This is a critical feature in **data governance**, ensuring that sensitive information (like salaries or confidential business data) is only visible to authorized users.

---

## Why Row-Level Filters Are Important

While **object-level security** controls who can access entire tables, views, or schemas, it does not protect *specific rows* within a table.  
RLF adds an additional layer of control — allowing different users to query the same table but see filtered data.

**Example Use Cases:**
- Limiting employees to see only their own department’s data.
- Restricting financial analysts to view only transactions from assigned regions.
- Allowing managers to view all rows, but subordinates to view only their own.

---

## Step 1: Setup — Understanding the Scenario

We’ll work with a table named **`customer_raw`** under the schema `dev.bronze`.  
Both `edata_user` and `data_engineer_user` currently have full access to this table.  
We’ll restrict access as follows:
- `edata_user` → can see only *Building* market segment
- `data_engineer_user` → can see only *Household* market segment

---

## Step 2: Create Metadata Schema and Control Table

We’ll use a **metadata schema** to hold a control table defining user-to-segment mappings.

```sql
CREATE SCHEMA dev.metadata;
```

Create the mapping table:

```sql
CREATE TABLE dev.metadata.allowed_market_segment (
  user_email STRING,
  market_segment STRING
);
```

Insert sample records for each user:

```sql
INSERT INTO dev.metadata.allowed_market_segment VALUES
('edata_user@databricks.com', 'Building'),
('data_engineer_user@databricks.com', 'Household');
```

This table maps which user is allowed to see which market segment.

---

## Step 3: Create a Boolean Filter Function

We now create a SQL function that checks if the **current user** has access to a given market segment.

Test the `current_user()` built-in function:

```sql
SELECT current_user();
```

It returns the email of the logged-in Databricks user.  
We’ll use this to dynamically determine access.

### Function Logic (Conceptual)
1. Capture the current user using `current_user()`.
2. Check if this user’s email and the market segment exist in the mapping table.
3. Return `TRUE` if allowed, else `FALSE`.

### SQL Query Prototype

```sql
SELECT EXISTS (
  SELECT 1
  FROM dev.metadata.allowed_market_segment AS a
  WHERE a.user_email = current_user()
    AND a.market_segment = 'Building'
);
```

If the query returns `TRUE`, the user can see rows for *Building*.  

### Create the Function

```sql
CREATE OR REPLACE FUNCTION dev.metadata.is_allowed_market_segment(mkt_segment STRING)
RETURNS BOOLEAN
LANGUAGE SQL
RETURN
  EXISTS (
    SELECT 1
    FROM dev.metadata.allowed_market_segment a
    WHERE a.user_email = current_user()
      AND a.market_segment = mkt_segment
  );
```

This function returns `TRUE` if the user is authorized for the given market segment.

---

## Step 4: Validate the Function

Test it by passing different market segments:

```sql
SELECT dev.metadata.is_allowed_market_segment('Building');  -- TRUE for edata_user
SELECT dev.metadata.is_allowed_market_segment('Household'); -- FALSE for edata_user
```

---

## Step 5: Create Dynamic Query (Filter on the Fly)

Use this function in a `WHERE` clause to filter table rows dynamically:

```sql
SELECT *
FROM dev.bronze.customer_raw
WHERE dev.metadata.is_allowed_market_segment(market_segment);
```

✅ **Output:** Each user now sees only rows matching their allowed market segment.

---

## Step 6: Create a Dynamic View (Optional)

You can create a *dynamic view* for users to access the filtered data directly.

```sql
CREATE OR REPLACE VIEW dev.bronze.allowed_market_segment_view AS
SELECT *
FROM dev.bronze.customer_raw
WHERE dev.metadata.is_allowed_market_segment(market_segment);
```

Now, whenever a user queries this view, they’ll automatically see only their permitted data.

---

## Step 7: Apply the Row Filter to a Table

To permanently associate this function with the base table, use `ALTER TABLE`:

```sql
ALTER TABLE dev.bronze.customer_raw
SET ROW FILTER dev.metadata.is_allowed_market_segment
ON (market_segment);
```

Verify the applied filter:

```sql
DESCRIBE EXTENDED dev.bronze.customer_raw;
```

You should see a **Row Filter** entry in the table metadata.

---

## Step 8: Test with Different Users

### For `edata_user`
```sql
SELECT * FROM dev.bronze.customer_raw;
```
→ Sees only *Building* (and possibly *Machinery* if allowed).

### For `data_engineer_user`
```sql
SELECT * FROM dev.bronze.customer_raw;
```
→ Sees only *Household*.

---

## Step 9: Reverse the Logic (Optional)

You can invert the logic to *exclude* rather than *allow* specific data by changing `EXISTS` → `NOT EXISTS`.

Example:
```sql
CREATE OR REPLACE FUNCTION dev.metadata.exclude_market_segment(mkt_segment STRING)
RETURNS BOOLEAN
LANGUAGE SQL
RETURN
  NOT EXISTS (
    SELECT 1
    FROM dev.metadata.allowed_market_segment a
    WHERE a.user_email = current_user()
      AND a.market_segment = mkt_segment
  );
```

This hides only the specified segments and shows everything else.

---

## Best Practices and Notes

- ✅ **Use RLF for sensitive datasets** — especially HR, Finance, or restricted domains.  
- ⚙️ Combine with **object-level permissions** for multi-layered security.  
- 🧱 Store mappings in a **metadata schema** for easier maintenance.  
- 🧑‍💼 Avoid hardcoding user names — use `current_user()` or `current_group()`.  
- 💡 Always test RLF logic with multiple user sessions before enabling in production.  
- 🔒 Restrict modification of RLF functions and metadata tables to admins only.

---

## Summary

Row-Level Filters in Unity Catalog offer a powerful mechanism to enforce **data privacy and fine-grained security** across users and teams.

They allow you to dynamically filter table rows based on user identity — ensuring users only see what they are supposed to see.

---

**Next Steps:**  
In the next module, we’ll explore **Column-Level Masking** — another key feature of Unity Catalog for securing sensitive information like personal identifiers and financial details.
