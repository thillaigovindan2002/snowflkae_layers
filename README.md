### production question

## Unity Catalog
Centralized governance + metadata layer in Databricks.

Works across multiple workspaces in the same metastore.
## Unity Catalog vs Catalog (Main Difference)
| **Item** | **Meaning** |
| --- | --- |
| **Unity Catalog** | Entire governance system |
| **Catalog** | Logical container inside Unity Catalog |

## Hive Metastore vs Unity Catalog
| **Hive Metastore** | **Unity Catalog** |
| --- | --- |
| Workspace‑level metadata | Account‑level governance |
| Limited security | Fine‑grained security |
| No centralized governance | Centralized governance |
| Limited lineage | Built‑in lineage |
| No row/column security | Row & column‑level security |
| Manual access control | Centralized RBAC |

## Best Practice Architecture
Unity Catalog
   
 finance
 hr
 sales
 marketing
Inside each: catalog → schema → tables

## Hierarchy in Unity Catalog
Inside each workspace: Catalog → Schema → Tables.

It manages:->Access control, Metadata, Data lineage, Auditing, Row/column security, Data discovery, Volumes/files, Time travel governance

across: Databricks workspaces, Azure storage, Lakehouse environments
**overnance Flow with Azure**
Microsoft Entra ID
        ↓
┌─────────────────────────┐
│     Unity Catalog       │
│ Governance Layer        │
└─────────────────────────┘
     ↓       ↓       ↓
 Metadata  Security  Lineage
        ↓
   Azure Databricks
        ↓
ADLS Gen2 / OneLake / Delta Tables
        ↓
Fabric / Power BI / SQL Analytics

Identity layer → Microsoft Entra ID.

Governance layer → Unity Catalog.

Compute layer → Databricks clusters.

Storage layer → ADLS Gen2 / OneLake.

Consumption layer → BI tools like Power BI, Fabric, SQL Analytics

## Partitioning vs Liquid Clustering

Partitioning → Physically splits data into folders.
~~~~~
CREATE TABLE sales (
  order_id INT,
  customer_id INT,
  region STRING,
  amount DOUBLE
)
PARTITIONED BY (customer_id);
~~~~~~~

Liquid Clustering → Internal file organization, adaptive, automatic.

~~~~~
CREATE TABLE sales (
  order_id INT,
  customer_id INT,
  region STRING,
  amount DOUBLE
)
CLUSTER BY (customer_id);
~~~~~~~

Traditional partitioning physically separates data into fixed folders, while Liquid Clustering dynamically organizes Delta Lake files internally for 
adaptive query optimization with lower maintenance overhead.

## Table Types in Unity Catalog

| **Feature** | **Managed** | **External** | **Foreign** |
| --- | --- | --- | --- |
| Data stored in Databricks storage | Yes | No | No |
| Metadata in Unity Catalog | Yes | Yes | Yes |
| Physical data controlled by Databricks | Yes | No | No |
| Live external querying | No | No | Yes |
| DROP deletes data | Yes | No | No |

## Managed Table
Databricks manages metadata + physical files.

No storage path needed.

DROP TABLE deletes both metadata + files.

Undrop available for 7 days
~~~~~~~
CREATE TABLE main.sales.orders (
  order_id BIGINT,
  customer_id BIGINT,
  amount DOUBLE,
  order_date DATE
) USING DELTA;
~~~~~~~~
## External Table
Unity Catalog manages metadata only.

Data remains in ADLS storage.

DROP TABLE deletes metadata, not files.
~~~~~~
CREATE STORAGE CREDENTIAL adls_cred WITH AZURE_MANAGED_IDENTITY;
CREATE EXTERNAL LOCATION raw_sales_loc
URL 'abfss://raw@storageacct.dfs.core.windows.net/sales/'
WITH STORAGE CREDENTIAL adls_cred;

CREATE TABLE main.raw.sales_ext (
  order_id BIGINT,
  customer_id BIGINT,
  amount DOUBLE,
  order_date DATE
) USING DELTA
LOCATION 'abfss://raw@storageacct.dfs.core.windows.net/sales/';
~~~~~~~
## Foreign Table (Federation)
Query external DBs without copying data.

Example: SQL Server federation.
~~~~~~
CREATE CONNECTION sql_conn
TYPE sqlserver
OPTIONS (
  host 'sqlserver.database.windows.net',
  port '1433',
  user 'adminuser',
  password 'password'
);

CREATE FOREIGN CATALOG sql_foreign_catalog
USING CONNECTION sql_conn
OPTIONS (database 'SalesDB');

