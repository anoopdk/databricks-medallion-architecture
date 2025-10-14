# Column-Level Masking in Databricks

## Overview
Column-level masking in Databricks helps protect **sensitive or personally identifiable information (PII)** such as phone numbers, SSNs, or Aadhaar numbers.  
Instead of removing or permanently redacting data, column masking **dynamically hides** sensitive column values for specific users or groups while allowing privileged users (like Admin or HR) to view the real data.

This allows fine-grained access control without modifying the underlying data.

---

## Why Use Column-Level Masking?
- ✅ **Compliance:** Ensure protection of PII data per GDPR, HIPAA, or internal policies.  
- 🔒 **Dynamic masking:** Hide data dynamically without physically altering it.  
- 👥 **Granular access:** Allow only specific users or groups to see unmasked data.  
- 🧭 **Central governance:** Manage all masking policies centrally via Unity Catalog.

---

## Key Concepts
- **Masking Function:** Logic that redacts or transforms data based on user roles.  
- **Admin Users:** View actual sensitive data.  
- **Non-Admin Users:** See masked data (e.g., `***-***-****`).  
- **Scalar Functions:** Used to create masking logic since they return one value per input.  
- **`IS_MEMBER()` Function:** Built-in Databricks function to check group membership.

---

## Implementation Approaches

### **Method 1: Using a Custom Metadata Table**
In this method, you create a **custom metadata table** that maps users to their respective groups (Admin, User, etc.).  
The masking function queries this mapping to decide who sees the real data.

#### **Steps:**
1. **Create a metadata table:**
   ```sql
   CREATE TABLE user_groups (
       user_email STRING,
       group STRING
   );

   INSERT INTO user_groups VALUES
       ('edata_user@databricks.com', 'admin'),
       ('data_engineer@databricks.com', 'user');
   ```

2. **Create the masking function:**
   ```sql
   CREATE OR REPLACE FUNCTION phone_mask(call STRING)
   RETURNS STRING
   LANGUAGE SQL
   RETURN (
       CASE 
           WHEN (SELECT group FROM user_groups WHERE user_email = current_user()) = 'admin'
           THEN call
           ELSE '***-***-****'
       END
   );
   ```

3. **Apply the masking function to the table:**
   ```sql
   ALTER TABLE customer_raw
   ALTER COLUMN c_phone
   SET MASK phone_mask;
   ```

4. **Verify behavior:**
   - Admin users see actual phone numbers.  
   - Non-admin users see masked values (`***-***-****`).

5. **Remove masking if needed:**
   ```sql
   ALTER TABLE customer_raw
   ALTER COLUMN c_phone
   DROP MASK;
   ```

---

### **Method 2: Using Built-in `IS_MEMBER()` Function**
Databricks provides an in-built function **`IS_MEMBER()`** that checks whether a user belongs to a given group.  
This approach removes the need for a separate metadata table.

#### **Steps:**
1. **Create a masking function using `IS_MEMBER`:**
   ```sql
   CREATE OR REPLACE FUNCTION phone_mask_v2(call STRING)
   RETURNS STRING
   LANGUAGE SQL
   RETURN (
       CASE 
           WHEN IS_MEMBER('admins') THEN call
           ELSE '***-***-****'
       END
   );
   ```

2. **Apply the masking function:**
   ```sql
   ALTER TABLE customer_raw
   ALTER COLUMN c_phone
   SET MASK phone_mask_v2;
   ```

3. **Verify Results:**
   - Users in `admins` group see full phone numbers.  
   - Others see masked values.

---

## Comparison: Metadata Table vs. IS_MEMBER()

| Feature | Metadata Table | IS_MEMBER() |
|----------|----------------|-------------|
| Setup | Requires a separate mapping table | Uses existing Databricks/Azure AD groups |
| Maintenance | Needs manual updates for user changes | Automatically syncs with directory |
| Flexibility | Can define custom mappings | Limited to predefined workspace groups |
| Performance | Slight overhead | Better performance |
| Recommended For | Large orgs with complex role mappings | Simple org setups |

---

## Best Practices
- Use **Unity Catalog** to centrally manage masking functions.  
- Keep naming conventions consistent (e.g., `mask_email`, `mask_phone`).  
- Test masking behavior with both admin and non-admin users.  
- Apply masking only where required to avoid unnecessary compute overhead.  
- Maintain audit logs for who accessed masked vs. unmasked data.

---

## Summary
Column-level masking in Databricks provides **fine-grained access control** over sensitive data.  
You can achieve this either through:
- A **custom metadata table**, giving full control and flexibility, or  
- The built-in **`IS_MEMBER()`** function for a simpler, integrated setup.

This feature is crucial for **data governance, privacy, and compliance** in enterprise Databricks environments.
