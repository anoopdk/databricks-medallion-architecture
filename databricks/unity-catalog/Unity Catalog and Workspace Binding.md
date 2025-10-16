# Unity Catalog and Workspace Binding — Detailed Guide

## 1. Introduction

**Workspace–Catalog Binding** is a Unity Catalog feature in Databricks that allows you to **control which workspaces can access which catalogs**.  
This is especially useful when multiple environments (e.g., Dev, QA, Prod) share a single **Unity Catalog metastore**, but you want to keep their data logically separated or partially shared.

By default, all catalogs within a metastore are accessible from all workspaces connected to that metastore. Workspace binding lets you restrict that visibility to only specific workspaces.

---

## 2. Why Workspace–Catalog Binding is Important

| Use Case | Benefit |
|-----------|----------|
| Environment Isolation | Keep Dev catalogs hidden from QA and Prod users |
| Security | Limit sensitive data catalogs to specific workspaces |
| Governance | Centralized metadata management via a single metastore |
| Collaboration | Create shared catalogs for cross-environment access (e.g., reference data) |

---

## 3. Prerequisites

Before configuring bindings, ensure the following:

1. **Same Metastore:** All workspaces (Dev, QA, etc.) must be attached to the same Unity Catalog metastore.
2. **Same Identity Provider:** Workspaces should exist within the same Azure AD (or same tenant).
3. **Admin Permissions:** You need account admin or metastore admin rights.
4. **Catalog-level only:** Bindings apply only at the *catalog* level (not schema or table).

---

## 4. Architecture Overview

```text
+-----------------------------------------------------------+
|                     Unity Catalog Metastore               |
|-----------------------------------------------------------|
|  Catalogs: dev, qa, prod, common                          |
|  Bindings:                                                |
|     - dev     -> Dev Workspace                            |
|     - qa      -> QA Workspace                             |
|     - prod    -> Prod Workspace                           |
|     - common  -> Dev + QA + Prod                          |
+-----------------------------------------------------------+
       ↑                  ↑                  ↑
   Dev Workspace     QA Workspace       Prod Workspace
```

---

## 5. Step-by-Step Implementation

### Step 1 — Confirm Metastore Connection
1. Go to **Account Console → Metastores**.
2. Verify both Dev and QA workspaces are connected to the same metastore (e.g., *Azure Central India*).
3. In each workspace, click the ⚙️ icon → **Settings → Unity Catalog** → confirm metastore name.

---

### Step 2 — Verify Catalog Visibility
1. In **Dev Workspace**, open **Data → Catalogs**.
2. Observe available catalogs (e.g., `dev`, `qa`, `common`).
3. Switch to **QA Workspace** and check catalogs — you’ll see all of them since both share the same metastore.

---

### Step 3 — Restrict Catalogs to Specific Workspaces
Example: Restrict `dev` catalog to **Dev workspace only**.

1. Open **Dev Workspace** → **Data → Catalogs → dev**.  
2. Go to the **Workspace Binding** tab.  
3. By default, it says: `Available to: All workspaces`.  
4. Uncheck **All workspaces**.  
5. Click **Assign to workspace** → select only *Dev workspace*.  
6. Save.

Repeat the same process for other environment-specific catalogs (e.g., `dev_external`, `dev_test` etc.).

---

### Step 4 — Validate from QA Workspace
1. Switch to **QA workspace** → **Data → Catalogs**.  
2. Refresh the list — `dev` and `dev_external` should now be hidden.  
3. Log in as a non-admin (like a QA data engineer) to double-check catalog visibility.

---

### Step 5 — Create a QA-Only Catalog
1. From **QA workspace**, click **+ Create Catalog → qa**.  
2. Go to its **Workspace Binding**.  
3. Uncheck **All workspaces** → assign only *QA workspace*.  
4. Save.  
5. Verify: in Dev workspace, the `qa` catalog should now be invisible.

---

### Step 6 — Create a Common Shared Catalog
1. In **Dev workspace**, click **+ Create Catalog → common**.  
2. Leave binding as **All workspaces** (default).  
3. This catalog will now be visible in Dev, QA, and any other workspace connected to the same metastore.  
4. Use this for shared objects (e.g., reference tables, volumes, or common views).

---

### Step 7 — Manage Permissions
Binding controls *visibility* only. To control *access*, use Unity Catalog **permissions**.

Example:
1. Go to `common` catalog → **Permissions → Grant**.  
2. Assign privileges to group `data_engineers`:  
   - USE CATALOG  
   - USE SCHEMA  
   - SELECT  
3. Remove default `All account users` permissions.  
4. Save and verify as a member of the `data_engineers` group.

---

### Step 8 — Validation Scenarios

| User | Workspace | Expected Catalogs |
|------|------------|------------------|
| Admin | Dev | `dev`, `common` |
| Admin | QA | `qa`, `common` |
| Data Engineer | Dev | `dev`, `common` |
| Data Engineer | QA | `qa`, `common` |
| Business Analyst | QA | `common` only |

---

## 6. Key Notes

- **System Catalogs:** `system` and `databricks_internal` will always appear; you can restrict object access using privileges.
- **Binding granularity:** Only possible at *catalog* level. Use grants/revokes for schema/table-level control.
- **Propagation:** Changes take a few seconds to reflect. Refresh UI or re-login.
- **Audit:** Unity Catalog audit logs record binding and permission changes.

---

## 7. Best Practices

✅ Use clear naming conventions: `dev`, `qa`, `prod`, `common`.  
✅ Use group-based access (SCIM-managed from Azure AD).  
✅ Always revoke `All account users` to enforce least privilege.  
✅ Maintain an environment-to-catalog matrix (for documentation).  
✅ Test bindings using non-admin users.  
✅ Automate catalog creation and binding with Terraform or Databricks Account API.

---

## 8. Automation (Optional)

You can automate workspace binding using the **Databricks Account API**:

Example API call:
```bash
PATCH /api/2.1/unity-catalog/catalogs/{catalog_name}
{
  "workspace_binding": {
    "workspaces": ["<workspace-id-1>", "<workspace-id-2>"]
  }
}
```

This can be incorporated into your Terraform scripts or CI/CD pipelines for environment provisioning.

---

## 9. Governance Checklist

| Item | Description | Verified |
|------|--------------|-----------|
| Same metastore for all workspaces | Ensures catalogs can be shared | ☐ |
| Environment-specific catalogs created | dev / qa / prod | ☐ |
| Workspace binding configured | Catalogs mapped properly | ☐ |
| Shared catalog (`common`) configured | For reusable datasets | ☐ |
| Default account users revoked | Enforces least privilege | ☐ |
| Group permissions granted | For data engineer / analyst roles | ☐ |
| Validation done with non-admin user | Visibility confirmed | ☐ |
| Audit logs reviewed | Governance verified | ☐ |

---

## 10. Conclusion

Workspace–Catalog Binding in Unity Catalog provides a clean and secure way to separate data across environments while keeping metadata centralized.  
It strikes the right balance between isolation and collaboration — giving you both governance and flexibility within your data platform.