SELECT * FROM sql_foreign_catalog.dbo.customers;
~~~~~~
## Why UDFs in Security
Purpose → Dynamic masking, Row filtering, Access rules.

### Column‑Level Masking UDF
~~~~~~
CREATE FUNCTION mask_ssn(salary STRING)
RETURN CASE
  WHEN is_account_group_member('admin') THEN salary
  ELSE '****'
END;

ALTER TABLE catalog.schema.table
ALTER COLUMN salary
SET MASK catalog.schema.mask_ssn;
~~~~~~~~
### Row‑Level Masking UDF
~~~~~
CREATE FUNCTION region_filter(region STRING)
RETURN CASE
  WHEN is_account_group_member('india_team') THEN region = 'India'
  WHEN is_account_group_member('us_team') THEN region = 'US'
  ELSE FALSE
END;

ALTER TABLE employees
SET ROW FILTER region_filter ON (region);
~~~~~~~
### Masking UDF example in Unity Catalog
Step 1 — Create Masking UDF
~~~~~
CREATE FUNCTION salary_mask(salary STRING)
RETURN
CASE
    WHEN is_account_group_member('admin')
         THEN salary
    ELSE '****'
END;
~~~~~~
Step 2 — Apply Masking UDF to Table
~~~~~
CREATE TABLE employees_secured
(
    emp_id INT,
    name STRING,
    salary STRING MASK salary_mask
);
~~~~~
## Volumes in Unity Catalog
Definition → Governed file storage locations in Databricks.

Purpose → Store CSV, JSON, PDFs, Images, ML models, unstructured/semi‑structured files without Delta tables.
| **Item** | **Table** | **Volume** |
| --- | --- | --- |
| Structured data | Yes | No |
| SQL querying | Yes | No |
| File storage | Limited | Primary purpose |
| Delta format | Usually yes | Any file type |

### Types of Volumes in Unity Catalog

## Managed Volume:
Databricks manages the storage location and lifecycle of the files.
~~~
CREATE VOLUME sales_catalog.sales_schema.sales_volume;
/Volumes/sales_catalog/sales_schema/sales_volume/
~~~
## External Volume: 
The storage location is managed by the user and points to existing cloud storage.
~~~
CREATE EXTERNAL VOLUME sales_catalog.sales_schema.ext_volume
LOCATION 'abfss://container@storageaccount.dfs.core.windows.net/data/';
~~~
##  How Unity Catalog Works with Azure

Storage Layer :Uses: 👉 Azure Data Lake Storage (ADLS Gen2)

Identity Layer Uses:👉 Microsoft Entra ID

Compute Layer Uses:👉 Databricks clusters

Governance Layer Uses:👉 Unity Catalog

## Core Components of Unity Catalog

~~~
| Metastore          | Central metadata repository |      | Catalog-level | Finance catalog    |
| Catalog            | Top-level container         |      | Schema-level  | HR schema          |
| Schema             | Database/grouping           |      | Table-level   | Employee table     |
| Tables/Views       | Data objects                |      | Row-level     | Only India records |
| Volumes            | File storage governance     |      | Column-level  | Hide salary column |
| External Locations | Secure cloud storage access |      
~~~

## Unity Catalog Features

Unity Catalog provides centralized governance through RBAC, Row-Level Security, Column-Level Security, Auditing, Lineage, Metadata Management, Data Discovery, Data Quality Monitoring, Volumes, and Time Travel, ensuring secure and governed access to data assets across Databricks.

| Feature | Purpose | Example / Use Case |
|----------|---------|-------------------|
| Access Control (RBAC) | Controls who can access what data and resources | GRANT SELECT ON TABLE sales TO analysts; |
| Row-Level Security (RLS) | Restricts data visibility at the row level | Manager sees only records from their region |
| Column-Level Security (CLS) | Restricts access to sensitive columns | Mask or hide Salary column |
| Auditing | Tracks user activities and data access | Who accessed data, queries executed, failed access attempts |
| Lineage | Tracks end-to-end data flow automatically | Raw Table → Transformation → Gold Table → Dashboard |
| Metadata Management | Central repository for metadata | Stores table names, schemas, owners, tags |
| Data Discovery | Enables easy dataset search and discovery | Search "customer" to find tables, views, and owners |
| Data Quality Monitoring | Monitors and validates data quality | Checks nulls, duplicates, invalid values |
| Volumes | Governed storage for non-tabular files | Stores CSVs, PDFs, Images, ML Models |
| Time Travel | Accesses historical versions of Delta tables | SELECT * FROM sales VERSION AS OF 5; |

