# DP-600 Study Notes: In-Depth Exam Cheat Sheet
This guide condenses the absolute high-yield topics that appear most frequently on the **DP-600: Implementing Analytics Solutions Using Microsoft Fabric** exam. Use this for quick revision and exam-day prep.

---

## ⚡ Direct Lake Mode & Fallback Traps
Direct Lake mode is the primary focus of the DP-600. It reads data directly from OneLake Parquet files into the Power BI Analysis Services (AS) memory cache, completely bypassing the SQL engine.

### Fallback to DirectQuery Mode
Under specific conditions, Direct Lake mode will silently fall back to **DirectQuery mode**, which translates DAX queries into T-SQL queries and runs them on the SQL endpoint. This significantly degrades visual performance.

#### The 4 Critical Fallback Triggers:
1. **Memory Size Limit**: Each Fabric Capacity size (F2/F4 up to F2048) has a maximum memory limit (size on disk) allowed for semantic model columns in RAM. If a column load request exceeds this limit, fallback is triggered.
2. **Database-Level RLS/OLS**: If Row-Level Security (RLS) or Object-Level Security (OLS) is configured at the SQL Database layer (Lakehouse SQL Endpoint or Warehouse) instead of the Power BI Semantic Model layer, Power BI must fall back to SQL querying to enforce permissions.
3. **Views Usage**: Modeling relationships or pointing tables in the semantic model to a SQL view instead of physical Delta tables triggers fallback.
4. **Schema Out of Sync**: Renaming, adding, or deleting columns in the physical Delta table without synchronizing the semantic model's schema will cause a fallback.

### Direct Lake Verification Heuristics
To verify Direct Lake active status:
- In **DAX Studio / SQL Profiler**, connect to the XMLA endpoint and query the DMV:
  ```sql
  SELECT * FROM $System.DISCOVER_MDC_DETAILS;
  ```
- Check the `DirectLakeActive` column: `1` (True) means Direct Lake is active; `0` (False) means fallback has occurred.

---

## ⚙️ Medallion Architecture & Performance Optimizations
The Medallion pattern cleans and refines data as it flows from Files (Bronze) to aggregated Star Schemas (Gold).

### Delta Parquet Format Heuristics
- **ACID Transactions**: Supported via the transaction log (`_delta_log/` directory containing JSON commit files).
- **Time Travel**: Allows querying previous snapshots of a table using transaction log history.
- **V-Order (Fabric Proprietary)**: A sorting algorithm applied to Parquet files during write operations to optimize array loading for Power BI AS. Enabled by default in Fabric.
- **Z-Order (Clustered Indexing)**: Multidimensional clustering that rearranges data based on a list of columns (e.g., `order_date`) to reduce the volume of data scanned.
- **Liquid Clustering**: Replaces partitioning and Z-Order for dynamically growing large tables, maintaining sorting efficiency over time without manual partition tuning.

---

## 🏗️ Data Warehouse vs. Lakehouse SQL Endpoint
Understand the clear boundary between the Synapse Data Warehouse and the Lakehouse SQL Analytics Endpoint.

| Scenario / Goal | Choose Lakehouse SQL Endpoint | Choose Data Warehouse |
| :--- | :---: | :---: |
| Build Spark-based PySpark/Scala pipelines | **Yes** | No |
| Write raw CSV/JSON/images to storage | **Yes** | No |
| Run standard DDL/DML SQL (`INSERT`, `DELETE`) | No | **Yes** |
| Execute multi-table relational transactions | No | **Yes** |
| Define schemas, procedures, functions in SQL | No | **Yes** |
| Auto-sync metadata from storage to SQL schema | **Yes** (Automatic) | No |

---

## 📐 Semantic Models & DAX Optimization
A clean Star Schema layout is required for efficient Direct Lake memory paging.

### Relationship Cardinality Guidelines:
- Keep relationships **One-to-Many (`1:*`)** from Dimension (the "one" side) to Fact (the "many" side).
- Set cross-filtering to **Single**. Avoid bidirectional relationships as they introduce circular reference risks, performance degradation, and ambiguity.
- If multiple date fields are in a Fact table, create one active relationship (e.g., `OrderDate`) and inactive relationships (e.g., `ShipDate`), activating the inactive ones using `USERELATIONSHIP()` in DAX.

### DAX Optimization Rules:
- **CALCULATE**: Always modifies the current filter context.
- **FILTER**: Evaluates a condition row-by-row. Do not pass a large table as the first argument. E.g., replace `FILTER(FactSales, FactSales[Qty] > 10)` with simple columns inside `CALCULATE` or filter specific column values using `KEEPFILTERS(VALUES(FactSales[Qty]))`.
- **Context Transition**: The process that converts row context (like in `SUMX` or calculated columns) into filter context. Triggered automatically by calling a measure or enclosing an expression in `CALCULATE`.

---

## 🔒 Security & Governance

### Workspace Permissions Roles:
1. **Admin**: Full control. Modify members, edit items, delete workspace.
2. **Member**: Can publish, edit, and delete items. Can add members if permitted.
3. **Contributor**: Can create and edit items (code, tables, reports). Cannot manage members or share items. **Perfect role for developers.**
4. **Viewer**: Read-only access to report visuals. Cannot inspect underlying Delta files or Spark notebooks directly.

### Data Mesh & Endorsements:
- **Domains**: Logical groupings of workspaces (e.g., Finance domain) to delegate administration.
- **Lineage View**: Shows the path of data from sources (ADLS Gen2) -> Lakehouse -> Semantic Model -> Report.
- **Endorsements**:
  - *Promoted*: Can be set by any workspace member to indicate the item is ready.
  - *Certified*: Requires tenant-level authorization to mark the item as the enterprise source of truth.

---

## 📊 Capacity Sizing & Monitoring
Fabric compute is purchased via **Capacity Units (CUs)**.

### Smoothing & Throttling Engine:
- **Interactive Operations**: Visual interactions. Smoothed over a **10-minute window**.
- **Background Operations**: Pipelines, Spark runs, semantic model refreshes. Smoothed over a **24-hour window**.
- **Throttling Stages**:
  - *First*: Interactive queries are delayed (adding latency).
  - *Second*: Background runs are rejected.
  - *Mitigation*: Upgrade capacity tier (e.g., F2 to F4) or distribute heavy ETL jobs outside peak hours.
