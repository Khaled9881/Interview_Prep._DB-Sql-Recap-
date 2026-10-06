# SQL Server Indexes — Complete Guide

## 1. What Is an Index?

An index is a database object that speeds up data retrieval by creating a separate, sorted structure that points to the actual rows in a table. Without an index, SQL Server must perform a **table scan** — reading every row to find matches. With an index, SQL Server can perform an **index seek** — jumping directly to the relevant rows, similar to how a book's index lets you find a topic without reading every page.

The tradeoff: indexes speed up reads but slow down writes (INSERT/UPDATE/DELETE) because the index structure must also be updated, and they consume additional disk space.

---

## 2. The B-Tree Structure

Almost all SQL Server indexes (clustered and non-clustered) are organized as a **balanced tree (B-tree)**:

```
                [Root Node]
               /     |      \
        [Intermediate Nodes - Level N]
         /    |    |    |    |    \
   [Leaf Node][Leaf Node][Leaf Node]...
```

- **Root node** — single top-level page, entry point for all searches.
- **Intermediate levels** — non-leaf pages that route the search down the tree (like a table of contents).
- **Leaf level** — the bottom level, containing either the actual data (clustered index) or pointers to the data (non-clustered index).

Each page is 8 KB. The number of levels depends on table size — a well-organized index can locate a row in a huge table with very few page reads (typically 3–5 for millions of rows), because the tree is *balanced* and grows logarithmically.

---

## 3. Clustered Index

- Determines the **physical order** of data rows in the table.
- A table can have **only one** clustered index because data can physically be sorted only one way.
- The leaf level of a clustered index **is** the actual data — there's no separate lookup step.
- If a table has no clustered index, it's called a **heap** (unordered storage).
- Often created automatically on the **Primary Key**, but this is not mandatory — you can have a PK without it being clustered.

```sql
CREATE CLUSTERED INDEX IX_Orders_OrderDate
ON dbo.Orders (OrderDate);
```

**Choosing a good clustering key:**
- Narrow (fewer bytes — it gets copied into every non-clustered index).
- Unique or nearly unique.
- Static (rarely updated — updating it means physically moving the row).
- Ever-increasing (e.g., IDENTITY, sequential GUID) to avoid page splits from random inserts.

---

## 4. Non-Clustered Index

- A **separate structure** from the data, with its own B-tree.
- The leaf level contains the index key column(s) plus a **row locator**:
  - If the table has a clustered index → row locator = the clustering key.
  - If the table is a heap → row locator = a physical Row ID (RID).
- A table can have **up to 999 non-clustered indexes** (practically, far fewer is best).
- Because it's a separate structure, looking up full row data often requires a **"Key Lookup"** (or "RID Lookup") back to the clustered index/heap — an extra I/O step.

```sql
CREATE NONCLUSTERED INDEX IX_Customers_LastName
ON dbo.Customers (LastName);
```

---

## 5. Composite (Multi-Column) Indexes

An index on more than one column. Column **order matters enormously**:

```sql
CREATE NONCLUSTERED INDEX IX_Orders_CustID_OrderDate
ON dbo.Orders (CustomerID, OrderDate);
```