## 1. Databricks Cluster Types

| Cluster Type | Use Case | Cost | Lifecycle |
|--------------|----------|------|-----------|
| All-Purpose Cluster | Development, Notebooks, Debugging | Higher | Manual / Auto Stop |
| Job Cluster | Production ETL Jobs | Lower | Auto Create & Delete |
| SQL Warehouse | BI Reporting & Dashboards | Optimized | Fully Managed |

## 2. Core Compute Types

| Compute Type | Purpose | Used By | Lifecycle | Best For |
|-------------|----------|----------|------------|-----------|
| All-Purpose Cluster | Interactive Development | Developers, Data Scientists | Manual / Auto Stop | Development |
| Job Cluster | Execute Scheduled Jobs | ETL Pipelines | Auto Create & Delete | Production |
| SQL Warehouse | SQL Analytics | Analysts, BI Users | Auto Managed | Reporting |

## 3. All-Purpose vs Job Cluster vs SQL Warehouse

| Feature | All-Purpose Cluster | Job Cluster | SQL Warehouse |
|----------|---------------------|-------------|---------------|
| Development | ✅ | ❌ | ❌ |
| Production ETL | ❌ | ✅ | ❌ |
| BI / Dashboard | ❌ | ❌ | ✅ |
| Shared by Users | ✅ | ❌ | ✅ |
| Supports Notebooks | ✅ | ❌ | ❌ |
| Auto Creation | ❌ | ✅ | ✅ |
| Auto Deletion | ❌ | ✅ | ✅ |
| Cost Efficiency | Medium | High | High |
| Primary Users | Developers | Pipelines | Analysts |

## 4. Important Cluster Configurations

| Cluster Configuration | Description |
|----------------------|-------------|
| Single Node Cluster | Runs on one machine, suitable for testing and small datasets |
| Multi Node Cluster | Runs on multiple machines for large-scale processing |
| High Concurrency Cluster | Supports multiple users simultaneously |
| Standard Cluster | Default cluster type for general workloads |

## 5. Compute Modes

| Compute Mode | Meaning | Benefits | Limitation |
|--------------|---------|----------|------------|
| Classic | User manages cluster infrastructure | Full control and flexibility | More administration |
| Serverless | Databricks manages infrastructure | Fast startup, no management | Less control |

## 6. Classic vs Serverless

| Feature | Classic | Serverless |
|----------|----------|------------|
| Cluster Management | User Managed | Databricks Managed |
| Startup Time | Slower | Instant |
| Infrastructure Control | High | Low |
| Maintenance | User Responsibility | Databricks Responsibility |
| ETL Workloads | Best Choice | Limited |
| Dashboards | Supported | Best Choice |
| Ad-Hoc Analytics | Supported | Best Choice |
| Custom Libraries | Supported | Limited |

## 7. Cluster Size / Structure

| Type | Meaning | Recommended For |
|--------|----------|----------------|
| Single Node | One Machine | Testing, Learning |
| Multi Node | Multiple Machines | Large Datasets |
| Standard Cluster | Distributed Cluster | General Workloads |

## 8. Workload vs Recommended Compute

| Workload | Recommended Compute |
|-----------|--------------------|
| Development | All-Purpose Cluster |
| Production ETL | Job Cluster |
| Streaming | Lakeflow Pipeline Compute |
| Dashboard Reporting | SQL Warehouse |
| Ad-Hoc SQL Analysis | Serverless SQL Warehouse |
| ML Training | ML Cluster |

## 9. ML Cluster

| Feature | Description |
|----------|------------|
| Purpose | Machine Learning Workloads |
| Pre-installed Libraries | MLflow, Scikit-Learn, TensorFlow, PyTorch |
| Additional Libraries | XGBoost, LightGBM, Hyperopt |
| Advantage | No Manual Installation Required |

## 10. Most Important Interview Mapping

| Use Case | Recommended Setup |
|-----------|------------------|
| Development | All-Purpose Cluster + Classic |
| Production ETL | Job Cluster + Classic |
| BI Reporting | SQL Warehouse + Serverless |
| Ad-Hoc SQL | Serverless SQL Warehouse |
| Testing | Single Node All-Purpose Cluster |
| ML Training | ML Cluster |
| Streaming | Lakeflow Pipeline Compute |

## Interview One-Liner

| Scenario | Answer |
|-----------|--------|
| Development | All-Purpose Cluster 

