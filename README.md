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

**Identity layer → Microsoft Entra ID.

Governance layer → Unity Catalog.

Compute layer → Databricks clusters.

Storage layer → ADLS Gen2 / OneLake.

Consumption layer → BI tools like Power BI, Fabric, SQL Analytics**

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

## Lakeflow Pipeline Compute (formerly Delta Live Tables)
How is it created?  You do not create a cluster manually.

Instead:

Workspace
    ↓
Pipelines
    ↓
Create Pipeline


**Example
Pipeline Name : customer_pipeline
Notebook : customer_pipeline.py
Target Catalog : main
Target Schema : bronze
Mode : Triggered / Continuous
Compute : Managed by Databricks
Click Create.**

Databricks automatically provisions the compute.
Lakeflow Pipeline Compute (formerly Delta Live Tables) creation:

### How Lakeflow Pipeline Compute is Created
You do not create clusters manually.

Instead, Databricks provisions compute automatically when you define a pipeline.

Steps:

Go to Workspace → Pipelines → Create Pipeline.

Provide details:

Pipeline Name → e.g., customer_pipeline.

Notebook → e.g., customer_pipeline.py.

Target Catalog → e.g., main.

Target Schema → e.g., bronze.

Mode → Triggered (batch) or Continuous (streaming).

Compute → Managed by Databricks.

Click Create → Databricks provisions compute automatically.

🔹 Why This Matters
No cluster management → Databricks handles provisioning, scaling, monitoring.

Governance → Integrated with Unity Catalog for lineage & security.

Reliability → Built‑in error handling, retries, monitoring.

Flexibility → Supports both batch (Triggered) and streaming (Continuous).

⚖️ Interview One‑Liner
“Lakeflow Pipeline Compute lets you build ETL pipelines without managing clusters — you define the pipeline (name, notebook, catalog, schema, mode), and Databricks provisions compute automatically, supporting both batch and streaming.”

visual lifecycle diagram (Workspace → Pipeline → Notebook → Managed Compute → Bronze/Silver/Gold tables) so you can memorize the flow faster for interviews?

1. Broadcast Join ->large_df.join(broadcast(small_df), "id") 
2. Repartition Before Join -> 💡 Ensures same join keys go to same partitions. df1.repartition("id").join(df2.repartition("id"), "id")
3.	Filter Before Join 💡 Reduces join data size early.
df1.filter(col("status") == "active").join(df2, "id")

4. Bucketing Optimization -> Reduces shuffle during joins. 
df.write.bucketBy(8, "id").saveAsTable("emp_bucket")
5. Select Required Columns 💡 Avoids unnecessary data movement.
df1.select("id","name").join(df2.select("id","dept"), "id")
6. Handle Skew Join 💡manual and Automatically optimizes skew partitions in AQE. 
spark.conf.set("spark.sql.adaptive.enabled", "true")-> for all
spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")-> skew only
7.	Cache Frequently Used DataFrame 💡 Avoids recomputation in multiple joins. -> df.cache()
8.	Using semi join instead for inner join
9.•	join order lik small table first### explain interview point of view
Alright Thanigai 👌 — let’s break down these Spark Join Optimizations in a crisp interview‑style explanation so you can recall them instantly:

🔹 Join Optimization Techniques
Broadcast Join → large_df.join(broadcast(small_df), "id")  
Pushes small table to all executors → avoids shuffle.  
Interview Line: “Broadcast join is best when one table is small enough to fit in memory — avoids shuffle and speeds up joins.”

Repartition Before Join → df1.repartition("id").join(df2.repartition("id"), "id")  
Ensures same join keys go to same partitions.  
Interview Line: “Repartitioning on join keys reduces skew and ensures parallelism.”

Filter Before Join → df1.filter(col("status")=="active").join(df2,"id")  
Reduces dataset size early.  
Interview Line: “Always filter before join to cut down data movement and shuffle size.”

Bucketing Optimization → df.write.bucketBy(8,"id").saveAsTable("emp_bucket")  
Pre‑shuffles data into buckets → reduces shuffle during joins.  
Interview Line: “Bucketing is useful for repeated joins on the same key — avoids expensive shuffles.”

Select Required Columns → df1.select("id","name").join(df2.select("id","dept"),"id")  
Avoids unnecessary data movement.  
Interview Line: “Project only required columns before join to reduce shuffle size.”

Handle Skew Join (AQE) →

python
spark.conf.set("spark.sql.adaptive.enabled","true")
spark.conf.set("spark.sql.adaptive.skewJoin.enabled","true")
Adaptive Query Execution splits skewed partitions.  
Interview Line: “AQE automatically handles skew joins by splitting large partitions.”

Cache Frequently Used DataFrame → df.cache()  
Avoids recomputation across multiple joins.  
Interview Line: “Cache reused DataFrames to save recomputation cost.”

Semi Join Instead of Inner Join  
Processes only existence check, not full join.  
Interview Line: “Semi joins are lighter when you only need existence checks.”

Join Order (small table first)  
Optimizer prefers small table first for efficiency.  
Interview Line: “Always join smaller tables first to reduce shuffle and memory usage.”

⚖️ Interview One‑Liner Summary
“Spark join optimization is about reducing shuffle, skew, and data movement — use broadcast for small tables, repartition on keys, filter early, bucket for repeated joins, project only needed columns, enable AQE for skew, cache reused DataFrames, and prefer semi joins or small‑table‑first joins.”

👉 Thanigai, do you want me to also prepare a compact cheat‑sheet table (Technique → Code → One‑liner) so you can memorize these faster for interviews?

Query Optimization
•	execution plan
🔥 Most Asked Interview Scenario “How do you optimize slow queries?” 
✅ Answer: “I analyze execution plans to identify shuffles, skew joins, sort merge joins, full scans, and missing predicate pushdown or partition pruning.”
1 ️Broadcast Join : BroadcastHashJoin
2 ️Sort Merge Join : SortMergeJoin,Exchange,Sort
3 ️Predicate Pushdown :  PushedFilters:
4 ️Partition Pruning : PartitionFilters:
5 ️Window Function Plan : Window,Sort,Exchange
Alright Thanigai 👌 — here’s the interview‑style cheat sheet for Query Optimization in Spark using execution plans:

🔹 Most Asked Interview Scenario
Q: “How do you optimize slow queries?”  
A: “I analyze execution plans to identify shuffles, skew joins, sort merge joins, full scans, and missing predicate pushdown or partition pruning.”

🔹 Execution Plan Keywords
Optimization	Execution Plan Keyword	Interview Point
Broadcast Join	BroadcastHashJoin	Small table broadcast → avoids shuffle.
Sort Merge Join	SortMergeJoin, Exchange, Sort	Used for large joins → expensive shuffle.
Predicate Pushdown	PushedFilters:	Filter pushed to source → reduces scan size.
Partition Pruning	PartitionFilters:	Reads only relevant partitions → faster scans.
Window Functions	Window, Sort, Exchange	Expensive → optimize by reducing sort/shuffle.


