# 🔒 Implement Security and Governance in Workspace

> **DP-600 Exam Domain:** Maintain a Data Analytics Solution (25–30%)  
> **Subtopic:** Implement Security and Governance  
> **Skills Measured (as of July 21, 2026)**

---

## 📋 Table of Contents

- [Exam Objectives Breakdown](#-exam-objectives-breakdown)
- [Microsoft Fabric Security Architecture](#-microsoft-fabric-security-architecture)
- [1. Workspace-Level Access Controls](#1-implement-workspace-level-access-controls)
- [2. Item-Level Access Controls](#2-implement-item-level-access-controls)
- [3. Row-Level, Column-Level, Object-Level & File-Level Security](#3-implement-row-level-column-level-object-level-and-file-level-access-control)
- [4. Sensitivity Labels](#4-apply-sensitivity-labels-to-items)
- [5. Endorsement](#5-endorse-items)
- [Governance with Microsoft Purview](#-governance-with-microsoft-purview)
- [Practice Questions](#-practice-questions)
- [Resources & References](#-resources--references)

---

## 📌 Exam Objectives Breakdown

| # | Objective | Complexity |
|---|-----------|------------|
| 1 | Implement workspace-level access controls | ⭐⭐ |
| 2 | Implement item-level access controls | ⭐⭐⭐ |
| 3 | Implement row-level, column-level, object-level, and file-level access control | ⭐⭐⭐⭐ |
| 4 | Apply sensitivity labels to items | ⭐⭐ |
| 5 | Endorse items | ⭐⭐ |

---

## 🏗️ Microsoft Fabric Security Architecture

Fabric uses a **layered security model**:

```
┌─────────────────────────────────────────────┐
│           Tenant/Entra ID Level             │  ← Admin controls, tenant settings
├─────────────────────────────────────────────┤
│           Workspace Level                   │  ← Workspace roles (Admin/Member/etc.)
├─────────────────────────────────────────────┤
│           Item Level                        │  ← Share permissions, item roles
├─────────────────────────────────────────────┤
│     Data Access Level (RLS/CLS/OLS)         │  ← Row/Column/Object security
├─────────────────────────────────────────────┤
│           OneLake / File Level              │  ← ADLS Gen2, shortcut permissions
└─────────────────────────────────────────────┘
```

**Key principle:** Each layer is additive. A user needs access at EVERY layer to view data.

---

## 1. Implement Workspace-Level Access Controls

### 🔑 Workspace Roles

| Role | Description | Typical Use |
|------|-------------|-------------|
| **Admin** | Full control: manage roles, delete workspace, publish apps | Workspace owners |
| **Member** | Create/edit items, manage item permissions, publish apps | Power users / developers |
| **Contributor** | Create/edit items (cannot manage workspace members or publish apps) | Developers |
| **Viewer** | View items only (read-only access) | Business users, report consumers |

### 📊 Role Permissions Matrix

| Permission | Admin | Member | Contributor | Viewer |
|-----------|-------|--------|-------------|--------|
| Manage workspace access | ✅ | ✅ | ❌ | ❌ |
| Delete workspace | ✅ | ❌ | ❌ | ❌ |
| Create/edit items | ✅ | ✅ | ✅ | ❌ |
| View items | ✅ | ✅ | ✅ | ✅ |
| Publish app | ✅ | ✅ | ❌ | ❌ |
| Schedule refresh | ✅ | ✅ | ✅ | ❌ |
| Share items | ✅ | ✅ | ❌ | ❌ |
| Endorse items | ✅ | ✅ | ❌ | ❌ |
| Build permission (datasets) | ✅ | ✅ | ✅ | ❌ |

### ⚙️ Assigning Workspace Access

**Via Fabric Portal:**
1. Open the workspace
2. Click **Workspace settings** → **Manage access**
3. Add users/groups → assign role

**Via PowerShell:**
```powershell
Add-PowerBIWorkspaceUser `
    -WorkspaceId "workspace-guid" `
    -UserEmailAddress "user@company.com" `
    -UserAccessRight "Contributor"  # Admin, Member, Contributor, Viewer
```

**Via REST API:**
```http
POST https://api.fabric.microsoft.com/v1/workspaces/{workspaceId}/roleAssignments

{
  "principal": {
    "id": "user-object-id",
    "type": "User"
  },
  "role": "Contributor"
}
```

### 🏢 Best Practice: Use Security Groups
Assign workspace roles to **Entra ID (Azure AD) security groups**, not individual users:
- ✅ Easier management at scale
- ✅ Consistent access control
- ✅ Auditable group membership
- ✅ Reduced administrative overhead

### ⚠️ Key Limitations
- Workspace Viewer role grants access to **all items** in the workspace
- For granular per-item access: use **item-level permissions**
- Workspace roles do NOT automatically grant access to underlying data sources

---

## 2. Implement Item-Level Access Controls

### 🔑 What is Item-Level Access?

Item-level permissions allow sharing **specific items** with users who don't have workspace access.

### 📋 Semantic Model (Dataset) Permissions

| Permission | Description |
|-----------|-------------|
| **Read** | Can query the dataset via Power BI Service |
| **Build** | Can create new content (reports) using the dataset |
| **Write** | Can edit the dataset definition |
| **Reshare** | Can share the dataset with others |

> 🎯 **Exam Focus:** **Build** permission is critical for the shared semantic model pattern — allows analysts to create reports on top of certified datasets without workspace access.

### 📋 Lakehouse Permissions

| Permission | Access |
|-----------|--------|
| **Read** | Read data from lakehouse |
| **ReadAll** | Read all data including Delta tables |
| **ReadData** | Read data via SQL endpoint |
| **Write** | Write data to lakehouse |

### 🔗 Item Sharing Pattern

```
Workspace: No access
↓
Item Share: Specific Report + View permission
↓
User can: View that report ONLY (not other workspace items)
```

**Share via REST API:**
```http
POST https://api.fabric.microsoft.com/v1/workspaces/{workspaceId}/items/{itemId}/permissions

{
  "principal": {
    "id": "user-id",
    "type": "User"
  },
  "permissions": ["Read", "Build"]
}
```

---

## 3. Implement Row-Level, Column-Level, Object-Level, and File-Level Access Control

### 3a. Row-Level Security (RLS)

RLS restricts which **rows** a user can see in a table, based on their identity.

#### Types of RLS in Fabric

| Type | Location | Technology |
|------|---------|------------|
| **RLS in Semantic Model** | Power BI semantic model | DAX filter rules |
| **RLS in Warehouse/Lakehouse** | SQL analytics endpoint | T-SQL predicates |

#### Semantic Model RLS — Static RLS

```dax
// In Power BI Desktop → Manage Roles → New Role "Sales_East"
// DAX filter on Sales table:
[Region] = "East"
```

#### Semantic Model RLS — Dynamic RLS (Recommended for Enterprise)

Dynamic RLS uses the logged-in user's email to filter data:

```dax
// UserRegion mapping table:
// | UserEmail           | Region |
// |---------------------|--------|
// | alice@company.com   | East   |
// | bob@company.com     | West   |

// DAX filter on Sales table:
[Region] = LOOKUPVALUE(
    UserRegion[Region],
    UserRegion[UserEmail], USERPRINCIPALNAME()
)
```

**Assign users to RLS roles (Fabric Portal):**
```
Semantic Model → Manage permissions → Security → Add users to role
```

#### SQL-Level RLS (Warehouse / Lakehouse SQL Endpoint)

```sql
-- Create a security predicate function
CREATE FUNCTION dbo.fn_RegionFilter(@Region VARCHAR(50))
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN SELECT 1 AS fn_Result
    WHERE @Region = (
        SELECT Region FROM dbo.UserRegionMapping
        WHERE UserEmail = USER_NAME()
    );

-- Apply security policy
CREATE SECURITY POLICY SalesFilter
    ADD FILTER PREDICATE dbo.fn_RegionFilter(Region)
    ON dbo.Sales
    WITH (STATE = ON);
```

### 3b. Column-Level Security (CLS)

CLS restricts specific **columns** from being accessed by certain users.

#### In Warehouse: Column-Level Security via GRANT/DENY

```sql
-- Grant SELECT on specific columns only
GRANT SELECT ON dbo.Employees (EmployeeID, Name, Department)
    TO [AnalystGroup];

-- DENY access to sensitive column
DENY SELECT ON dbo.Employees (Salary, SSN)
    TO [AnalystGroup];
```

### 3c. Object-Level Security (OLS)

OLS makes entire **tables or columns invisible** to unauthorized roles in the semantic model.

**Key distinction:**
```
RLS: User can SEE the table, but rows are filtered
OLS: User CANNOT see the table at all (not in Fields pane)
```

**Configure via Tabular Editor:**
```json
// OLS configuration in BIM file
{
  "roles": [{
    "name": "StandardAnalyst",
    "tablePermissions": [{
      "name": "HR_Salary_Data",
      "metadataPermission": "None"  // Table completely hidden
    }]
  }]
}
```

**Configure per column (column OLS):**
```json
{
  "roles": [{
    "name": "AnalystRole",
    "tablePermissions": [{
      "name": "Employees",
      "columnPermissions": [{
        "name": "Salary",
        "metadataPermission": "None"
      }]
    }]
  }]
}
```

### 3d. File-Level Security in OneLake

#### OneLake Access Control Methods

| Method | Use Case |
|--------|---------|
| **Workspace roles** | Broad access to all items in workspace |
| **OneLake data access roles** | Granular access to specific folders/files |
| **Shortcut permissions** | Control access via shortcut source permissions |
| **ADLS Gen2 ACLs** | File/folder-level permissions for external access |

#### OneLake Data Access Roles (Folder-level Security)

```
Setup:
1. Workspace → Lakehouse → Manage OneLake data access
2. Create a role: "Finance_Readers"
3. Add folder path: /Finance/Invoices/
4. Assign users/groups to the role

Available permissions:
- Reader: Can read files in assigned paths
- ReadWrite: Can read and write files
```

#### File-Level Security via POSIX ACLs

```python
from azure.storage.filedatalake import DataLakeServiceClient

service_client = DataLakeServiceClient(
    account_url="https://{account}.dfs.core.windows.net",
    credential=credential
)

file_client = service_client.get_file_client(
    file_system="fabricworkspace",
    file_path="Sales/2024/sales_data.parquet"
)

# Set ACL: User object ID with read-only access
file_client.set_access_control(
    acl="user:{object-id}:r--,group::r--,other::---"
)
```

---

## 4. Apply Sensitivity Labels to Items

### 🔑 What are Sensitivity Labels?

Sensitivity labels (from Microsoft Purview / Microsoft Information Protection) classify and protect Fabric items based on data sensitivity.

### 🏷️ Common Label Tiers

| Label | Typical Use |
|-------|-------------|
| **Public** | Non-sensitive, shareable externally |
| **General** | Internal, not sensitive |
| **Confidential** | Business-sensitive data |
| **Highly Confidential** | PII, financial, legal data |

### ⚙️ Applying Labels in Fabric

**Manual label assignment:**
1. Open item (report, dataset, lakehouse, etc.)
2. Click the **sensitivity label icon** (lock/shield icon in toolbar)
3. Select the appropriate label

**Via PowerShell (bulk apply):**
```powershell
Install-Module -Name MicrosoftPowerBIMgmt

$datasets = Get-PowerBIDataset -WorkspaceId "workspace-id"

foreach ($dataset in $datasets) {
    Set-PowerBIDataset `
        -Id $dataset.Id `
        -WorkspaceId "workspace-id" `
        -SensitivityLabelId "label-guid"
}
```

### 📊 Label Inheritance

Sensitivity labels propagate downstream:
```
Semantic Model (Highly Confidential)
    ↓ inherits
Report (Highly Confidential)
    ↓ exports to Excel
Excel file (encrypted, Highly Confidential label)
```

### ⚙️ Admin Configuration Requirements
- Labels configured in **Microsoft Purview Compliance Center**
- Tenant admin must enable "Apply sensitivity labels to content" in Fabric admin portal
- Users must have appropriate Purview licenses (Microsoft 365 E3/E5 or equivalent)

---

## 5. Endorse Items

### 🔑 What is Endorsement?

Endorsement signals the **quality and trustworthiness** of Fabric items to help users find reliable data.

### 📊 Endorsement Levels

| Level | Who Can Set | Meaning |
|-------|------------|---------|
| **Promoted** | Workspace Member or Admin | Item is ready for broad use, tested |
| **Certified** | Tenant-designated certifiers only | Meets organizational standards, governance-approved |

### ⚙️ Setting Endorsement

**Promoted endorsement:**
1. Open the item → Click **⋯ (More options)**
2. Select **Settings** → **Endorsement** tab
3. Select **Promoted**

**Certified endorsement:**
1. Tenant admin designates certifiers: Fabric Admin Portal → **Tenant settings → Certification**
2. Certifiers mark items as Certified via item settings

**Via REST API:**
```http
PATCH https://api.fabric.microsoft.com/v1/workspaces/{workspaceId}/items/{itemId}

{
  "endorsementDetails": {
    "endorsement": "Certified"
  }
}
```

### 📋 Endorsement vs Sensitivity Labels

| | Endorsement | Sensitivity Label |
|--|------------|------------------|
| **Purpose** | Data quality signal | Data protection classification |
| **Set by** | Content owners/certifiers | Users/admins |
| **Technical protection** | None (informational only) | Can enforce encryption |
| **Source** | Microsoft Fabric | Microsoft Purview |

---

## 🏛️ Governance with Microsoft Purview

### Microsoft Purview Hub in Fabric

Available at: **Fabric Workspace → Governance (left sidebar) → Purview hub**

**Features:**
- **Data catalog**: Discover and search endorsed items
- **Sensitivity label reports**: See label coverage across items
- **Data lineage**: Trace data flow from source → transformation → report
- **Information protection**: View label application status

### Data Lineage View

```
Fabric Portal → Workspace → Lineage (view switcher)

Shows: Data sources → Dataflows → Lakehouses → Semantic Models → Reports → Dashboards
```

---

## ❓ Practice Questions

**Q1:** A user needs to view reports in a workspace but should NOT be able to create or edit any content. Which workspace role is correct?
- A) Contributor
- B) Member
- C) **Viewer** ✅
- D) Reader

---

**Q2:** Analysts need to create reports using a certified semantic model, but should NOT have workspace access. What permission on the semantic model should you grant?
- A) Write
- B) Read
- C) **Build** ✅
- D) Reshare

> **Explanation:** Build permission allows users to create reports from the dataset without workspace membership.

---

**Q3:** A `Salary` column should be completely invisible to the "Junior Analyst" role — not even appearing in the Fields pane. Which security feature achieves this?
- A) Row-Level Security (RLS)
- B) Column-Level Security (CLS) with DENY
- C) **Object-Level Security (OLS)** ✅
- D) Sensitivity Labels