## Lakeflow Pipeline Compute (formerly Delta Live Tables)
How is it created?  You do not create a cluster manually.

Instead:

Workspace
    ↓
Pipelines
    ↓
Create Pipeline

Pipeline Name : customer_pipeline

Notebook : customer_pipeline.py

Target Catalog : main

Target Schema : bronze
Mode : Triggered / Continuous

Compute : Managed by Databricks

Click Create.

Databricks automatically provisions the compute.
Lakeflow Pipeline Compute (formerly Delta Live Tables) creation

## Databricks Notebook Magic Commands
| Command  | Purpose |
|-----------|----------|
| `%python` | Switch to Python language |
| `%sql` | Switch to SQL language |
| `%scala` | Switch to Scala language |
| `%r` | Switch to R language |
| `%run` | Execute another notebook |
| `%fs` | Perform file system operations |
| `%sh` | Run shell/Linux commands |
| `%md` | Create Markdown documentation |
| `%pip` | Install Python libraries |
| `%time` | Measure execution time |


## %run Example
~~~
# utils_notebook

def add(a, b):
    return a + b

print("Notebook executed")

%run /Shared/utils_notebook

result = add(2, 3)

print(result)
~~~
OUTPUT
Notebook executed
5
**When %run executes**

Runs all code in that notebook
Loads all functions and variables

## import Example

Instead of a notebook, create a Python file.
~~~~
def add(a, b):
    return a + b

import utils

result = utils.add(2, 3)

print(result)
~~~~~
OUTPUT
5
## Simple Explanation

When import executes:

Loads the Python module
Makes functions available
Does not run unnecessary notebook code
Faster and cleaner
## dbutils.notebook.run()

Used to execute another notebook as a separate notebook job.
~~
Child Notebook

Path: /Shared/calculator


# calculator
 
dbutils.notebook.exit("5")

Parent Notebook
Python
result = dbutils.notebook.run(
"/Shared/calculator",
60
)
 
print(result)

Output
Plain Text
5
~~
## what is the difference between %run and import in Databricks?

%run executes the entire target notebook and makes all variables and functions available,
whereas import loads only the required Python module or functions. For production environments, import is preferred because it is faster, cleaner, and provides better code modularity. 
## Secret Scopes and Secrets in Databricks
**What is a Secret Scope?**

A Secret Scope is a secure container used to store sensitive information such as:

Database passwords
API Keys
Access Tokens
Connection Strings
Encryption Keys
SSL/TLS Certificates
~~~
password = dbutils.secrets.get(
    scope="prod-scope",
    key="db-password"
)
~~~
## Architecture View
~~
User Code
    ↓
Secret Scope
    ↓
Secure Storage
    ↓
External Service
~~
## Benefits of Secret Scopes

✅ No hardcoded credentials

✅ Centralized secret management

✅ Role-based access control (ACL)

✅ Secure and auditable

✅ Enterprise-ready

## pes of Secret Scopes
1. Databricks-backed Secret Scope

Secrets are stored inside Databricks.
~~~
Databricks
    ↓
Secret Scope
    ↓
Secrets
~~~
2. Azure Key Vault-backed Secret Scope

Secrets are stored in Azure Key Vault.

~~~
Databricks
↓
Secret Scope
↓
Azure Key Vault
↓
Secrets
~~~
## How can credentials be stored securely in Databricks?
Answer

We create a Secret Scope and store secrets within it.
~~~
password = dbutils.secrets.get(
scope="prod-scope",
key="db-password"
)
~~~
## How to Read Any Secret
~~~
dbutils.secrets.get(
scope="<scope-name>",
key="<key-name>"
)
~~~
~~~
Azure Entra ID
   (Users & Groups)
            ↓
Databricks ACL
    (Secret Scope Access)
            ↓
Managed Identity
     or Service Principal
            ↓
Azure Key Vault RBAC
            ↓
Secrets
            ↓
ADLS / Databases / APIs
~~~

### How Lakeflow Pipeline Compute is Created
You do not create clusters manually.

Instead, Databricks provisions compute automatically when you define a pipeline.

Go to Workspace → Pipelines → Create Pipeline.

Pipeline Name → e.g., customer_pipeline.

Notebook → e.g., customer_pipeline.py.

Target Catalog → e.g., main.

Target Schema → e.g., bronze.

Mode → Triggered (batch) or Continuous (streaming).

Compute → Managed by Databricks.

Click Create → Databricks provisions compute automatically.

### Why This Matters No cluster management  Databricks handles provisioning, scaling, monitoring.

