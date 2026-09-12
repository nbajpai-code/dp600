# 🔄 Maintain the Analytics Development Lifecycle

> **DP-600 Exam Domain:** Maintain a Data Analytics Solution (25–30%)  
> **Subtopic:** Maintain Analytics Development Lifecycle  
> **Skills Measured (as of July 21, 2026)**

---

## 📋 Table of Contents

- [Exam Objectives Breakdown](#-exam-objectives-breakdown)
- [1. Configure Version Control for a Workspace](#1-configure-version-control-for-a-workspace)
- [2. Create and Manage Power BI Desktop Projects (.pbip)](#2-create-and-manage-a-power-bi-desktop-project-pbip)
- [3. Create and Configure Deployment Pipelines](#3-create-and-configure-deployment-pipelines)
- [4. Perform Impact Analysis](#4-perform-impact-analysis-of-downstream-dependencies)
- [5. Deploy and Manage Semantic Models via XMLA](#5-deploy-and-manage-semantic-models-using-xmla-endpoint)
- [6. Create Reusable Assets](#6-create-reusable-assets)
- [ALM Best Practices](#-alm-best-practices-for-fabric)
- [Practice Questions](#-practice-questions)
- [Resources & References](#-resources--references)

---

## 📌 Exam Objectives Breakdown

| # | Objective | Complexity |
|---|-----------|------------|
| 1 | Configure version control for a workspace | ⭐⭐⭐ |
| 2 | Create and manage a Power BI Desktop project (.pbip) | ⭐⭐⭐ |
| 3 | Create and configure deployment pipelines | ⭐⭐⭐ |
| 4 | Perform impact analysis of downstream dependencies | ⭐⭐ |
| 5 | Deploy and manage semantic models using XMLA endpoint | ⭐⭐⭐⭐ |
| 6 | Create reusable assets (.pbit, .pbids files, shared semantic models) | ⭐⭐ |

---

## 1. Configure Version Control for a Workspace

### 🔑 What is Fabric Git Integration?

Microsoft Fabric supports **native Git integration** connecting a Fabric workspace to an Azure DevOps Git repository:
- Source control for Fabric items (semantic models, reports, notebooks, etc.)
- Collaborative development workflows
- Branch-based development (dev/test/prod)
- Change history and rollback

### ⚙️ Supported Items for Git Sync

| Item Type | Git Support |
|-----------|------------|
| Semantic models (.pbip format) | ✅ |
| Reports (.pbip format) | ✅ |
| Notebooks | ✅ |
| Data pipelines | ✅ |
| Lakehouses (metadata) | ✅ |
| Warehouses (metadata) | ✅ |
| Dataflows Gen2 | ✅ |
| Dashboards | ❌ |

### ⚙️ Setting Up Git Integration

**Prerequisites:**
- Fabric workspace (Premium or Fabric capacity)
- Azure DevOps organization with a repository
- Tenant admin must enable: "Users can synchronize workspace items with their Git repositories"

**Step-by-step:**
```
1. Open Fabric workspace
2. Workspace settings → Git integration
3. Connect to:
   - Organization: [Your Azure DevOps org]
   - Project: [Your DevOps project]
   - Repository: [Your Git repo]
   - Branch: main (or your target branch)
   - Folder: /workspace-name/ (folder within repo)
4. Click "Connect and sync"
```

### 📁 Git Repository File Structure

```
/fabric-workspace/
├── .platform/                      ← Workspace metadata
│   └── config.json
├── Sales Report.Report/            ← Report folder
│   ├── definition.pbir             ← Report definition
│   └── report.json
├── Sales Model.SemanticModel/      ← Semantic model folder
│   ├── definition/
│   │   ├── model.bim               ← Tabular model JSON
│   │   └── tables/
│   │       └── Sales.json
│   └── .platform
└── Sales Notebook.Notebook/
    └── notebook-content.py
```

### 🌿 Branching Strategy for Fabric

```
feature/new-report → dev → staging → main
     ↑                                 ↓
Developer workspace           Production workspace
```

- Each developer works in a personal workspace connected to their **feature branch**
- Pull Requests (PRs) merge: feature → dev → staging → main
- Each environment (dev/test/prod) maps to a Fabric workspace on its branch

### 🔄 Sync Operations

| Operation | Description | When to Use |
|-----------|-------------|------------|
| **Update all** | Pull remote changes to workspace | After PR merge, start of day |
| **Commit** | Push workspace changes to Git | After completing work |
| **Undo** | Revert uncommitted changes | Discard local changes |
| **Source** | Override workspace with Git version | Restore from source control |

### ⚡ Key Exam Points
- Git integration connects **one workspace to one branch** at a time
- Different workspaces can connect to **different branches** (enables dev/test/prod separation)
- Item deletion in Git removes item from workspace on next sync
- Service Principal authentication supported for CI/CD pipelines

---

## 2. Create and Manage a Power BI Desktop Project (.pbip)

### 🔑 What is a .pbip File?

A Power BI Desktop Project (`.pbip`) is a **folder-based format** replacing the binary `.pbix` format:
- Git-friendly source control (text-based files, diffable)
- Collaborative editing
- Separation of report and semantic model
- CI/CD pipeline integration

### 📁 .pbip Folder Structure

```
MyReport.pbip                    ← Project entry file (JSON)
MyReport.SemanticModel/          ← Semantic model folder
│   definition.pbism             ← Model entry file
│   definition/
│       model.bim                ← Tabular model JSON (schema, measures, tables)
│       tables/
│           Sales.json
│           Date.json
│   .platform                    ← Fabric platform metadata
MyReport.Report/                 ← Report folder
    definition.pbir              ← Report entry file
    definition/
        report.json              ← Report layout, pages, visuals
        pages/
            Page1.json
    .platform
```

### ⚙️ Creating a .pbip Project

**In Power BI Desktop:**
1. **File → Options → Preview features → Enable "Power BI Project (.pbip) save format"**
2. **File → Save As** → Save as type: **Power BI Project files (*.pbip)**
3. Desktop creates the folder structure automatically

### 🔄 Why .pbip is Better for Git

```
.pbix (binary):  git diff → "Binary files differ" — useless for code review
.pbip (text):    git diff → shows exact DAX measure changes, column additions, etc.
```

**Example git diff for a measure change:**
```diff
--- a/Sales.SemanticModel/definition/tables/Measures.json
+++ b/Sales.SemanticModel/definition/tables/Measures.json
@@ -10,7 +10,7 @@
     {
       "name": "Total Sales",
-      "expression": "SUM(Sales[Amount])",
+      "expression": "CALCULATE(SUM(Sales[Amount]), Sales[Status] = \"Completed\")",
       "formatString": "$#,##0.00"
     }
```

### ⚙️ model.bim File (TMSL JSON)

The `model.bim` file defines the full semantic model schema:

```json
{
  "compatibilityLevel": 1605,
  "model": {
    "name": "Sales Model",
    "tables": [
      {
        "name": "Sales",
        "columns": [
          { "name": "OrderID", "dataType": "int64" },
          { "name": "Amount", "dataType": "decimal" }
        ],
        "measures": [
          {
            "name": "Total Sales",
            "expression": "SUM(Sales[Amount])",
            "formatString": "$#,##0.00"
          }
        ]
      }
    ],
    "relationships": []
  }
}
```

---

## 3. Create and Configure Deployment Pipelines

### 🔑 What are Deployment Pipelines?

Deployment pipelines provide a managed way to promote content through environments:

```
┌─────────────┐    Deploy    ┌─────────────┐    Deploy    ┌─────────────┐
│ Development │ ──────────► │    Test      │ ──────────► │ Production  │
│  Workspace  │              │  Workspace   │              │  Workspace  │
└─────────────┘              └─────────────┘              └─────────────┘
     Dev team                    QA team                  Business users
```

### ⚙️ Creating a Deployment Pipeline

**In Fabric Portal:**
1. Left sidebar → **Deployment pipelines**
2. Click **New pipeline**
3. Name the pipeline (3 stages created by default: Development, Test, Production)
4. Assign a workspace to each stage:
   - Click stage → **Assign a workspace**

> **Note:** Pipelines can be configured with **2–10 stages** (not just 3).

### 📊 Deployment Rules

Deployment rules allow **different configurations per environment**:

```
Use case: Same semantic model, different data sources per environment
Dev:  connects to dev_database.dbo.Sales
Test: connects to test_database.dbo.Sales  
Prod: connects to prod_database.dbo.Sales
```

**Setting deployment rules:**
1. In the pipeline, click the **⚙️ icon** on a stage transition
2. **Deployment rules** → Add rule
3. Choose item → choose property to override:
   - Data source connection string
   - Parameter values
   - Lakehouse/warehouse connection

### 🚀 Deploying Content

**Full deployment:** Select items in stage → click **Deploy** → promotes all selected items

**Selective deployment:** Deploy individual items (useful for hotfixes)

**Backward deployment:** Production → Test → Development (rollback scenario)

### 🔄 Auto-bind Semantic Models and Reports
When a report and its underlying semantic model are deployed together, Fabric automatically **rebinds the report** to the new copy of the semantic model in the target environment.

### ⚙️ Deployment Pipeline Permissions

| Permission | Capability |
|-----------|------------|
| **Pipeline Admin** | Manage pipeline, assign workspaces, change rules |
| **Pipeline Member** | Deploy content, view comparisons |

> ⚠️ **Important:** Users deploying must have **at least Member access** on both source and target workspaces.

---

## 4. Perform Impact Analysis of Downstream Dependencies

### 🔑 What is Impact Analysis?

Impact analysis shows **what will break** if you change a Fabric item — all downstream consumers.

### 🔍 Running Impact Analysis

**In Fabric Portal:**
1. Navigate to the item (e.g., a semantic model)
2. Click **⋯ (More options)** → **View lineage** or **Impact analysis**

**What it shows:**
```
Semantic Model: "Sales Model"
  ↓ Used by 5 reports
  ├── Sales Dashboard (published app)
  ├── Regional Performance Report  
  ├── Monthly KPI Report
  ├── Exec Summary (shared with 50 users)
  └── Ad-hoc Analysis Report
```

### 📊 Lineage View

The **lineage view** provides a visual graph:

```
Fabric Portal → Workspace → Switch view to "Lineage"

Data sources → Dataflows → Lakehouses/Warehouses → Semantic Models → Reports → Dashboards → Apps
```

### 🎯 When to Use Impact Analysis

| Scenario | What to Check |
|----------|--------------|
| Renaming a column | Reports using that column (will break) |
| Changing a data source | All items sourcing from it |
| Deleting a table | Measures/reports referencing it |
| Modifying a dataflow | Downstream semantic models |
| Adding RLS to a model | Reports that will be filtered |

### ⚡ Impact Analysis via REST API

```http
GET https://api.fabric.microsoft.com/v1/workspaces/{workspaceId}/items/{itemId}/getImpactedItems

Response:
{
  "impactedItems": [
    {
      "itemId": "report-guid",
      "itemType": "Report",
      "displayName": "Sales Dashboard",
      "workspaceId": "workspace-guid"
    }
  ]
}
```

---

## 5. Deploy and Manage Semantic Models Using XMLA Endpoint

### 🔑 What is the XMLA Endpoint?

The **XMLA endpoint** is an Analysis Services-compatible endpoint exposing advanced management capabilities:
- Programmatic deployment of semantic models
- Advanced management (partition management, refresh)
- Direct model editing via Tabular Editor or SSMS
- Script-based operations (TMSL)

**XMLA Endpoint URL format:**
```
powerbi://api.powerbi.com/v1.0/myorg/{WorkspaceName}
```

### ⚙️ Enabling XMLA Endpoint

```
Fabric Admin Portal → Capacity settings → [Your capacity]
→ Power BI workloads → XMLA Endpoint → "Read Write"
```

### 🔌 Connecting to XMLA

**SQL Server Management Studio (SSMS):**
1. File → Connect Object Explorer
2. Server type: **Analysis Services**
3. Server name: `powerbi://api.powerbi.com/v1.0/myorg/WorkspaceName`
4. Authentication: Azure Active Directory - MFA

**PowerShell with TMSL:**
```powershell
$connectionString = "Provider=MSOLAP;Data Source=powerbi://api.powerbi.com/v1.0/myorg/MyWorkspace;Integrated Security=ClaimsToken;"

$refreshScript = @"
{
  "refresh": {
    "type": "full",
    "objects": [{"database": "Sales Model"}]
  }
}
"@

Invoke-ASCmd -ConnectionString $connectionString -Query $refreshScript
```

### 🚀 Deploying Semantic Models via XMLA

**Method 1: Deploy from Tabular Editor**
1. Open Tabular Editor → connect to source model
2. **File → Deploy** → Enter XMLA endpoint + workspace
3. Choose: New database OR Overwrite existing (retain partitions option)

**Method 2: TMSL Script via SSMS**
```json
{
  "create": {
    "database": {
      "name": "Sales Model",
      "compatibilityLevel": 1605,
      "model": {
        "tables": [
          {
            "name": "Sales",
            "partitions": [{
              "name": "Sales",
              "source": {
                "type": "m",
                "expression": "let Source = ... in Source"
              }
            }]
          }
        ]
      }
    }
  }
}
```

**Method 3: ALM Toolkit (free tool)**
- Compare two semantic models side-by-side
- Deploy only the differences (schema comparison)
- Ideal for incremental schema deployments

### 📊 Managing Partitions via XMLA

```json
// TMSL: Refresh specific partition only
{
  "refresh": {
    "type": "full",
    "objects": [
      {
        "database": "Sales Model",
        "table": "Sales",
        "partition": "Sales_2024"
      }
    ]
  }
}
```

### 🔒 XMLA Access Modes

| XMLA Mode | Capabilities |
|-----------|------------|
| **Read** (default) | Run DAX queries, read model metadata |
| **Read/Write** | Deploy schemas, create databases, manage partitions, trigger refresh |

> ⚠️ **Exam note:** Read/Write XMLA must be explicitly enabled by the capacity admin.

---

## 6. Create Reusable Assets

### 6a. Power BI Template (.pbit)

A `.pbit` file contains report layout + model schema **without data**.

**Use cases:**
- Standardized report templates for different business units
- Starter templates for common report patterns
- Distribute report structure without data

**Creating a .pbit:**
```
Power BI Desktop → File → Export → Power BI Template (.pbit)
→ Add description → File saved without data
```

**Using a .pbit:**
```
Double-click .pbit → Power BI Desktop opens
→ Prompts for parameters and data source credentials
→ Data loads and report is ready
```

### 6b. Power BI Data Source File (.pbids)

A `.pbids` file stores connection information so users can quickly connect to a data source.

```json
// Example .pbids file (JSON format)
{
  "version": "0.1",
  "connections": [
    {
      "details": {
        "protocol": "azure-sql",
        "address": {
          "server": "myserver.database.windows.net",
          "database": "SalesDB"
        }
      },
      "mode": "DirectQuery"
    }
  ]
}
```

**Common .pbids protocols:**

| Protocol | Source Type |
|---------|------------|
| `azure-sql` | Azure SQL Database |
| `analysis-services` | SSAS / Power BI Premium XMLA |
| `sharepoint` | SharePoint Online |
| `web` | Web URL |
| `odbc` | ODBC connection |

### 6c. Shared Semantic Models

A **shared semantic model** is a central, reusable semantic model that multiple reports connect to.

**Benefits:**
```
Central semantic model → Multiple downstream reports
✅ Single source of truth (one set of measures, one set of relationships)
✅ Governed and endorsed
✅ Version-controlled changes propagate to all reports
✅ Reduced data duplication
```

**Setting up access:**
1. Publish semantic model to a workspace
2. Endorse it (Promoted or Certified)
3. Grant **Build permission** to report creators
4. Report creators connect via: **Power BI Desktop → Get Data → Power BI datasets**

**Connection Types:**

| Connection Type | Description | Local DAX? |
|----------------|-------------|------------|
| **Live Connection** | Report directly queries remote model | ❌ No local measures |
| **Composite Model** | Local + remote semantic model combined | ✅ Add local measures/tables |

```
Composite model setup:
Power BI Desktop → Get Data → Power BI datasets → [Select model]
→ Add DirectQuery/Import local tables in Power Query
→ Build measures on top of the remote model
```

---

## 🏆 ALM Best Practices for Fabric

### Recommended Workflow

```
Developer Branch ──── PR ────► Main Branch
      ↓                              ↓
Dev Workspace                 Production Workspace
(Git: feature/*)             (Git: main)
      ↓                              ↓
Test in Dev              Deploy via Pipeline
      ↓                              ↓
Merge PR                   Promote to Production
```

### Environment Separation Strategy

| Environment | Workspace | Git Branch | Data |
|-------------|-----------|------------|------|
| Development | workspace-dev | feature/* | Dev/sample data |
| Test | workspace-test | release/* | Anonymized prod data |
| Production | workspace-prod | main | Live prod data |

### Source Control Checklist

```
Before merging a PR:
✅ Report tested with expected data
✅ Measures validated in DAX Studio
✅ RLS roles verified (test with "View as role")
✅ Performance tested (Performance Analyzer)
✅ Sensitivity labels applied
✅ Endorsement status set
✅ Impact analysis reviewed
```

---

## ❓ Practice Questions

**Q1:** A developer edited a semantic model locally but realizes the Git version is correct. Which operation reverts local changes?
- A) Commit changes to Git
- B) Update all (pull from Git)
- C) Deploy to next stage
- D) **Undo changes** ✅

---

**Q2:** You want to deploy a semantic model to Production with a different data source connection string than Development. What feature enables this?
- A) Workspace roles
- B) **Deployment rules** ✅
- C) Sensitivity labels
- D) XMLA endpoint

---

**Q3:** An analyst wants to check which reports will be affected before renaming a key column in a shared semantic model. What Fabric feature should they use?
- A) Deployment pipeline
- B) **Impact analysis / lineage view** ✅
- C) Endorsement
- D) Git integration

---

**Q4:** A developer needs to refresh only the last 30 days partition in a semantic model. Which tool/feature supports this?
- A) Deployment pipeline
- B) Git integration
- C) **XMLA endpoint with TMSL** ✅
- D) Sensitivity labels

---

**Q5:** You want to distribute a standard sales report template to regional teams so they can connect it to their own data sources. Which file format should you use?
- A) .pbix (Power BI file)
- B) .pbids (Power BI data source file)
- C) **.pbit (Power BI template file)** ✅
- D) .pbip (Power BI project file)

---

**Q6:** Your organization uses separate Dev, Test, and Prod workspaces. What is the CORRECT order for the analytics development lifecycle?
- A) Production → Test → Development
- B) Development → Production (skipping Test)
- C) **Development → Test → Production (via deployment pipeline)** ✅
- D) Git → XMLA → Deployment pipeline → Git

---

**Q7:** Which file format for Power BI is BEST suited for team collaboration with Git-based version control?
- A) .pbix (binary format)
- B) .pbit (template format)
- C) **.pbip (project folder format)** ✅
- D) .pbids (data source format)

> **Explanation:** `.pbip` is folder-based with text files (JSON, TMSL), making it diffable and merge-friendly in Git. `.pbix` is binary and cannot be meaningfully diff'd.

---

## 📚 Resources & References

| Resource | Link |
|----------|------|
| Git integration overview | [learn.microsoft.com/fabric/cicd/git-integration/intro-to-git-integration](https://learn.microsoft.com/fabric/cicd/git-integration/intro-to-git-integration) |
| Git integration tutorial | [learn.microsoft.com/fabric/cicd/git-integration/git-get-started](https://learn.microsoft.com/fabric/cicd/git-integration/git-get-started) |
| Power BI projects (.pbip) | [learn.microsoft.com/power-bi/developer/projects/projects-overview](https://learn.microsoft.com/power-bi/developer/projects/projects-overview) |
| Deployment pipelines | [learn.microsoft.com/fabric/cicd/deployment-pipelines/intro-to-deployment-pipelines](https://learn.microsoft.com/fabric/cicd/deployment-pipelines/intro-to-deployment-pipelines) |
| Deployment rules | [learn.microsoft.com/fabric/cicd/deployment-pipelines/create-rules](https://learn.microsoft.com/fabric/cicd/deployment-pipelines/create-rules) |
| XMLA endpoint | [learn.microsoft.com/power-bi/enterprise/service-premium-connect-tools](https://learn.microsoft.com/power-bi/enterprise/service-premium-connect-tools) |
| Impact analysis | [learn.microsoft.com/fabric/governance/lineage](https://learn.microsoft.com/fabric/governance/lineage) |
| Shared semantic models | [learn.microsoft.com/power-bi/connect-data/service-datasets-across-workspaces](https://learn.microsoft.com/power-bi/connect-data/service-datasets-across-workspaces) |
| Power BI templates (.pbit) | [learn.microsoft.com/power-bi/create-reports/desktop-templates](https://learn.microsoft.com/power-bi/create-reports/desktop-templates) |
| ALM Toolkit | [alm-toolkit.com](http://alm-toolkit.com/) |
| Tabular Editor | [tabulareditor.com](https://tabulareditor.com/) |

---

> 📌 **Study Tip:** Set up a hands-on lab: connect a Fabric workspace to Azure DevOps, create a `.pbip` file, configure a 3-stage deployment pipeline with a deployment rule, and practice deploying via XMLA endpoint.

---

*Last updated: September 2026 | DP-600 Skills measured as of July 21, 2026*
