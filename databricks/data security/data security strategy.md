# Data Security in Databricks

## **Problem Statement**
The platform should support various levels of data access depending on dataset sensitivity and downstream user roles or purposes.

---

## **Solution**
Data security in Databricks can be implemented at three primary levels — **Object-Level**, **Row-Level**, and **Column-Level**.  
The choice of approach depends on the granularity and sensitivity of the data access restrictions required.

### **1. Object-Level Permissions**
Object-level permissions control access to securable objects such as catalogs, schemas, and tables.  
There are two main approaches:  
- **Top-Down Approach:** Access is granted from the top level (catalog) and inherited by lower levels (schema, tables).  
- **Bottom-Up Approach:** Access is granted starting at the lowest level (table or view), providing more fine-grained control.  

The **Bottom-Up Approach** is generally preferred when precise access control is required.

---

### **2. Row-Level Filters / Security**
Row-level filters define access control at the data row level, allowing users to see only the data relevant to their role or region.  
For example, a user from the UK region can be restricted to view only UK-related records in a global dataset.  
This can be achieved using **Databricks Functions** and by setting **Row Filter Constraints** on the table.

---

### **3. Column-Level Masking**
Column-level masking allows sensitive data within a column to be hidden or obfuscated for specific users, without removing the column or creating separate views.  
By defining a **masking function** and applying it as a **table-level constraint**, sensitive fields (such as personal identifiers, salaries, or phone numbers) can be dynamically masked based on user privileges.

---

### **Summary**
| Security Type | Purpose | Example Use Case | Implementation Method |
|----------------|----------|------------------|-----------------------|
| **Object-Level Permissions** | Restrict access to catalogs, schemas, and tables | Give only analysts access to SQL warehouse | Unity Catalog permission grants |
| **Row-Level Security** | Restrict which rows a user can see | Show only UK data to UK users | Databricks Functions + Row Filter Constraints |
| **Column-Level Masking** | Mask or hide sensitive data dynamically | Mask phone numbers for non-admin users | Databricks Masking Function + Column Constraint |