Governance → Integrated with Unity Catalog for lineage & security.

Reliability → Built‑in error handling, retries, monitoring.

Flexibility → Supports both batch (Triggered) and streaming (Continuous).

⚖️ Interview One‑Liner
“Lakeflow Pipeline Compute lets you build ETL pipelines without managing clusters — you define the pipeline (name, notebook, catalog, schema, mode), and Databricks provisions compute automatically, supporting both batch and streaming.”

visual lifecycle diagram (Workspace → Pipeline → Notebook → Managed Compute → Bronze/Silver/Gold tables) so you can memorize the flow faster for interviews?



Auto Loader has built‑in functionality to handle corrupted rows or schema mismatches.

This is managed using the Rescue Data concept.
~~~~
.option("cloudFiles.schemaEvolutionMode","rescue")
~~~~
If not provided, rescue mode is the default.

Auto Loader automatically creates a special column: _rescued_data.

This column stores:

Rows with corrupted data

Rows with schema mismatches (extra columns, unexpected datatypes, etc.)
### Rescue Data Handling
Auto Loader handles corrupted/unmatching schema rows automatically.

Use: .option("cloudFiles.schemaEvolutionMode","rescue").

Creates _rescued_data column → stores corrupted rows.

👉 Interview Line: “Rescue data ensures ingestion continues even if schema mismatches occur, storing bad rows in a separate column.”

⚖️ Architect‑Level Interview Answer
“Databricks Auto Loader supports two file detection modes: directory listing (default, simple but less efficient) and file notification (recommended for production). It offers three trigger types — availableNow, processingTime, and once — for batch and streaming workloads. Schema evolution is managed via schemaLocation, checkpointLocation ensures exactly‑once processing, and rescue data handles corrupted rows gracefully. Together, these features make Auto Loader a powerful tool for incremental and real‑time ingestion.”

### Debugging Code in Databricks
Databricks supports interactive debugging with step in, step over, step out, breakpoints, continue execution.

Debugging Console → lets you inspect variables, evaluate expressions, and control execution flow.

### Difference Between Step In vs Step Out

| **Feature** | **Step Into** (F11) | **Step Over** (F10) | **Step Out** (Shift+F11) |
| --- | --- | --- | --- |
| Goes inside function | ✅ Yes | ❌ No | Already inside |
| Executes current function | Line by line | Entire function | Remaining lines only |
| Returns to caller | After function ends | Immediately after | Yes |

### Genie Code in Databricks
Genie integrates AI into Databricks.

**Two modes**

Chat Mode → conversational assistance.

Agent Mode → autonomous code agent.

**Genie can**

Edit code (new code generation).

Explain code.

Rename variables/functions.

Fix errors.

👉 Interview Line: “Genie brings AI into Databricks with chat and agent modes, helping edit, explain, and fix code directly in notebooks.”


### Lakeflow Connect Overview
Lakeflow Connect is Databricks’ data ingestion service.

It provides managed connectors → no‑code ingestion pipelines.

Supports full load and incremental ingestion automatically.

Designed for metadata‑driven, reusable connections.

### When to Use Lakeflow Connect
For reusable metadata‑driven connections.

When you need a managed way to connect to external sources.

Ideal for enterprise ingestion pipelines where automation and governance are key.

### Lakeflow Spark Declarative Pipelines
Works with Lakeflow Connect.

Declarative → no need to write full read/write code.

Provides ready‑to‑use SQL‑based transformations.

Fully managed → you don’t need to handle infrastructure or orchestration.

### Lakeflow Spark Declarative Pipelines
“Lakeflow Spark Declarative Pipelines let you define ingestion and transformation declaratively, while Databricks manages execution and scaling.”

### Declarative Automation Bundles

“Automation Bundles bring CI/CD discipline into Databricks by packaging and deploying data/AI workflows declaratively.”

### CI/CD Process in Databricks

End‑to‑end CI/CD involves:

Version Control → Git integration.

Build/Test → automated testing of notebooks/pipelines.

Deploy → using Asset Bundles or DevOps pipelines.

Monitor → system tables, dashboards.

Hot Fix → urgent patch applied directly to production, bypassing full release cycle.

### Databricks Architecture — Control Plane
Control Plane → interacts with users and applications.

Manages jobs, notebooks, clusters, security, governance.

Separates from Data Plane (where actual data processing happens).

### Data Quality Monitoring
Built‑in expectations and validation frameworks.

Checks for nulls, duplicates, invalid values.

