
# User Management in Databricks Using Azure Entra ID (Comprehensive Guide)

## 📘 Overview
This guide explains how to manage users, groups, and service principals in Databricks using **Azure Entra ID (formerly Azure AD)** and the **Databricks Account Console**.  
It includes detailed steps, the reasoning behind each action, and best practices for secure and scalable access management.

---

## 🔹 1. Understanding the Components

Before we begin, it’s important to understand the key layers involved:

| Layer | Purpose | Admin Role Needed |
|-------|----------|------------------|
| **Azure Entra ID (Microsoft Entra)** | Central identity management – users, groups, service principals | *Cloud Admin* |
| **Databricks Account Console** | Global control plane for users, groups, and workspace assignments | *Account Admin* |
| **Databricks Workspace** | Environment where users actually work (SQL, ML, Data Eng) | *Workspace Admin* |

👉 You need **three roles** to complete this setup:
1. **Cloud Admin access** – to create users and service principals in Azure Entra ID.  
2. **Account Admin access** – to manage them at the Databricks account level.  
3. **Workspace Admin access** – to assign permissions within individual Databricks workspaces.

---

## 🔹 2. Creating Users in Azure Entra ID

### Step 1: Open Azure Entra ID
1. Log in to your **Azure Portal**.  
2. Navigate to **Microsoft Entra ID** from the left-hand menu.

### Step 2: Add a New User
1. Click **Add → User → Create new user**.  
2. Provide:
   - **User Principal Name (UPN):** Format `username@domain.com`.  
     - Example: `dataanalyst@xyz.com`
   - **Display Name:** Example – “Data Analyst”.
   - **Password:** Set manually so you can log in later.
   - Ensure the **“Account enabled”** box is checked.

3. Click **Review + Create → Create**.

🧭 **Why:**  
All users in Databricks are ultimately authenticated via Entra ID. This ensures single sign-on (SSO), centralized password management, and compliance.

---

## 🔹 3. Adding Users to Databricks Account Console

Once users exist in Entra ID, they must be added to the Databricks **Account Console** to use any workspace.