- This index is useful for queries filtering on `CustomerID` alone, or `CustomerID + OrderDate` together.
- It is **not** efficiently used for queries filtering on `OrderDate` alone (leftmost-column rule, similar to a phone book sorted by last name then first name — you can't jump to "born in June" efficiently).
- Put the most selective / most frequently filtered column first, generally.

---

## 6. Covering Indexes & INCLUDE

A **covering index** contains all the columns a query needs, so SQL Server never has to go back to the base table (no Key Lookup).

Use `INCLUDE` to add columns to the **leaf level only** (not part of the sort key) — cheaper than adding them to the key:

```sql
CREATE NONCLUSTERED INDEX IX_Orders_Covering
ON dbo.Orders (CustomerID)
INCLUDE (OrderDate, TotalAmount);
```

This satisfies:
```sql
SELECT OrderDate, TotalAmount FROM dbo.Orders WHERE CustomerID = 5;
```
entirely from the index — an **Index Seek** with no Key Lookup.

---

## 7. Unique Index

Enforces uniqueness of key values (like a UNIQUE constraint, but more flexible — you can control fill factor, filtering, etc.):

```sql
CREATE UNIQUE NONCLUSTERED INDEX IX_Users_Email
ON dbo.Users (Email);
```

---

## 8. Filtered Index

An index built on a **subset of rows** using a WHERE clause — smaller, faster, and cheaper to maintain than a full-table index.

```sql
CREATE NONCLUSTERED INDEX IX_Orders_Active
ON dbo.Orders (OrderDate)
WHERE IsCancelled = 0;
```

Great for columns with skewed data (e.g., a status column that's 95% "Completed" and 5% "Pending" — index only the "Pending" rows).

---

## 9. Columnstore Index

Stores data **column-wise** rather than row-wise, using heavy compression. Designed for data warehousing / analytical (OLAP) workloads with large aggregations and scans.

```sql
CREATE CLUSTERED COLUMNSTORE INDEX IX_Sales_CS
ON dbo.FactSales;

CREATE NONCLUSTERED COLUMNSTORE INDEX IX_Sales_NCCS
ON dbo.FactSales (ProductID, Quantity, SalesAmount);
```

- Dramatically reduces I/O and storage for analytical queries (10x+ compression common).
- Can coexist with a traditional rowstore B-tree via nonclustered columnstore on an OLTP table (real-time operational analytics).

---

## 10. Full-Text Index

Enables linguistic searches (word/phrase matching, stemming) inside large text columns, used with `CONTAINS` / `FREETEXT` predicates. Separate subsystem from regular B-tree indexes.

```sql
CREATE FULLTEXT INDEX ON dbo.Articles (Body)
KEY INDEX PK_Articles;
```

---

## 11. XML / Spatial Indexes

- **XML Index** — speeds up queries against `xml`-typed columns (primary XML index + optional secondary PATH/VALUE/PROPERTY indexes).
- **Spatial Index** — speeds up queries on `geometry`/`geography` data types (used for mapping/location data).

---

## 12. Hash Indexes (Memory-Optimized Tables)

Used only with In-Memory OLTP (memory-optimized tables). Backed by a hash table rather than a B-tree — extremely fast for equality lookups, but poor for range scans.

```sql
CREATE TABLE dbo.Session_MemOpt (
    SessionID INT NOT NULL PRIMARY KEY NONCLUSTERED HASH WITH (BUCKET_COUNT = 1000000),
    Data NVARCHAR(100)
) WITH (MEMORY_OPTIMIZED = ON);
```

---

## 13. How the Query Optimizer Uses Indexes

When you run a query, the optimizer chooses among:

| Operation | Meaning |
|---|---|
| **Clustered Index Scan** | Reads entire table in clustered order (like a full table scan). |
| **Clustered Index Seek** | Navigates directly to matching rows via the B-tree. |
| **Index Seek** (non-clustered) | Navigates directly within a non-clustered index. |
| **Index Scan** (non-clustered) | Reads the entire non-clustered index. |
| **Key Lookup** | After a non-clustered seek, goes back to the clustered index to fetch remaining columns not in the index. |
| **RID Lookup** | Same as Key Lookup but for heaps (uses Row ID). |

A **Seek** is generally far cheaper than a **Scan**, especially on large tables. You can inspect this via the **Actual/Estimated Execution Plan** in SSMS.

---

## 14. Index Maintenance

### Fragmentation
Over time, INSERT/UPDATE/DELETE operations cause **page splits** and gaps, leading to:
- **Internal fragmentation** — pages not fully packed (wastes space, more I/O per read).
- **External fragmentation** — logical page order doesn't match physical disk order (poor for sequential scans).

Check fragmentation:
```sql
SELECT index_id, avg_fragmentation_in_percent, page_count
FROM sys.dm_db_index_physical_stats(DB_ID(), OBJECT_ID('dbo.Orders'), NULL, NULL, 'LIMITED');
```

### Reorganize vs Rebuild

| Fragmentation % | Recommended Action |
|---|---|
| 5–30% | `ALTER INDEX ... REORGANIZE` (online, light-weight, defragments leaf level only) |
| > 30% | `ALTER INDEX ... REBUILD` (recreates index from scratch, can be ONLINE in Enterprise/certain editions, updates statistics) |

```sql
ALTER INDEX IX_Orders_OrderDate ON dbo.Orders REORGANIZE;
ALTER INDEX IX_Orders_OrderDate ON dbo.Orders REBUILD WITH (ONLINE = ON, FILLFACTOR = 90);
```

### Fill Factor
Percentage of each leaf page to fill during index creation/rebuild, leaving room for future inserts before a page split occurs.
- Default = 0/100 (fully packed) — best for read-heavy, mostly-static tables.
- Lower value (e.g., 80–90) — better for tables with frequent inserts/updates, at the cost of using more storage.

---

## 15. Statistics

SQL Server keeps **statistics** (histograms of data distribution) on indexed and non-indexed columns to help the optimizer estimate row counts (cardinality) and choose good execution plans.

- Auto-created/updated by default (`AUTO_CREATE_STATISTICS`, `AUTO_UPDATE_STATISTICS`).
- Stale statistics are one of the most common causes of bad execution plans.
- Manually update: 
```sql
UPDATE STATISTICS dbo.Orders;
-- or fully recompute:
UPDATE STATISTICS dbo.Orders WITH FULLSCAN;
```

---

## 16. Finding Missing or Unused Indexes

**Missing index suggestions** (from the optimizer's perspective):
```sql
SELECT mid.statement, migs.avg_user_impact, mid.equality_columns, mid.inequality_columns, mid.included_columns
FROM sys.dm_db_missing_index_details mid
JOIN sys.dm_db_missing_index_group_stats migs
  ON mid.index_handle = migs.group_handle
ORDER BY migs.avg_user_impact DESC;
```

**Unused indexes** (candidates for removal — they only cost write overhead):
```sql
SELECT OBJECT_NAME(s.object_id) AS TableName, i.name AS IndexName,
       s.user_seeks, s.user_scans, s.user_lookups, s.user_updates
FROM sys.dm_db_index_usage_stats s
JOIN sys.indexes i ON s.object_id = i.object_id AND s.index_id = i.index_id
WHERE s.database_id = DB_ID()
ORDER BY s.user_updates DESC;
```
(An index with high `user_updates` but near-zero `user_seeks/scans/lookups` is a strong candidate to drop.)

---

## 17. Key Design Best Practices

1. **Every table should generally have a clustered index** — heaps perform poorly for most workloads.
2. Index columns used frequently in `WHERE`, `JOIN`, `ORDER BY`, and `GROUP BY` clauses.
3. Favor **narrow, selective** keys — an index on a column with only 2 distinct values (e.g., a boolean) rarely helps.
4. Use **covering indexes** for hot, frequently-run queries to eliminate Key Lookups.
5. Avoid **over-indexing** — every index adds overhead to every INSERT/UPDATE/DELETE. More isn't always better.
6. Watch out for indexes that are near-duplicates of each other (e.g., `(A,B)` and `(A,B,C)` — usually only need the wider one).
7. Rebuild/reorganize indexes and update statistics on a maintenance schedule (or use Ola Hallengren's popular open-source maintenance scripts).
8. Use **filtered indexes** for sparse/skewed columns instead of full-table indexes.
9. Be mindful of the clustering key's impact — it's embedded in *every* non-clustered index as the row locator.
10. Monitor with execution plans and DMVs rather than guessing.

---

## 18. Quick Command Reference

```sql
-- Create
CREATE [UNIQUE] [CLUSTERED|NONCLUSTERED] INDEX IndexName
ON Schema.Table (Col1 [ASC|DESC], Col2 ...)
[INCLUDE (Col3, Col4)]
[WHERE <filter_predicate>]
[WITH (FILLFACTOR = 90, ONLINE = ON, DATA_COMPRESSION = PAGE)];

-- View indexes on a table
EXEC sp_helpindex 'dbo.Orders';
-- or
SELECT * FROM sys.indexes WHERE object_id = OBJECT_ID('dbo.Orders');

-- Disable / Enable
ALTER INDEX IX_Name ON dbo.Orders DISABLE;
ALTER INDEX IX_Name ON dbo.Orders REBUILD; -- re-enables

-- Drop
DROP INDEX IX_Name ON dbo.Orders;
```

---

## 19. Summary Table

| Index Type | Best For | Notes |
|---|---|---|
| Clustered | Range queries, primary access path | Only 1 per table; defines physical order |
| Non-Clustered | Selective lookups on non-key columns | Up to 999 per table; may need Key Lookup |
| Composite | Multi-column filters | Column order matters |
| Covering (INCLUDE) | Eliminating Key Lookups | Extra storage, faster reads |
| Unique | Enforcing uniqueness + lookup speed | Also usable as a constraint |
| Filtered | Skewed / sparse data | Smaller, cheaper to maintain |
| Columnstore | Data warehousing, analytics, aggregation | Huge compression, poor for single-row lookups |
| Full-Text | Text/word search | Separate subsystem |
| XML / Spatial | XML or geo data queries | Specialized use cases |
| Hash | In-Memory OLTP equality lookups | No range scan support |