🔹 Interview One‑Liner Summary
“Query optimization in Spark means reading the execution plan carefully — look for BroadcastHashJoin (good), SortMergeJoin + Exchange (expensive shuffle), PushedFilters (predicate pushdown), PartitionFilters (partition pruning), and Window/Sort/Exchange (heavy operations). Then apply optimizations like broadcast joins, filtering early, bucketing, caching, and AQE.”

👉 Thanigai, do you want me to also prepare a step‑by‑step debugging workflow (df.explain, stats, AQE toggles) so you can demonstrate how you actually debug a slow Spark job in real‑time during interviews?

How do you debug slow Spark jobs?” ✅ Answer: “I analyze Spark explain plans to identify expensive shuffles, skew joins, sort merge joins, missing partition pruning, and inefficient scans.”
Job Running Slow( df.explain(True)) Check: shuffle? sort merge join? partition pruning? broadcast happening? predicate pushdown?

“Why Spark selecting wrong join?” Possible reason: missing statistics Solution ANALYZE TABLE;
	| Command           	| Purpose        |
	| ----------------- 	| -------------- |
	| ANALYZE TABLE     | compute stats |
	| DESCRIBE EXTENDED | view stats     |
| explain(True)     	| optimizer plan |
## interview point of view in above uestion
Alright Thanigai 👌 — here’s how you frame the “How do you debug slow Spark jobs?” question in interviews with a crisp workflow and supporting commands:

🔹 Interview Answer
“I debug slow Spark jobs by analyzing the execution plan (df.explain(True)) to identify expensive shuffles, skew joins, sort merge joins, missing partition pruning, and inefficient scans. If Spark selects the wrong join, I check table statistics and fix it using ANALYZE TABLE.”

🔹 Debugging Workflow
Check Execution Plan → df.explain(True)

Look for:

Shuffle → large data movement.

SortMergeJoin → heavy shuffle + sort.

Partition Pruning → missing? → full scans.

BroadcastHashJoin → happening or not?

Predicate Pushdown → filters applied at source?

Fix Wrong Join Choice

Cause: Missing statistics.

Solution:

sql
ANALYZE TABLE table_name COMPUTE STATISTICS;
DESCRIBE EXTENDED table_name;
With stats, Spark optimizer picks the right join (e.g., BroadcastHashJoin instead of SortMergeJoin).

Adaptive Query Execution (AQE)

Enable AQE for skew handling:

python
spark.conf.set("spark.sql.adaptive.enabled","true")
spark.conf.set("spark.sql.adaptive.skewJoin.enabled","true")
🔹 Interview One‑Liner Summary
“To debug slow Spark jobs, I analyze the execution plan for shuffles, skew joins, sort merge joins, missing pruning, and pushdown. If Spark picks the wrong join, I compute statistics with ANALYZE TABLE. AQE helps handle skew automatically.”

👉 Thanigai, do you want me to also prepare a visual workflow diagram (Explain Plan → Identify Issue → Apply Fix → AQE → Faster Queries) so you can memorize the debugging lifecycle faster for interviews?

?“How do you debug slow Spark jobs?” ✅ Answer: “I analyze Spark explain plans to identify expensive shuffles, skew joins, sort merge joins, missing partition pruning, and inefficient scans.”
Alright Thanigai 👌 — here’s how you frame the “How do you debug slow Spark jobs?” question in interviews with a clear workflow:

🔹 Interview Answer
“I debug slow Spark jobs by analyzing the execution plan (df.explain(True)) to identify expensive shuffles, skew joins, sort merge joins, missing partition pruning, and inefficient scans. If Spark selects the wrong join, I check table statistics and fix it using ANALYZE TABLE.”

🔹 Debugging Workflow
Execution Plan Analysis → df.explain(True)

Look for:

Shuffle → large data movement.

SortMergeJoin → heavy shuffle + sort.

Partition Pruning → missing? → full scans.

BroadcastHashJoin → happening or not?

Predicate Pushdown → filters applied at source?

Statistics Check

Cause: Spark optimizer may pick wrong join strategy if stats are missing.

Solution:

sql
ANALYZE TABLE table_name COMPUTE STATISTICS;
DESCRIBE EXTENDED table_name;
With stats, Spark can choose BroadcastHashJoin instead of SortMergeJoin.

Adaptive Query Execution (AQE)

Enable AQE for skew handling:

python
spark.conf.set("spark.sql.adaptive.enabled","true")
spark.conf.set("spark.sql.adaptive.skewJoin.enabled","true")
🔹 Interview One‑Liner Summary
“To debug slow Spark jobs, I analyze the explain plan for shuffles, skew joins, sort merge joins, missing pruning, and pushdown. If Spark picks the wrong join, I compute statistics with ANALYZE TABLE. AQE helps handle skew automatically.”

👉 Thanigai, would you like me to also prepare a compact cheat‑sheet table (Issue → Execution Plan Keyword → Fix) so you can memorize the debugging lifecycle faster for interviews?

Today
01:21 Unity Catalog
	🔹 What is Unity Catalog? 👉 Unity Catalog is the centralized governance and metadata layer in Databricks. can access multible Workspace/ in same metastore
1. Unity Catalog vs Catalog (Main Difference)
| Item          | Meaning                                |
| ------------- | -------------------------------------- |
| Unity Catalog | Entire governance system               |
| Catalog       | Logical container inside Unity Catalog |

| Hive Metastore            | Unity Catalog               |
| ------------------------- | --------------------------- |
| Workspace-level metadata  | Account-level governance    |
| Limited security          | Fine-grained security       |
| No centralized governance | Centralized governance      |
| Limited lineage           | Built-in data lineage       |
| No row/column security    | Row & column-level security |
| Manual access control     | Centralized RBAC            |


🔥 Best Practice Architecture
Unity Catalog
   ↓
├── finance
├── hr
├── sales
└── marketing
Inside each: catalog → schema → tables

It manages:->Access control, Metadata, Data lineage, Auditing, Row/column security, Data discovery, Volumes/files, Time travel governance

across: Databricks workspaces, Azure storage, Lakehouse environments
---
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
   
   ----
🔥 Interview-Level Final Understanding

👉 Partitioning: data split into small chunks  : Physical folder separation
CREATE TABLE sales
(
 order_id INT,
 customer_id INT,
 region STRING,
 amount DOUBLE
)
partition BY (customer_id);
👉 Liquid Clustering: Arrangement changes automatically based on usage.
Intelligent internal file organization
Databricks:
✅ Reads fewer files
✅ Uses statistics + clustering metadata
CREATE TABLE sales
(
 order_id INT,
 customer_id INT,
 region STRING,
 amount DOUBLE
)
CLUSTER BY (customer_id);
They can work together.
🔥 One-line Interview Answer
👉
Traditional partitioning physically separates data into fixed folders, while Liquid Clustering dynamically organizes Delta Lake files internally for 
adaptive query optimization with lower maintenance overhead.