### Step 1: Open Databricks Account Console
- Go to [account.databricks.com](https://account.databricks.com) (or your enterprise URL).
- Select **User Management** in the left panel.

### Step 2: Add the User
- Under **Users**, click **Add User**.  
- Paste the **User Principal Name (UPN)** from Azure Entra ID.  
- Click **Add User**.

✅ The user is now visible in your Databricks account.

🧭 **Why Manual Add?**  
This is a manual provisioning method. For enterprises, use **SCIM integration** to automate provisioning.

---

## 🔹 4. Automating User Provisioning (Recommended)

You can configure **SCIM (System for Cross-domain Identity Management)** between **Azure Entra ID and Databricks**.

### Benefits of SCIM:
- Automatically syncs user and group changes from Entra ID.  
- Eliminates the need to manually add or remove users.  
- Keeps Databricks account roles consistent with organizational roles.

### Reference:
Documentation link (Databricks official):  
🔗 *Configure SCIM provisioning with Microsoft Entra ID*

🧭 **When to Use:**  
If your organization uses centralized identity governance, **always enable SCIM**.  
Manual addition should only be used in POCs or learning environments.

---

## 🔹 5. Creating and Managing Groups

Groups allow you to manage permissions collectively instead of user-by-user.

### Step 1: Create a Group
1. Go to the **Groups** tab in Account Console.  
2. Click **Add Group** and name it logically – e.g., `grp_data_analyst`.  
3. Save the group.

### Step 2: Add Members
1. Click the **group name → Add Members**.  
2. Search for your user (e.g., *Data Analyst*).  
3. Click **Add**.

🧭 **Why Use Groups:**  
Instead of assigning workspace permissions individually, assign them to a **group**.  
All users added later will **inherit the same permissions** automatically.

💡 *Best Practice:*  
Create groups aligned to job roles — e.g., `grp_data_engineer`, `grp_data_analyst`, `grp_ml_team`.

---

## 🔹 6. Assigning Groups or Users to Workspaces

### Step 1: Go to Workspace Permissions
- In Account Console, go to **Workspaces → Select Workspace → Permissions tab**.

### Step 2: Add Permissions
- Click **Add Permissions**.  
- Search for your **group** (`grp_data_analyst`) or a **user**.  
- Choose their **role**:
  - **User:** Standard access to workspace.  
  - **Workspace Admin:** Full control of workspace (avoid for general users).

🧭 **Which to Choose:**  
- Use **User** for analysts and engineers.  
- Use **Workspace Admin** sparingly — typically only for DevOps or Platform teams.

---

## 🔹 7. Setting Permissions Inside the Workspace

Now that users can access the workspace, you need to control *what they can do inside it.*

### Step 1: Log in as Workspace Admin
1. Open the workspace.  
2. Click your **profile icon → Settings → Identity and Access Management**.

### Step 2: Manage Group Permissions
- Go to **Groups → Manage**.  
- Locate your group (e.g., `grp_data_analyst`).  
- Click the **three dots → Edit → Entitlements**.

### Step 3: Configure Entitlements
Enable or disable:
- **Workspace Access:** General access (for Data Eng and ML tabs).  
- **Databricks SQL Access:** For SQL analysts.  
- **Cluster Create:** Allows cluster creation (use cautiously).

🧭 **When to Choose What:**
| Role | Workspace Access | Databricks SQL | Cluster Create |
|------|------------------|----------------|----------------|
| **Data Analyst** | ❌ | ✅ | ❌ |
| **Data Engineer** | ✅ | ❌ | ✅ |
| **ML Engineer** | ✅ | ✅ | ✅ |

🧠 **Tip:**  
Disable cluster creation for analysts — use shared clusters or SQL warehouses.

---

## 🔹 8. Creating Service Principals (Automation Accounts)

Service Principals are **non-human identities** used for:
- Running jobs and pipelines.
- Authenticating API calls or CLI commands.
- Ensuring automation continuity even if users leave.

### Two Types of Service Principals
| Type | Managed By | Use Case |
|------|-------------|----------|
| **Databricks Managed** | Databricks itself | Quick setup for internal automation |
| **Microsoft Entra ID Managed** | Azure Entra ID | Enterprise-grade security, SSO, policy control |

### Step 1: Create Databricks-Managed SP
1. In Account Console → **Service Principals → Add**.  
2. Choose **Databricks Managed**.  
3. Name it logically (e.g., `sp_databricks_automation`).  
4. Click **Add**.

### Step 2: Create Entra ID Managed SP
1. In Azure Entra ID → **App Registrations → New Registration**.  
2. Name it (e.g., `sp_azure_databricks`).  
3. Click **Register**.  
4. Copy the **Application (Client) ID**.

5. Go back to Databricks Account Console → **Add Service Principal → Microsoft Entra ID Managed**.  
6. Paste the **Application ID** and the **Display Name**.  
7. Click **Add**.

🧭 **Which One to Use:**
- For production or CI/CD — use **Entra ID Managed** (secure, auditable, integrates with policies).
- For internal dev or quick tests — **Databricks Managed** is fine.

---

## 🔹 9. Assign Service Principals to Workspaces

1. Go to **Account Console → Workspaces → Permissions tab**.  
2. Click **Add Permission → Search SP name** (e.g., `sp_azure_databricks`).  
3. Set the role to **User** (not admin).

Then inside the workspace, under **Identity and Access → Service Principals**, assign the required entitlements (workspace access, SQL, cluster create, etc.).

---

## 🔹 10. Using Service Principals for Job Automation

In **Workflows → Jobs**, you can choose a “Run As” identity.

- Click **Edit (pencil icon)** next to *Run As User*.  
- Search and select your **Service Principal**.  
- Save.

✅ This ensures jobs are run under a service principal instead of a human user.

🧭 **Why:**  
- Jobs continue to run even if a user leaves the organization.  
- Improves auditability and security.

---

## 🔹 11. Key Differences Between Roles

| User Type | Access Scope | Admin Rights | Typical Use |
|------------|---------------|--------------|--------------|
| **Account Admin** | Entire Databricks account | Full | Create workspaces, assign admins |
| **Workspace Admin** | Single workspace | Full (within workspace) | Configure workspace, add users |
| **Normal User** | Workspace-level | None | Develop, analyze, run notebooks |
| **Service Principal** | Automation | Controlled | Run jobs, APIs, pipelines |

---

## 🔹 12. Best Practices Summary

✅ Create **users and groups in Entra ID**, not Databricks manually.  
✅ Use **SCIM** for auto-provisioning.  
✅ Assign **permissions via groups** rather than individuals.  
✅ Use **service principals** for all automation and CI/CD.  
✅ Restrict **cluster creation** to engineers or admins only.  
✅ Use **role-based groups** (Analyst, Engineer, ML, Admin).  
