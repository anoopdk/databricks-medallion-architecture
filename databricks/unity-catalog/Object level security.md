
# Unity Catalog Object-Level Security in Databricks

This document provides a **step-by-step, detailed guide** on managing object-level security and permissions in **Databricks Unity Catalog**. It covers concepts, best practices, and practical examples that explain how and why permissions are assigned at various object levels.

---

## 📘 1. Overview

In Unity Catalog, **object-level security** (also known as *object privilege management*) allows you to control **who can access, modify, or view data objects** like catalogs, schemas, tables, views, volumes, and functions.

The Unity Catalog introduces a **hierarchical model** for managing access:

- **Metastore** → Top-level container.
- **Catalogs** → Logical grouping of schemas.
- **Schemas** → Contain tables, views, volumes, functions, and models.
- **Objects** → The actual data structures (tables, views, etc.) to which privileges can be assigned.

---

## 🔑 2. Types of Privilege Assignment Approaches

Unity Catalog allows two main strategies for assigning privileges:

### 🧭 a. Top-Down Approach (Bulk Privilege Assignment)

- Privileges are granted at a higher (parent) level — for example, on the **catalog** or **schema**.
- All **child objects automatically inherit** these privileges.
- Example: Granting `SELECT` at the catalog level automatically applies it to all schemas and tables within that catalog.

**Use Case:**  
Ideal when you want to **grant broad or bulk access** to a user or group — e.g., giving an analyst access to all tables in a schema.

**Example SQL:**
```sql
GRANT SELECT ON CATALOG dev TO `da_group`;
```

---

### 🎯 b. Bottom-Up Approach (Fine-Grained Access Control)

- Privileges are granted at the **lowest object level** (e.g., specific table or volume).
- Additional `USE CATALOG` and `USE SCHEMA` privileges are required for users to see and query the object.

**Use Case:**  
Best when you want to give **restricted or precise access** — e.g., an engineer accessing only one table or volume.

**Example SQL:**
```sql
GRANT SELECT ON TABLE bronze.customer_raw TO `de_group`;
GRANT USE SCHEMA ON SCHEMA bronze TO `de_group`;
GRANT USE CATALOG ON CATALOG dev TO `de_group`;
```

---

## 🧩 3. Required Roles and Access Levels

To configure object-level security, ensure the following permissions are available:

| Role | Responsibility |
|------|----------------|
| **Cloud Admin** | Creates users and service principals in Azure Entra ID. |
| **Account Admin** | Manages users, groups, and workspace assignments in the Databricks Account Console. |
| **Workspace Admin** | Grants object-level permissions and manages entitlements within workspaces. |
| **Metastore Admin** | Controls all securable objects within the Unity Catalog. |

---

## 🧱 4. Understanding Unity Catalog Object Model

The hierarchical object model in Unity Catalog is as follows:

```
Metastore
└── Catalogs
    └── Schemas
        ├── Tables
        ├── Views
        ├── Volumes
        ├── Functions
        └── Models
```

Each of these objects is **securable**, meaning permissions can be granted or revoked independently.

---

## 🧑‍💻 5. Practical Example: Assigning Object-Level Privileges

### Step 1: Check Current Grants

Before applying new privileges, you can list existing permissions using:

```sql
SHOW GRANTS ON METASTORE;
```

This command displays all principals (users, groups, or service principals) and their existing permissions.

---

### Step 2: Grant Privileges at Metastore Level

To allow a group (e.g., Data Engineers) to create new catalogs:

```sql
GRANT CREATE CATALOG ON METASTORE TO `de_group`;
```

✅ **Why:** Enables Data Engineers to create new catalogs without giving them full Metastore Admin privileges.

After this grant, users in `de_group` will see the **“Add Catalog”** option in the Databricks Catalog Explorer.

---

### Step 3: Assign Catalog-Level Privileges

To provide Data Analysts (`da_group`) with access to an entire catalog (bulk privilege):

```sql
GRANT SELECT, USE CATALOG, USE SCHEMA ON CATALOG dev TO `da_group`;
```