CREATE TABLE sales
(
    order_id BIGINT,
    customer_id BIGINT,
    year INT,
    region STRING,
    amount DOUBLE
)
USING DELTA
PARTITIONED BY (year) - its folder/year=2025/
CLUSTER BY (customer_id);its Reads only relevant clustered files for:customer_id = 101
---
| Feature                                | Managed | External | Foreign |
| -------------------------------------- | ------- | -------- | ------- |
| Data stored in Databricks storage      | Yes     | No       | No      |
| Metadata in Unity Catalog              | Yes     | Yes      | Yes     |
| Physical data controlled by Databricks | Yes     | No       | No      |
| Live external querying                 | No      | No       | Yes     |
| DROP deletes data                      | Yes     | No       | No      |

🔥 1. Managed Table in Databricks
👉 Databricks manages:Metadata, Physical data files
No storage path needed. *undrop commant availabe only in managedtable for 7 days retrival deleted data
👉 Deletes:
Metadata
Physical files
CREATE TABLE main.sales.orders
(
    order_id BIGINT,
    customer_id BIGINT,
    amount DOUBLE,
    order_date DATE
)
USING DELTA;
🔥 2. External Table
👉 Unity Catalog manages:✅ Metadata
BUT:❌ Data remains in external ADLS storage

🔹 Step 1 — Create Storage Credential
CREATE STORAGE CREDENTIAL adls_cred
WITH AZURE_MANAGED_IDENTITY;
🔹 Step 2 — Create External Location
CREATE EXTERNAL LOCATION raw_sales_loc
URL 'abfss://raw@storageacct.dfs.core.windows.net/sales/'
WITH STORAGE CREDENTIAL adls_cred;
🔹 Step 3 — Create External Table
CREATE TABLE main.raw.sales_ext
(
    order_id BIGINT,
    customer_id BIGINT,
    amount DOUBLE,
    order_date DATE
)
USING DELTA
LOCATION 'abfss://raw@storageacct.dfs.core.windows.net/sales/';
🔹 Drop Table :-> DROP TABLE main.raw.sales_ext;
👉 Deletes ONLY metadata,❌ ADLS files remain

🔥 3. Foreign Table (Federation)
👉 Query external databases WITHOUT copying data
🔹 Step 1 — Create Connection
CREATE CONNECTION sql_conn
TYPE sqlserver
OPTIONS (
  host 'sqlserver.database.windows.net',
  port '1433',
  user 'adminuser',
  password 'password'
);
🔹 Step 2 — Create Foreign Catalog
CREATE FOREIGN CATALOG sql_foreign_catalog
USING CONNECTION sql_conn
OPTIONS (
  database 'SalesDB'
);

SELECT * FROM sql_foreign_catalog.dbo.customers;
🔹 What Happens?
Databricks exposes: sql_foreign_catalog.tables from SQL Server.##in each question in interview point of dont skip single in above content but i need read mode its collabreate so i need clearly
Alright Thanigai 👌 — let’s put all of this into a clear interview‑style “read mode” summary so you can recall every single point without skipping anything.

🔹 Unity Catalog Overview
Unity Catalog → Centralized governance + metadata layer in Databricks.

Works across multiple workspaces in the same metastore.

Manages: Access control, Metadata, Lineage, Auditing, Row/Column security, Data discovery, Volumes/files, Time travel governance.

Integrates with Microsoft Entra ID for identity + RBAC.

🔹 Unity Catalog vs Catalog
Item	Meaning
Unity Catalog	Entire governance system
Catalog	Logical container inside Unity Catalog


🔹 Hive Metastore vs Unity Catalog
Hive Metastore	Unity Catalog
Workspace‑level metadata	Account‑level governance
Limited security	Fine‑grained security
No centralized governance	Centralized governance
Limited lineage	Built‑in lineage
No row/column security	Row & column‑level security
Manual access control	Centralized RBAC


🔹 Best Practice Architecture
Code
Unity Catalog
   ↓
├── finance
├── hr
├── sales
└── marketing
Inside each: catalog → schema → tables
🔹 Partitioning vs Liquid Clustering
Partitioning → Physically splits data into folders.
Example:

sql
CREATE TABLE sales (
  order_id INT,
  customer_id INT,
  region STRING,
  amount DOUBLE
)
PARTITIONED BY (customer_id);
→ Folder structure like /customer_id=101/.

Liquid Clustering → Internal file organization, adaptive, automatic.
Example:

sql
CREATE TABLE sales (
  order_id INT,
  customer_id INT,
  region STRING,
  amount DOUBLE
)
CLUSTER BY (customer_id);
→ Reads fewer files, uses stats + metadata.

Interview One‑liner:  
“Partitioning physically separates data into fixed folders, while Liquid Clustering dynamically organizes Delta Lake files internally for adaptive query optimization with lower maintenance overhead.”

🔹 Table Types in Unity Catalog
Feature	Managed	External	Foreign
Data stored in Databricks storage	Yes	No	No
Metadata in Unity Catalog	Yes	Yes	Yes
Physical data controlled by Databricks	Yes	No	No
Live external querying	No	No	Yes
DROP deletes data	Yes	No	No


1. Managed Table
Databricks manages metadata + physical files.

No storage path needed.

DROP TABLE deletes both metadata + files.

Undrop available for 7 days.

sql
CREATE TABLE main.sales.orders (
  order_id BIGINT,
  customer_id BIGINT,
  amount DOUBLE,
  order_date DATE
) USING DELTA;
2. External Table
Unity Catalog manages metadata only.

Data remains in ADLS storage.

DROP TABLE deletes metadata, not files.

sql
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
3. Foreign Table (Federation)
Query external DBs without copying data.

Example: SQL Server federation.

sql
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
⚖️ Interview One‑Liner Summary
“Unity Catalog is Databricks’ centralized governance layer across workspaces. It supports managed, external, and foreign tables, enforces fine‑grained RBAC, and provides lineage, auditing, and row/column security. Partitioning physically separates data, while Liquid Clustering dynamically organizes files for adaptive optimization.”

👉 Thanigai, do you want me to also prepare a visual layered diagram (Entra ID → Unity Catalog → Databricks → ADLS/Delta → BI tools) so you can memorize the governance flow faster for interviews?

🔥 Why UDF Used in Security?
Used for: Dynamic masking, Row filtering, Access rules
🔥 Real function Usage of UDF in Databricks
| Use Case              | Example             |
| --------------------- | ------------------- |
| Dynamic masking       | SSN masking         |
| Custom business rules | Tax calculation     |
| Data standardization  | Name formatting     |
| Row filtering         | Region-based access |

🔥funtion witn column level masking
CREATE FUNCTION mask_ssn(salary STRING)
RETURN
CASE
  WHEN is_account_group_member('admin')
    THEN salary
  ELSE '****'
END;

*alter table catalag.shema/table 
alter column salary 
set MASK catalog.shema.mask_ssn

🔥🔥funtion witn row level masking