Example: expect(amount > 0).

Integrated with Unity Catalog for governance.

Databricks Architecture — Control Plane
Control Plane → interacts with users and applications.

Manages jobs, notebooks, clusters, security, governance.

Separates from Data Plane (where actual data processing happens).

👉 Interview Line: “Control Plane = management & orchestration; Data Plane = actual compute and storage.”

### Data Quality Monitoring
Built‑in expectations and validation frameworks.

Checks for nulls, duplicates, invalid values.

Example: expect(amount > 0).

Integrated with Unity Catalog for governance.

### Delta Sharing
Open protocol to share Delta Lake tables securely.

Share with other users, organizations, or platforms.

No data duplication — direct access to live tables.

### Lakehouse Federation
Query external data sources without ingestion.

Access data in place (e.g., SQL DB, cloud storage).

Unified governance via Unity Catalog.

👉 Interview Line: “Lakehouse Federation lets you query external sources directly without moving data into Databricks.”

### System Tables
Prebuilt tables for monitoring and governance.

Store metadata: job runs, billing, cluster usage, query history.

Dashboards can be built on top for observability.

### Genie
AI assistant inside Databricks.

Generates SQL queries automatically.

Works with semantic layer → centralized KPIs, metrics, relationships.

Components: Materialized Views (mview) and Genie Spaces.

👉 Interview Line: “Genie generates SQL queries using a semantic layer of KPIs and curated datasets governed by Unity Catalog.”

### Databricks One
New feature → dedicated UI for business users.

Purpose: simplify data access, reporting, and collaboration.

Bridges gap between technical teams and business stakeholders.

##  Volume vs VACUUM vs ZORDER vs Time Travel vs OPTIMIZE vs UNDROP

## Volume Overview
A Volume is a governed file storage location in Unity Catalog.

Purpose: store unstructured or semi‑structured files like PDFs, CSVs, JSON, Images, ML models.
~~~
CREATE VOLUME main.finance.raw_files;
~~~
Creates a volume inside catalog main, schema finance.
### When to Use Volumes
Unstructured data (PDFs, images, JSON).

File sharing across teams with governance.

ML artifacts (models, training files).

Raw ingestion files before transformation into Delta tables.

### VACUUM Overview
VACUUM physically removes old, unused Delta files from storage.

Delta Lake keeps transaction logs and old versions for Time Travel.

Without VACUUM → storage keeps growing endlessly.

~~~~~
VACUUM sales RETAIN 168 HOURS;
~~~~~

168 HOURS = 7 days retention.

Files older than 7 days are deleted permanently.
### Z‑ORDER Overview
Definition: Z‑ORDER optimizes file organization for faster filtering in Delta Lake.

It improves data skipping and query performance 

~~~
OPTIMIZE sales
ZORDER BY (customer_id);
~~~


Rows with similar customer_id values are stored closer together.

Queries filtering on customer_id will read fewer files.

When to Use Z‑ORDER
✅ Large tables with billions of rows.

✅ Columns frequently used in filters (WHERE, JOIN).

✅ Analytics workloads where query speed matters.


### Time Travel Overview

Time Travel allows you to query older versions of Delta tables.

Delta Lake maintains transaction history and file versions.

This makes it possible to audit, debug, or recover data without restoring backups.


Current Data
~~~~
id   amount
1    100
~~~~
Later Updated
~~~
id   amount
1    500
~~~~
Query Old Version
~~~~~
SELECT * FROM sales VERSION AS OF 1;

SELECT * FROM sales TIMESTAMP AS OF '2025-05-01';
~~~~~~~


### 
When to Use Time Travel
✅ Audit → check historical values.

✅ Rollback → restore table to previous state.

✅ Debugging → compare old vs new data.

✅ Accidental delete recovery → retrieve lost rows.

### OPTIMIZE Overview
Definition: OPTIMIZE compacts many small Delta files into fewer, larger, efficient files.

Problem: Streaming or incremental loads often create lots of small files, which hurt query performance.
~~~
OPTIMIZE sales;
~~~~

When to Use OPTIMIZE
✅ After heavy ingestion (batch or streaming).

✅ Large Delta tables with many small files.

✅ To improve query speed and efficiency.

### UNDROP Overview
Definition: UNDROP recovers accidentally dropped tables or schemas in Databricks.

It works only for managed Delta tables where the underlying files still exist.

~~~~
DROP TABLE sales;

UNDROP TABLE sales;
~~~~
### When to Use?

✅ Accident recovery
✅ Operational mistakes
