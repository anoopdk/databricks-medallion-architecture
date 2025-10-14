
# 🧠 Functions in Databricks – Detailed Guide

Databricks supports the use of **functions** as schema-level securables that encapsulate reusable logic.  
Functions can be **built-in** (provided by Databricks) or **user-defined functions (UDFs)** created by users in SQL or Python.  

UDFs extend the capabilities of SQL and DataFrame APIs by allowing custom operations and calculations to be **stored**, **secured**, and **reused** across different notebooks and users.

---

## 1. Function Types

Databricks supports the following types of functions:

### 🔸 Built-in Functions
- Default Databricks functions such as `expr`, `concat`, etc.

### 🔸 User-Defined Functions (UDFs)
Custom functions created by users, categorized as:

- **Scalar Functions** – Accept zero or more inputs and return a single output value.  
- **Table Functions** – Accept zero or more inputs and return a set of rows (table format).

---

## 2. Scalar Functions

Scalar functions take zero or more input parameters but always return a single output value.  
They can be written in either **SQL** or **Python**, and are stored within the schema, making them reusable across notebooks and sessions.

### ✅ Example – SQL Scalar Function

```sql
CREATE OR REPLACE FUNCTION dev.bronze.calculate_tax(sal DOUBLE)
RETURNS DOUBLE
LANGUAGE SQL
RETURN sal * 0.3;
```

**Using in SQL Notebook:**
```sql
SELECT *, dev.bronze.calculate_tax(1000) AS tax FROM dev.bronze.emp;
```

**Using in Python Notebook:**
```python
%python
from pyspark.sql.functions import expr
df = spark.read.table("dev.bronze.emp")
df = df.withColumn("tax", expr("dev.bronze.calculate_tax(salary)"))
# Note - to use sql functions in Python cell/notebook you need to use expr
```

---

### ✅ Example – Python Scalar Function

```sql
CREATE OR REPLACE FUNCTION dev.bronze.calculate_tax(sal DOUBLE)
RETURNS DOUBLE
LANGUAGE PYTHON
$$
    output = sal * 0.3
    return output
$$;

-- Note -You need to use double $($$) and write python logic within it.
```

**Using Python UDF in SQL Cell:**
```sql
%sql
SELECT *, dev.bronze.calculate_tax(1000) AS tax FROM dev.bronze.emp;
```

---

## 3. Table Functions

Table functions return a **set of rows (similar to a table)**.  
They are useful for creating parameterized queries or reusable data retrieval logic that can be treated as a table in SQL.

### ✅ Example – SQL Table Function

```sql
CREATE OR REPLACE FUNCTION dev.bronze.get_emp_details(dept_num STRING)
RETURNS TABLE(employeeId STRING, employeeName STRING)
LANGUAGE SQL
RETURN (
    SELECT emp_id AS employeeId, emp_name AS employeeName
    FROM dev.bronze.emp
    WHERE dept_id = dept_num
);
```

**Using Table Function in SQL Notebook:**
```sql

--Note - when calling a table functions in sql cell, it should be treated as a table instead.

SELECT * FROM dev.bronze.get_emp_details('D01');
```

> **Note:** When invoking a table function in SQL, treat it as a **table source**, not as a scalar function call.

---

## 4. Storage and Persistence

Unlike standard **Python session UDFs**, Databricks UDFs (Functions) are **registered and stored persistently** within **Unity Catalog**.  

This means they can be managed like other schema-level securables (e.g., **tables**, **views**, etc.) and accessed later across multiple sessions and notebooks.

---

## 5. Best Practices

- Always define functions under a specific schema (e.g., `dev.bronze`) for better governance.  
- Use **SQL scalar functions** for lightweight computations.  
- Use **Python functions** for complex logic.  
- Grant proper permissions on functions at the schema level (e.g., `EXECUTE` privilege).  
- Maintain naming conventions like `calculate_*` or `get_*` for clarity.  
- Use **table functions** to modularize recurring data retrieval logic.

---

## 🧩 Final Key Points

- Functions are **securables** that exist at the **schema level** within Unity Catalog.  
- Two main types:
  - **Built-in Functions** – Provided by Databricks (e.g., `expr`, `concat`).  
  - **User-Defined Functions (UDFs)** – Created by users.

### UDF Categories
- **Scalar Functions**
  - Accept zero or more inputs.  
  - Return a single output value.  
  - Defined using a `RETURN` clause.  
- **Table Functions**
  - Accept zero or more inputs.  
  - Return a set of rows (table format).  
  - Defined using a `RETURNS TABLE` clause.

### Additional Notes
- Functions can be written in **SQL** or **Python**.  
- SQL-based functions can be invoked from Python (using `expr`) and vice versa.  
- Regular Python UDFs:
  - Exist only within a session.  
  - Are not stored persistently.  
- Function UDFs:
  - Are registered and **stored persistently** in the metastore.  
  - Can be managed like tables or views.  

> **In summary:** Databricks Function UDFs are **persistently stored, reusable, and managed** entities — unlike temporary, session-based Python UDFs.