CREATE FUNCTION region_filter(region STRING)
RETURN
CASE
  WHEN is_account_group_member('india_team')
       THEN region = 'India'
  WHEN is_account_group_member('us_team')
       THEN region = 'US'
  ELSE FALSE
END;
*ALTER TABLE employees SET ROW FILTER region_filter ON (region);
--
🔥 Step 1 — Create Masking UDF
CREATE FUNCTION salary_mask(salary STRING)
RETURN
CASE
    WHEN is_account_group_member('admin')
         THEN salary

    ELSE '****'
END;

CREATE TABLE employees_secured
(
    emp_id INT,
    name STRING,
    salary STRING MASK salary_mask
);
---------------------------------------------------------------------------------------
 🔥 What are Volumes in Unity Catalog?
👉 Volumes are governed file storage locations in Databricks.
Used for storing: CSV, JSON, PDFs, Images, ML models, Unstructured/semi-structured files
WITHOUT creating Delta tables.
🔥 Volume vs Table
| Item            | Table       | Volume          |
| --------------- | ----------- | --------------- |
| Structured data | Yes         | No              |
| SQL querying    | Yes         | No              |
| File storage    | Limited     | Primary purpose |
| Delta format    | Usually yes | Any file type   |
🔹 Why Volumes Needed?Before Unity Catalog:
Users directly accessed cloud paths
Hard to govern securely
Example: abfss://raw@storage/...

Problems: No centralized governance, Difficult permissions, Hard auditing for files.
🔥 Volumes Solve This->Unity Catalog provides: ✅ Governance, ✅ Security, ✅ Access control, ✅ Auditing
for files.
🔥 2 Types of Volumes:
1.Managed Volume : Similar to:👉 Managed table
CREATE VOLUME main.finance.raw_files; -> catalog.schema.volume
Users access files via: /Volumes/main/finance/raw_files/
to use: df.write.csv("/Volumes/main/finance/raw_files/customers") or select * from csv.'/Volumes/main/finance/raw_files/customers'
2.External Volume : Similar to:👉 External table
🔹 Example ADLS Path : abfss://raw@storageacct.dfs.core.windows.net/files/
🔹 Step 1 — Create Storage Credential
CREATE STORAGE CREDENTIAL adls_cred
WITH AZURE_MANAGED_IDENTITY;
🔹 Step 2 — Create External Location
CREATE EXTERNAL LOCATION ext_loc
URL 'abfss://raw@storageacct.dfs.core.windows.net/files/'
WITH STORAGE CREDENTIAL adls_cred;
🔹 Step 3 — Create External Volume
CREATE EXTERNAL VOLUME main.finance.external_files
LOCATION 'abfss://raw@storageacct.dfs.core.windows.net/files/';
🔥 Important Interview Trap
👉 “Can we query volume directly using SQL?”
❌ No (not like tables)
Volumes are file storage locations.

🔥 One-line Interview Answer
👉Volumes in Unity Catalog are governed file storage abstractions used for unstructured and semi-structured data, where managed volumes are fully controlled by Databricks and external volumes govern externally stored cloud files through centralized metadata and security policies.
 ---- -------------------------------------
 
 🔹 Core Components of Unity Catalog						🔸 Levels of Security
| Component          | Purpose                     |      | Level         | Example            |
| ------------------ | --------------------------- |      | ------------- | ------------------ |
| Metastore          | Central metadata repository |      | Catalog-level | Finance catalog    |
| Catalog            | Top-level container         |      | Schema-level  | HR schema          |
| Schema             | Database/grouping           |      | Table-level   | Employee table     |
| Tables/Views       | Data objects                |      | Row-level     | Only India records |
| Volumes            | File storage governance     |      | Column-level  | Hide salary column |
| External Locations | Secure cloud storage access |      

🔹 1. Access Control (RoleBasedAC)👉 Controls:Who can access what 777 acces
	2.Uses:👉 Microsoft Entra ID : 🔸 Example: GRANT/revoke SELECT ON TABLE sales TO analysts;
🔹 2. Row-Level Security (RLS)
👉 Restricts rows based on user:-> Example: Manager sees only their region:
region = current_user_region()
🔹 3. Column-Level Security (CLS)
👉 Restricts sensitive columns:-> Example: Hide salary:
MASK salary USING hash_function()
🔹 4. Auditing
👉 Tracks:
Who accessed data
What queries executed
Failed access attempts
Use:
Compliance
Security investigations
🔹 5. Lineage

👉 Tracks data flow automatically

Raw Table
   ↓
Transformation
   ↓
Gold Table
   ↓
Dashboard
Use:
Impact analysis
Debugging
Compliance
🔹 6. Metadata Management
👉 Central metadata repository
Stores: Table names, Schema, Owners, Tags,
Benefit:Easier governance, Central management
🔹 7. Data Discovery :->👉 Search datasets easily
Example:Search: customer
Returns: Tables, Views, Owners
🔹 8. Data Quality Monitoring 👉 Works with:Expectations , Validation frameworks
Checks: Nulls, Duplicates, Invalid values
Example: expect(amount > 0)
🔹 9. Volumes: 👉 Governed file storage in Unity Catalog
Stores: PDFs, Images, CSVs
Types:
| Type            | Meaning                  |
| --------------- | ------------------------ |
| Managed Volume  | Controlled by Databricks |
| External Volume | External cloud storage   |
🔹 10. Time Travel
Using:👉 Delta Lake
Allows:Query old versions
Example: SELECT * FROM sales VERSION AS OF 5
Use: Recovery, Auditing, Debugging
----
🔹 🔥 How Unity Catalog Works with Azure
Storage Layer :Uses: 👉 Azure Data Lake Storage (ADLS Gen2)
Identity Layer Uses:👉 Microsoft Entra ID
Compute Layer Uses:👉 Databricks clusters
Governance Layer Uses:👉 Unity Catalog
Alright Thanigai 👌 — let’s stitch this into a clear interview‑style “read mode” summary so you can confidently explain UDFs in Security + Volumes in Unity Catalog + Core Security Components without skipping a single detail:

🔹 Why UDFs in Security
Purpose → Dynamic masking, Row filtering, Access rules.

Real Usage Examples:

Column‑Level Masking UDF
sql
CREATE FUNCTION mask_ssn(salary STRING)
RETURN CASE
  WHEN is_account_group_member('admin') THEN salary
  ELSE '****'
END;

ALTER TABLE catalog.schema.table
ALTER COLUMN salary
SET MASK catalog.schema.mask_ssn;
Row‑Level Masking UDF
sql
CREATE FUNCTION region_filter(region STRING)
RETURN CASE
  WHEN is_account_group_member('india_team') THEN region = 'India'
  WHEN is_account_group_member('us_team') THEN region = 'US'
  ELSE FALSE
END;

ALTER TABLE employees
SET ROW FILTER region_filter ON (region);
👉 Interview One‑liner:  
“UDFs in Unity Catalog enforce dynamic masking and row filtering, enabling fine‑grained column and row‑level security.”

 ## Volumes in Unity Catalog