---

**Q4:** A manager wants a dataset to appear with a special quality badge in the OneLake catalog indicating organizational standards compliance. What should be applied?
- A) Sensitivity label = Confidential
- B) **Certified endorsement** ✅
- C) Promoted endorsement
- D) Workspace role = Admin

---

**Q5:** You are implementing dynamic RLS. Which DAX function returns the currently logged-in user's identity?
- A) `USERNAME()`
- B) `CURRENTUSER()`
- C) **`USERPRINCIPALNAME()`** ✅
- D) `USERACCOUNT()`

---

**Q6:** Which combination is required for a user to see a report powered by a Direct Lake semantic model with RLS applied?
- A) Workspace Viewer role only
- B) **Report Read permission + Semantic Model Read permission + assigned to RLS role** ✅
- C) Report Build permission + Semantic Model Write permission
- D) Workspace Member role + Semantic Model Read permission

---

## 📚 Resources & References

| Resource | Link |
|----------|------|
| Workspace roles in Fabric | [learn.microsoft.com/fabric/security/permission-model](https://learn.microsoft.com/fabric/security/permission-model) |
| Row-level security (Power BI) | [learn.microsoft.com/power-bi/enterprise/service-admin-rls](https://learn.microsoft.com/power-bi/enterprise/service-admin-rls) |
| Object-level security (OLS) | [learn.microsoft.com/analysis-services/tabular-models/object-level-security](https://learn.microsoft.com/analysis-services/tabular-models/object-level-security) |
| Sensitivity labels in Fabric | [learn.microsoft.com/fabric/governance/sensitivity-labels-overview](https://learn.microsoft.com/fabric/governance/sensitivity-labels-overview) |
| Endorsement in Fabric | [learn.microsoft.com/fabric/governance/endorsement-overview](https://learn.microsoft.com/fabric/governance/endorsement-overview) |
| OneLake data access roles | [learn.microsoft.com/fabric/onelake/security/get-started-data-access-roles](https://learn.microsoft.com/fabric/onelake/security/get-started-data-access-roles) |
| Purview hub in Fabric | [learn.microsoft.com/fabric/governance/use-microsoft-purview-hub](https://learn.microsoft.com/fabric/governance/use-microsoft-purview-hub) |
| Item permissions | [learn.microsoft.com/fabric/security/permission-model](https://learn.microsoft.com/fabric/security/permission-model) |

---

> 📌 **Study Tip:** Draw the security layer diagram (Tenant → Workspace → Item → Data → File) and practice applying each layer in a Fabric trial. The exam frequently tests which layer controls what, and when to use RLS vs OLS vs workspace roles.

---

*Last updated: September 2026 | DP-600 Skills measured as of July 21, 2026*