✅ **Why:**  
This allows analysts to view and query all schemas and tables within the **dev** catalog.

---

### Step 4: Assign Schema-Level Privileges

For example, to grant read access to specific volumes in a schema:

```sql
GRANT READ VOLUME ON SCHEMA bronze TO `da_group`;
```

✅ **Why:**  
Schema-level privileges allow selective control — e.g., analysts can read specific data sources without managing the schema itself.

---

### Step 5: Assign Table-Level Privileges

To provide Data Engineers access to a single table only:

```sql
GRANT SELECT ON TABLE bronze.customer_raw TO `de_group`;
GRANT USE SCHEMA ON SCHEMA bronze TO `de_group`;
GRANT USE CATALOG ON CATALOG dev TO `de_group`;
```

✅ **Why:**  
Ensures engineers can read from a specific table while keeping others restricted.

---

### Step 6: Optional — Browse Privileges

`BROWSE` privileges allow users to **see objects** without being able to use or modify them.

```sql
GRANT BROWSE ON CATALOG dev TO `de_group`;
```

✅ **Why:**  
Useful when users should be able to view metadata or lineage but not access data directly.

---

## ⚙️ 6. Understanding Privilege Types

| Privilege | Description | Applies To |
|------------|--------------|------------|
| `SELECT` | Read table data | Tables, Views |
| `MODIFY` | Update, insert, or delete table data | Tables |
| `READ VOLUME` | Read files from a volume | Volumes |
| `WRITE VOLUME` | Write files into a volume | Volumes |
| `EXECUTE` | Run a function | Functions |
| `CREATE` | Create new objects (tables, schemas, functions) | Schema, Catalog |
| `USE SCHEMA` | Access schema contents | Schema |
| `USE CATALOG` | Access catalog contents | Catalog |
| `BROWSE` | View metadata, lineage only | Catalogs, Schemas |
| `MANAGE` | Full owner-like privileges (drop, rename, manage permissions) | All securables |

⚠️ **Important:**  
The `MANAGE` privilege is equivalent to ownership — use it carefully. It allows renaming, dropping, and modifying permissions.

---

## 🧠 7. Best Practices for Managing Object-Level Security

1. **Follow Principle of Least Privilege** — Grant only what’s required.  
2. **Use Groups Instead of Individual Users** — Easier to scale and maintain.  
3. **Prefer Top-Down for Bulk Access** — e.g., for analyst teams needing read access to multiple datasets.  
4. **Use Bottom-Up for Sensitive Data** — Restrict access at table or volume level.  
5. **Avoid Granting Manage Privileges** — Unless necessary for administrative tasks.  
6. **Automate with SQL Scripts or SCIM** — To synchronize and enforce consistent permissions.  

---

## 🧾 8. Summary

| Approach | Use Case | Example |
|-----------|-----------|----------|
| **Top-Down** | Bulk privileges for groups or roles | Grant `SELECT` on entire schema or catalog |
| **Bottom-Up** | Specific privileges for restricted users | Grant `SELECT` on individual table |
| **Metastore Level** | Control catalog creation rights | Grant `CREATE CATALOG` |
| **Browse Access** | Metadata-only visibility | Grant `BROWSE` |

---

## 🧩 9. SQL Commands Reference

You can manage privileges via the Databricks UI or SQL commands. Below are common operations:

```sql
-- Show current permissions
SHOW GRANTS ON TABLE bronze.customer_raw;

-- Grant privilege
GRANT SELECT ON TABLE bronze.customer_raw TO `de_group`;

-- Revoke privilege
REVOKE SELECT ON TABLE bronze.customer_raw FROM `de_group`;

-- Grant all privileges
GRANT ALL PRIVILEGES ON SCHEMA bronze TO `da_group`;
```

---

## 🏁 10. Conclusion

By following the steps above, you can effectively implement **object-level security in Unity Catalog**.  
This ensures data access is **consistent, auditable, and secure**, allowing you to balance collaboration with governance.

> **Next Step:** In the upcoming section, we will cover *function-level permissions* in Unity Catalog.

---