Definition → Governed file storage locations in Databricks.

Purpose → Store CSV, JSON, PDFs, Images, ML models, unstructured/semi‑structured files without Delta tables.

Volume vs Table
| **Item** | **Table** | **Volume** |
| --- | --- | --- |
| Structured data | Yes | No |
| SQL querying | Yes | No |
| File storage | Limited | Primary purpose |
| Delta format | Usually yes | Any file type |


Types of Volumes
Managed Volume → Controlled by Databricks.
~~~~
CREATE VOLUME main.finance.raw_files;
Access via /Volumes/main/finance/raw_files/.
~~~~~
External Volume → External cloud storage governed by UC.
~~~~~
CREATE STORAGE CREDENTIAL adls_cred WITH AZURE_MANAGED_IDENTITY;
CREATE EXTERNAL LOCATION ext_loc
URL 'abfss://raw@storageacct.dfs.core.windows.net/files/'
WITH STORAGE CREDENTIAL adls_cred;

CREATE EXTERNAL VOLUME main.finance.external_files
LOCATION 'abfss://raw@storageacct.dfs.core.windows.net/files/';
~~~~~
##  Core Components of Unity Catalog
| **Component** | **Purpose** | **Security Level** | **Example** |
| --- | --- | --- | --- |
| Metastore | Central metadata repository | Catalog‑level | Finance catalog |
| Catalog | Top‑level container | Schema‑level | HR schema |
| Schema | Database/grouping | Table‑level | Employee table |
| Tables/Views | Data objects | Row‑level | Only India records |
| Volumes | File storage governance | Column‑level | Hide salary column |
| External Locations | Secure cloud storage access | — | ADLS path |

**Access Control (RBAC)**
Controls: Who can access what.

Uses: Microsoft Entra ID integration.
~~~~
GRANT SELECT ON TABLE sales TO analysts;
REVOKE SELECT ON TABLE sales FROM analysts;
~~~~
👉 Interview Line: “Unity Catalog enforces role‑based access control via Entra ID groups.”

**Row‑Level Security (RLS)**
Restricts rows based on user identity/region.

Example: Manager sees only their region.
~~~
region = current_user_region()
~~~
👉 Interview Line: “RLS ensures users only see rows relevant to their role or region.”

**Column‑Level Security (CLS)**
Restricts sensitive columns.

Example: Hide salary column.
~~~~
MASK salary USING hash_function()
~~~~~
👉 Interview Line: “CLS masks sensitive columns like salary or SSN.”

**Auditing**
Tracks:

Who accessed data

What queries executed

Failed access attempts

Use: Compliance, investigations.

👉 Interview Line: “Auditing provides full visibility for compliance and security investigations.”

**Lineage**
Tracks data flow automatically:
Raw → Transformation → Gold → Dashboard.

Use: Impact analysis, debugging, compliance.

👉 Interview Line: “Lineage shows how data flows across tables and dashboards.”

**Metadata Management**
Central metadata repository.

Stores: Table names, schemas, owners, tags.

Benefit: Easier governance, central management.

👉 Interview Line: “Metadata management centralizes schema and table information.”

**Data Discovery**
Search datasets easily.

Example: Search customer → returns tables, views, owners.

👉 Interview Line: “Data discovery enables quick search across catalogs and schemas.”

**Data Quality Monitoring**
Works with expectations/validation frameworks.

Checks: Nulls, duplicates, invalid values.
~~~~
EXPECT(amount > 0)
~~~~
👉 Interview Line: “Data quality monitoring enforces rules like non‑negative amounts.”
**Volumes**
Governed file storage in Unity Catalog.

Stores: PDFs, Images, CSVs.

Types:

👉 Interview Line: “Volumes govern unstructured/semi‑structured files with centralized security.”

**Time Travel**
Powered by Delta Lake.

Allows querying old versions.
~~~~~
SELECT * FROM sales VERSION AS OF 5;
~~~~~~
Use: Recovery, auditing, debugging.

👉 Interview Line: “Time Travel lets you query historical versions of data for recovery and compliance.”
## Databricks Cluster Types
| **Cluster Type** | **Use Case** | **Cost** | **Lifecycle** |
| --- | --- | --- | --- |
| **All‑Purpose** | Dev / notebooks | Higher | Manual / auto‑stop |
| **Job Cluster** | Production jobs | Lower | Auto create & delete |
| **SQL Warehouse** | BI / dashboards | Optimized | Managed (auto‑handled) |

## Core Compute Types
| **Type** | **Purpose** | **Used By** | **Lifecycle** | **Best For** |
| --- | --- | --- | --- | --- |
| **All‑Purpose Cluster** | Interactive work | Devs, Data Scientists | Manual / auto‑stop | Development, debugging |
| **Job Cluster** | Runs scheduled jobs | Pipelines | Auto create & delete | Production ETL |
| **SQL Warehouse** | SQL queries only | BI tools, Analysts | Auto‑managed | BI / Reporting |

## Cluster Types & Configurations
**All‑Purpose Cluster (Interactive Cluster)**
Used for: Development, ad‑hoc analysis, notebooks, data exploration.

Features:

Shared by multiple users.

Always running (until auto‑terminated).

Supports notebooks, SQL, Python.

Benefits: Flexible, supports experimentation.

Example Use Case: Writing PySpark code, testing transformations, debugging pipelines.

**Job Cluster (Ephemeral Cluster)**
Used for: Scheduled jobs, production pipelines, batch processing.

Features:

Created automatically when job starts.

Terminates after job completes.

Not shared between users.

Benefits: Cost‑efficient 💰, clean environment every run.

Example Use Case: Daily ETL pipeline, batch processing.

**SQL Warehouse (formerly SQL Endpoint)**
Used for: BI tools, dashboards, SQL analytics.

Features:

Optimized for SQL queries.

Supports tools like Power BI / Tableau.

Serverless options available.

Benefits: Fast, auto‑managed, ideal for analysts.

Example Use Case: BI dashboards, ad‑hoc SQL queries.
## Important Cluster Configurations
| **Type** | **Meaning** | **When to Use** |
| --- | --- | --- |
| **Single Node Cluster** | Runs on one machine | Testing, small data |
| **High Concurrency Cluster** | Supports multiple users simultaneously | Shared environments, BI workloads |
| **Standard Cluster** | Default distributed cluster | General workloads |

## Compute Modes
| **Mode** | **Meaning** | **Benefit** | **Limitation** |
| --- | --- | --- | --- |
| **Classic** | You manage the cluster (VMs, configs, autoscaling) | Full control, flexibility | More setup, slower startup |
| **Serverless** | Databricks manages everything | No setup, instant start ⚡ | Less control, limited customization |
## Cluster Size / Structure
| **Type** | **Meaning** | **When to Use** |
| --- | --- | --- |
| **Single Node** | Runs on one machine | Small data, testing, quick experiments |
| **Multi Node** | Runs on multiple machines | Big data, production workloads |
| **Standard Cluster** | Default distributed cluster | General workloads, balanced choice |
| **High Concurrency** | Supports multiple users simultaneously | Shared environments, notebooks, BI tools |

