# Databrick
### Delta Lake in Databricks
Definition: Delta Lake is a storage layer built on top of Parquet with ACID transactions.

Key Interview Point: “Delta Lake adds reliability to data lakes by enabling ACID, schema enforcement, and time travel.”

### OPTIMIZE vs ZORDER
OPTIMIZE → Compacts small files into larger ones → faster queries.

ZORDER → Clusters data by column values during OPTIMIZE.

Dependency: ZORDER requires OPTIMIZE; without OPTIMIZE, ZORDER does not work.

Disadvantages: Expensive, not incremental, skewed data reduces effectiveness.

Interview One‑liner: “OPTIMIZE reduces small files; ZORDER improves multi‑column filtering but is costly.”

###Liquid Clustering
Definition: Dynamic clustering that adapts automatically as data evolves.

Why required: Handles skew better, reduces manual maintenance, replaces ZORDER.

Interview One‑liner: “Liquid clustering is self‑adjusting and modern, unlike static ZORDER.”

### Versioning & Recovery
Transaction Log → Every commit creates a new version.

**Time Travel:**
~~~~~
SELECT * FROM sales VERSION AS OF 25;
~~~~~
**Restore:**
~~~~~
RESTORE TABLE sales TO VERSION AS OF 25;
~~~~~

Default retention: 30 days (configurable).

Interview Point: “Delta Lake supports time travel and restore via its transaction log.”

### View vs MView vs Volume vs Foreign Table

### View
Definition: Logical layer, no storage, always queries base tables.

~~~~~
CREATE OR REPLACE VIEW sales_view AS
SELECT customer_id, SUM(amount) AS total_sales
FROM sales
GROUP BY customer_id;
~~~~~
Interview Point: “Views are logical, lightweight, and always query the base table.”

### Materialized View (MView)
Definition: Stores precomputed results for faster queries.

~~~~~~
CREATE MATERIALIZED VIEW sales_mview
AS SELECT customer_id, SUM(amount) AS total_sales
FROM sales
GROUP BY customer_id;
~~~~~~
Refresh:

~~~~~~
REFRESH MATERIALIZED VIEW sales_mview;
~~~~~~
Interview Point: “MViews trade storage for speed; they need refresh.”

### Volume
Definition: Managed storage in Unity Catalog for raw files (CSV, Parquet, JSON).

~~~~~
CREATE VOLUME my_volume;
~~~~~
Files can be uploaded into this volume and accessed via SQL or notebooks.

Interview Point: “Volumes are Databricks‑managed storage for raw files.”

### Foreign Table
Definition: References external data sources (S3, ADLS, etc.) without copying data.

~~~~~
CREATE FOREIGN TABLE ext_sales
USING CSV OPTIONS ('path'='/mnt/s3/sales.csv');
~~~~~
Interview Point: “Foreign Tables connect external data lakes directly.”

**Quick Interview Recall**

View → Logical, no storage.

MView → Precomputed, faster, needs refresh.

Volume → Managed storage for raw files.

Foreign Table → External data reference, no copy.
### Internal Table
Definition: A table fully managed by Databricks (Unity Catalog).

Storage: Data + metadata stored inside Databricks‑managed location.

Lifecycle: Dropping the table also deletes the underlying data.

Use Case: Best for new projects where Databricks should control governance, security, and lifecycle.

Interview One‑liner: “Internal tables are Databricks‑managed — simple governance, but less flexible.”

### External Table
Definition: A table that references data stored outside Databricks (e.g., S3, ADLS, GCS).

Storage: Metadata in Databricks, but data files remain in external storage.

Lifecycle: Dropping the table does not delete the underlying data.

Use Case: Best when integrating with existing data lakes or sharing data across multiple platforms.

Interview One‑liner: “External tables give flexibility — you manage the data lifecycle outside Databricks.”
| Feature | **Internal Table** | **External Table** |
| --- | --- | --- |
| Storage | Databricks‑managed | External (S3, ADLS, GCS) |
| Lifecycle | Drop deletes data | Drop keeps data |
| Governance | Simplified, Databricks controls | Flexible, user controls |
| Best Use | New projects, full governance | Existing lakes, cross‑platform sharing |