## Classic vs Serverless Comparison
| **Aspect** | **Classic (Customer‑managed)** | **Serverless (Databricks‑managed)** |
| --- | --- | --- |
| Control | Full control (VMs, configs, libraries) | No control, Databricks manages |
| Startup | Slow (1–5 mins) | Instant ⚡ |
| Best For | ETL pipelines, ML, custom workloads | BI dashboards, SQL queries |
| Cost Model | Pay for uptime (even idle) | Pay per query / usage |
| Flexibility | High | Limited |
| Users | Data Engineers, Platform teams | Analysts, BI users |

## Extended Workload Mapping
| **Workload** | **Recommended Compute** |
| --- | --- |
| Development | All‑Purpose Cluster (Auto Termination 15–30 min) |
| Production ETL | Job Cluster |
| Streaming | Lakeflow Pipeline Compute |
| Dashboard | SQL Warehouse |
| ML Training | ML Cluster (pre‑installed MLflow, Scikit‑learn, TensorFlow, PyTorch, XGBoost, LightGBM, Hyperopt) |
| Ad‑hoc SQL | Serverless SQL Warehouse |

## Benefits Summary
Classic Benefits:

Low cost if optimized.

Full control over cluster size, libraries, configs.

Supports all workloads (ETL, ML, streaming).

Better for production pipelines → privacy, visibility, control.

Serverless Benefits:

No cluster setup → zero DevOps.

Instant query execution ⚡.

Auto‑scaling handled internally.

Pay only for what you use 💰.

Best for BI tools (Power BI, Tableau).


## Classic vs Serverless Compute         
| Feature               | **Classic (Customer-managed)**             | **Serverless (Databricks-managed)**              |
| --------------------- | ------------------------------------------ | ------------------------------------------------ |
| **Programming**       | Spark, SQL, Python, Scala, R               | Mostly SQL + limited Python (depends on feature) |
| **Primary Purpose**   | Full control workloads, custom pipelines   | Fast analytics, BI, simple workloads             |
| **Typical Users**     | Data Engineers, Platform teams             | Analysts, BI users, Data Scientists              |
| **Cluster Setup**     | You configure cluster (nodes, autoscaling) | No cluster setup needed                          |
| **Cluster Lifecycle** | Manual / auto-termination                  | Fully managed (auto start/stop)                  |
| **Startup Time**      | Slow (1–5 mins)                            | Instant ⚡                                        |
| **Performance**       | Depends on config                          | Optimized by Databricks                          |
| **Cost Model**        | Pay for cluster uptime (even idle)         | Pay per query / usage                            |
| **Cost Efficiency**   | Lower if optimized well                    | High for intermittent workloads                  |
| **Maintenance**       | You manage configs, libraries              | No maintenance                                   |
| **Flexibility**       | High (custom libraries, configs)           | Limited customization                            |
| **Security/Control**  | Full control (VPC, networking)             | Managed by Databricks                            |
____________________________________________________________________________________________________________________________________________________________

## Notebook Magic Commands

| **Magic Command** | **Purpose** |
| --- | --- |
| **%python, %sql, %scala, %r** | Switch language inside a cell |
| **%run** | Run another notebook (reuse code) |
| **%fs** | File system operations (list, copy, move) |
| **%sh** | Run shell commands |
| **%md** | Add documentation/markdown |
| **%pip** | Install Python libraries |
| **%time** | Measure execution time |

## %run vs import

| **%run (Notebook‑based)** | **import (Python module‑based)** | **Why **``import``** is better** |
| --- | --- | --- |
| Runs entire notebook | Imports specific functions/classes | Loads only functions you need |
| Notebook‑based | Python module‑based | Doesn’t execute unnecessary code |
| Re‑executes every time | Loaded once | Faster ⚡ |
| Less control | More control | Cleaner architecture |
| Not ideal for production | Best practice | Recommended for modular pipelines |

## 1. %run — How it Works

Notebook: /Shared/utils_notebook
~~~~~
def add(a, b):
    return a + b
print("Notebook executed")
~~~~~
 Main Notebook
~~~~~
%run /Shared/utils_notebook
add(2, 3)   # Output: 5
~~~~~

Behavior:

%run executes the entire notebook.

It re‑runs all code (including print, heavy logic, file reads).

Every call reloads everything → slow and inefficient.

 Bad Example:

~~~~~~
df = spark.read.csv("big_file.csv")   # heavy operation ❌
def clean(df):
    return df.dropDuplicates()
~~~~~~
Using %run here → reloads the big file every time → slows down notebook execution.

2. import — Best Practice
 File: utils.py

~~~~
def add(a, b):
    return a + b
~~~~
📘 Main Notebook

~~~~
import sys
sys.path.append("/Workspace/Repos/your_repo/")
import utils
utils.add(2, 3)
~~~~~
👉 Behavior:

Loads only the functions/classes you need.

Doesn’t execute unnecessary code.

Faster ⚡ because it’s loaded once.

Cleaner architecture → modular, reusable, production‑ready.

## run vs dbutils.notebook.run vs import

| **Criteria** | **%run** (Notebook‑based) | **dbutils.notebook.run()** | **import (Python module)** |
| --- | --- | --- | --- |
| **Main Use Case** | Reuse variables & functions | Run notebook as a job | Modular reusable code |
| **Execution Type** | Inline (same notebook) | Separate job execution | Load functions only |
| **Thread/Process** | Same thread | New job / separate context | Same process |
| **Performance** | Medium (re‑runs full notebook) | Slower (job overhead) | Fast ⚡ |
| **Input Parameters** | ❌ Not supported | ✅ Supported | ✅ Supported |
| **Output Return** | ❌ No | ✅ Yes (string output) | ✅ Yes |
| **Exception Handling** | ❌ Limited | ✅ Supported | ✅ Supported |
| **Reusability** | Low | Medium | High ✅ |
| **Best for Production** | ❌ No | ⚠️ Limited | ✅ Yes |
| **Magic Command** | ✅ Yes | ❌ No | ❌ No |

## Scenario Mapping

| **Scenario** | **Best Option** |
| --- | --- |
| Quick dev reuse | ``%run`` |
| Pipeline orchestration | ``dbutils.notebook.run()`` |
| Production ETL / ML | ``import`` ✅ |

## Secret Scopes Interview Questions

### What are Secret Scopes in Databricks? 

 A secure container to store sensitive data like DB passwords, API keys, tokens, and certificates — instead of hardcoding them in notebooks.
### What are the types of Secret Scopes?

Databricks‑backed (stored inside Databricks, encrypted at rest).  
Azure Key Vault‑backed (integrated with Key Vault, best for enterprise security).
### Why do we use Secret Scopes? 

To avoid storing credentials in code. They provide centralized management, RBAC, and audit‑friendly governance.  
👉 Architecture Flow: User Code → Secret Scope → Secure Storage → External Service.

### How do you access secrets programmatically in Databricks?  
👉 Using dbutils.secrets.get(scope="prod-scope", key="db-password").

### What is the difference between Databricks‑backed and Key Vault‑backed scopes?  
👉 Databricks‑backed → secrets stored inside Databricks.  
👉 Key Vault‑backed → secrets stored in Azure Key Vault, Databricks scope maps to Key Vault.

### Reading Secrets from Secret Scope

~~~~
dbutils.secrets.get(scope="prod-scope", key="db-password")
~~~~

### Enterprise Security Flow

~~~
Azure Entra ID (Users & Groups)
        ↓
Databricks (ACL on Secret Scope)
        ↓
Managed Identity / Service Principal
        ↓
Azure Key Vault (RBAC)
        ↓
Secrets
        ↓
Data Systems (ADLS / DB / APIs)
~~~~

### Architecture level
“In Databricks, secure credential management is implemented using Key Vault-backed secret scopes. Access is controlled at two levels: Databricks ACLs define who can read the secret, while Azure Key Vault RBAC controls whether Databricks can retrieve the secret. Authentication is handled via Managed Identity or Service Principal, ensuring no secrets are hardcoded. This layered security model provides centralized, auditable, and scalable enterprise-grade access control.”

## Real ADF + Databricks + Key Vault architecture

### Identity Layer
Azure Entra ID → Centralized user & group management.

Provides RBAC (role‑based access control) and SSO authentication.
👉 Ensures only authorized users/services can request secrets.

### ADF Orchestration Layer
ADF uses Managed Identity → passwordless authentication.

Fetches secrets directly from Azure Key Vault.

Triggers Databricks notebooks via Linked Service.

👉 Interview Line: “ADF orchestrates pipelines and securely fetches secrets from Key Vault using Managed Identity.”

### Azure Key Vault Security Layer
Stores DB passwords, API keys, storage credentials.

No secrets stored in ADF or Databricks.

Access governed by RBAC policies.

👉 Interview Gold Statement: “Key Vault is the single source of truth for secrets — centralized, encrypted, and auditable.”

### Databricks Processing Layer
Databricks integrates with Key Vault via Secret Scopes.

Secrets accessed programmatically:

python
dbutils.secrets.get(scope="prod-scope", key="db-password")
Processing done with Spark → results stored in Delta tables.

👉 Interview Line: “Databricks notebooks fetch secrets at runtime via Key Vault‑backed scopes, ensuring no hardcoded credentials.”

🔹 Unity Catalog Governance Layer
Provides centralized data access control.

Table‑level permissions, lineage, and auditing.
👉 Ensures compliance and governance across ADLS and external systems.

### End‑to‑End Flow
ADF Authentication → Managed Identity → Key Vault.

Fetch Secrets → Key Vault returns DB password.

Trigger Databricks → Notebook execution.

Databricks Access Secrets → Secret Scope + dbutils.secrets.get.

Data Processing → Read from ADLS, transform with PySpark, write to Delta.

👉 Real Example: SQL DB → ADF pipeline → Key Vault secrets → Databricks processing → ADLS storage → Unity Catalog governance.


### Databricks Volume & ADLS Connection
Purpose: Connect Databricks with external storage (Azure ADLS, AWS S3, GCP GCS).

Process Flow:

Create Storage Credential in Unity Catalog → defines how Databricks authenticates to storage.

Grant Permission to ADLS Account → using RBAC/ACLs.

Create External Location → maps Databricks to ADLS/S3/GCS path.

Create Volume → logical mount inside Databricks that points to external location.

Databricks provides dbutils.fs utilities for file system operations:

| **Command** | **Purpose** |
| --- | --- |
| ``dbutils.fs.ls(path)`` | List files/folders |
| ``dbutils.fs.cp(src, ``dest)`` | Copy files |
| ``dbutils.fs.mv(src, ``dest)`` | Move files |
| ``dbutils.fs.rm(path, ``recurse=True)`` | Remove files/folders |
| ``dbutils.fs.mkdirs(path)`` | Create directory |
| ``dbutils.fs.head(path)`` | Preview first lines of a file |
| ``dbutils.fs.put(path, ``contents, ``overwrite=True)`` | Write file contents |


##  Databricks Job related questions
###  how can orchestrates  different notebook in databricks
1. Orchestrating Different Notebooks
Databricks Jobs can orchestrate multiple notebooks.

Supports parallel execution, conditional DAG flows, and error handling.
| **Feature** | **Airflow** | **ADF** | **Databricks Jobs** |
| --- | --- | --- | --- |
| Main Purpose | Workflow orchestration | Azure ETL orchestration | Native Databricks orchestration |
| Best For | Complex DAG workflows | Azure integrations | Spark/Notebook workflows |
| Language | Python DAG | UI based | UI + Notebook |
| Parallel execution | Strong | Yes | Yes |
| Conditional flow | Strong | Yes | Yes |
| Retry handling | Excellent | Good | Good |
| Multi‑cloud | Yes | Mostly Azure | Mostly Databricks |
| Scheduling | CRON | Trigger based | CRON |
| Notebook orchestration | Yes | Yes | Best |

## 2. Scheduling Entities
Jobs can schedule:

Notebook runs

Dashboard refreshes

Delta Live Tables pipelines

SQL queries

Scheduling via CRON expressions or time‑based triggers.


### Passing Parameters to Notebooks
Use Job Parameters / Base Parameters.
~~~~~
dbutils.widgets.text("table_name","")
dbutils.widgets.text("load_type","")
table_name = dbutils.widgets.get("table_name")
load_type = dbutils.widgets.get("load_type")
~~~~~
### Returning Values Between Tasks
Use dbutils.jobs.taskValues.set/get.
~~~~
# Notebook A
dbutils.jobs.taskValues.set(key="count", value=5000)

# Notebook B
count = dbutils.jobs.taskValues.get(taskKey="task_A", key="count", default=0)
print(count)
~~~~~
### Parameters in For‑Each Loop
Jobs support for‑each loops.

Parameters can be passed dynamically to each iteration (e.g., different table names).
👉 Useful for batch processing multiple datasets with one notebook.

### Failure Handling & Retries
Retry Policy: Configure retries per task.

Repair Run: Rerun only failed tasks without restarting the entire job.
👉 Ensures efficient recovery from errors.
### Automated Notifications
Jobs can send email notifications on success/failure.

Configurable in Job UI → Notifications tab.
👉 Ensures stakeholders are alerted immediately on job failures.
## Auto Loader
Auto Loader is a ready‑to‑use library in Databricks for incremental ingestion.

No complex setup required.

Supports multiple formats: CSV, JSON, Parquet, Avro, ORC, etc.
👉 Interview Line: “Auto Loader automatically detects and ingests new files from cloud storage with schema evolution and checkpointing.”

## File Detection Modes

### Directory Listing Mode (Default, older)
Auto Loader identifies new files by listing the input directory.

No additional permissions required.

Simple to configure but less efficient (repeated scans).
👉 Best for POCs or small projects.

### File Notification Mode (Recommended, newer)
Uses cloud file notifications + queue services.

Requires infra setup (Event Grid, Queue, etc.).

Highly efficient → lower latency, lower cost.
👉 Best for production workloads.

Flow:  
File → Cloud Storage → Event Broker → Queue Service → Auto Loader consumes new file list.

## Trigger Types
### availableNow

Processes all available files once, then stops.

Perfect for incremental batch workloads.
~~~
df = (
  spark.readStream
       .format("cloudFiles")
       .option("cloudFiles.format", "csv")
       .load("/mnt/raw/")
)

(df.writeStream
   .trigger(availableNow=True)
   .option("checkpointLocation", "/chk/")
   .start("/bronze/"))

~~~~~~
**🔥 What Happens?
✅ Processes all available files like batch
✅ Stops automatically after completion
✅ Still keeps Auto Loader benefits: schema evolution, incremental tracking, no duplicates**

### processingTime

Runs continuously at defined intervals.

Best for real‑time ingestion.

~~~~~
df = (
  spark.readStream
       .format("cloudFiles")
       .option("cloudFiles.format", "csv")
       .load("/mnt/raw/")
)

(df.writeStream
   .trigger(processingTime="1 minute")
   .option("checkpointLocation", "/chk/")
   .start("/bronze/"))

~~~~~~

### 
once

Processes all data once, then exits.

Good for one‑time loads.
~~~~
df = (
  spark.readStream
       .format("cloudFiles")
       .option("cloudFiles.format", "csv")
       .load("/mnt/raw/")
)

(df.writeStream
   .trigger(once=True)
   .option("checkpointLocation", "/chk/")
   .start("/bronze/"))
~~~~

## Schema Handling
Schema Location: .option("cloudFiles.schemaLocation","/schema/") → tracks schema changes.

Options:

Provide schema manually (StructType/StructField) → preferred for production.

No schema provided → defaults all columns to string.

Use .option("cloudFiles.inferColumnTypes","true") to infer datatypes automatically.

👉 Interview Line: “Schema location ensures Auto Loader can evolve schema safely across new files.”

### Checkpoint Location
.option("checkpointLocation","/chk/") → stores metadata of processed files.

Ensures exactly‑once processing and avoids duplicates.
### Corrupted Data Handling in Auto Loader
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

### Volume
Definition: Governed file storage location in Unity Catalog.

Use Case: Store unstructured data (PDFs, CSV, Images, JSON, ML models).
~~~~
CREATE VOLUME main.finance.raw_files;
~~~~~
Access Path: /Volumes/main/finance/raw_files/  
👉 Best for raw ingestion files, ML artifacts, file sharing.

### VACUUM
Definition: Physically removes old, unused Delta files from storage.
~~~~~~
VACUUM sales RETAIN 168 HOURS;
~~~~~
Why: Delta keeps old versions & logs → storage grows.
👉 Best for reducing storage cost, cleaning up old files.
⚠️ After VACUUM → old versions may not be recoverable.

### ZORDER
Definition: Optimizes file organization for faster filtering.
~~~
OPTIMIZE sales ZORDER BY (customer_id);
~~~~
Benefit: Rows with similar customer_id stored together → fewer files scanned.
👉 Best for large tables, frequent filtering columns, analytics workloads.

### Time Travel
Definition: Read old versions of Delta tables using transaction history.
~~~~
SELECT * FROM sales VERSION AS OF 1;
SELECT * FROM sales TIMESTAMP AS OF '2025-05-01';
~~~~
👉 Best for audit, rollback, debugging, accidental delete recovery.
### OPTIMIZE Command
Definition: Compacts small files into larger efficient ones.

Problem: Streaming/incremental loads → many small files → bad performance.
~~~~~~
OPTIMIZE sales;
~~~~~~~
👉 Best for after heavy ingestion, large Delta tables, improving query speed.

### UNDROP
Definition: Recover accidentally dropped table/schema.
~~~~~
DROP TABLE sales;
UNDROP TABLE sales;
~~~~~~
Requirement: Files must still exist, not deleted by VACUUM.
👉 Best for accident recovery, operational mistakes.
### Relationship Between Them

| **Feature** | **Purpose** | **When to Use** |
| --- | --- | --- |
| **[Volume](ca://s?q=Databricks_Volume)** | File storage (raw/unstructured) | ML models, ingestion files |
| **[OPTIMIZE](ca://s?q=Databricks_OPTIMIZE)** | File compaction | After ingestion, improve query speed |
| **[ZORDER](ca://s?q=Databricks_ZORDER)** | Faster filtering | Large tables, analytics |
| **[Time Travel](ca://s?q=Databricks_Time_Travel)** | Read old versions | Audit, rollback, debugging |
| **[VACUUM](ca://s?q=Databricks_VACUUM)** | Delete old unused files (hard delete) | Reduce storage cost |
| **[UNDROP](ca://s?q=Databricks_UNDROP)** | Recover dropped objects | Accident recovery |

### Daily ETL → Small Files

~~~
INSERT INTO sales ...
~~~~
### Optimize Performance
~~~~
OPTIMIZE sales
ZORDER BY (customer_id);
~~~~~~
OPTIMIZE: Compacts small files into larger ones.

ZORDER: Organizes data for faster filtering.

### Querying with ZORDER
~~~~
SELECT * FROM sales WHERE customer_id = 101;
~~~~~
Reads fewer files because rows with similar customer_id are stored together.

### Need Historical Data?
~~~
SELECT * FROM sales VERSION AS OF 5;
~~~
### Cleanup Old Files
~~~~~
VACUUM sales RETAIN 168 HOURS;
~~~~~~
Removes old, unused Delta files physically.
👉 Best for reducing storage cost.
⚠️ After VACUUM → old versions may not be recoverable.

### Accidentally Dropped?
~~~~~
DROP TABLE sales;
UNDROP TABLE sales;
UNDROP: Recovers dropped tables/schemas.
~~~~
👉 Works only if files still exist and VACUUM has not deleted them.

🔥 Interview Trap:  
👉 “Can UNDROP work after VACUUM?”  
❌ Usually NO — because physical files may already be deleted.

### OPTIMIZE vs ZORDER

| **Command** | **Purpose** |
| --- | --- |
| OPTIMIZE | Merge small files |
| ZORDER | Organize data for filtering |


### Time Travel vs UNDROP
| **Feature** | **Purpose** |
| --- | --- |
| Time Travel | Read old table versions |
| UNDROP | Recover deleted table object |


### Easy Analogy

| **Feature** | **Analogy** |
| --- | --- |
| Volume | Storage room |
| OPTIMIZE | Combine many tiny boxes |
| ZORDER | Arrange files smartly |
| Time Travel | View old snapshots |
| VACUUM | Throw away old garbage |
| UNDROP | Restore deleted folder |





