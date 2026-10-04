# SQL Server Views — Complete Guide

## 1. What Is a View?

A **view** is a virtual table based on the result of a stored `SELECT` query. It doesn't store data itself (with one exception — indexed views, see below) — every time you query a view, SQL Server runs the underlying `SELECT` statement against the base tables.

```sql
CREATE VIEW dbo.vw_ActiveCustomers AS
SELECT CustomerID, FirstName, LastName, Email
FROM dbo.Customers
WHERE IsActive = 1;
```

```sql
SELECT * FROM dbo.vw_ActiveCustomers WHERE LastName = 'Smith';
```

Think of a view as a **saved, named query** — a reusable abstraction layer on top of your tables.

---

## 2. Why Use Views?

1. **Simplify complexity** — hide complicated joins/aggregations behind a simple `SELECT * FROM view`.
2. **Security / column-level restriction** — grant users access to a view instead of the base table, exposing only certain columns or rows (e.g., hide `SSN` or `Salary`).
3. **Abstraction from schema changes** — if underlying tables change structure, you can adjust the view definition without breaking client queries (to a degree).
4. **Reusability** — centralize common business logic (e.g., "active customers" definition) in one place instead of repeating the WHERE clause everywhere.
5. **Backward compatibility** — present an old table structure as a view after refactoring the real tables.

---

## 3. Basic Syntax

```sql
CREATE VIEW [schema.]ViewName [(Column1, Column2, ...)]
[WITH ENCRYPTION | SCHEMABINDING | VIEW_METADATA]
AS
    SELECT statement
[WITH CHECK OPTION];
```

- `ALTER VIEW` — modify an existing view's definition.
- `DROP VIEW ViewName` — remove a view.
- `CREATE OR ALTER VIEW` — create if not exists, or replace if it does (SQL Server 2016 SP1+).

---

## 4. WITH SCHEMABINDING

Binds the view to the schema of its underlying tables, preventing those tables/columns from being altered or dropped without first dropping/modifying the view.

```sql
CREATE VIEW dbo.vw_OrderSummary
WITH SCHEMABINDING
AS
SELECT o.OrderID, o.CustomerID, o.TotalAmount
FROM dbo.Orders o;   -- requires 2-part name (schema.table) with SCHEMABINDING
```

- **Required** if you want to create an **indexed (materialized) view**.
- Requires fully qualified table names (`schema.table`, not just `table`).
- Prevents accidental breaking changes to columns the view depends on.

---

## 5. WITH CHECK OPTION

Ensures that INSERT/UPDATE statements made *through* the view still satisfy the view's WHERE clause — prevents rows from being inserted/updated in a way that would make them disappear from the view's result set.

```sql
CREATE VIEW dbo.vw_ActiveCustomers AS
SELECT CustomerID, FirstName, LastName, IsActive
FROM dbo.Customers
WHERE IsActive = 1
WITH CHECK OPTION;

-- This fails because it would create a row invisible to the view:
UPDATE dbo.vw_ActiveCustomers SET IsActive = 0 WHERE CustomerID = 5;
```

---

## 6. WITH ENCRYPTION

Obfuscates the view's definition text in the system catalog (`sys.syscomments` / `OBJECT_DEFINITION()`), so users can't view the source via `sp_helptext` or SSMS scripting.

```sql
CREATE VIEW dbo.vw_Secret WITH ENCRYPTION AS
SELECT ... ;
```

Note: this is obfuscation, not real security — it can be reverse-engineered with third-party tools. Don't rely on it to protect sensitive logic.

---

## 7. Updatable Views

You can `INSERT`, `UPDATE`, or `DELETE` through a view, but only if it meets certain conditions:

- The view references **exactly one base table** for the columns being modified (no joins across multiple tables for the modified columns — unless using `INSTEAD OF` triggers).
- No aggregate functions (`SUM`, `COUNT`, `AVG`, etc.), `GROUP BY`, `DISTINCT`, `TOP`, or `UNION` in the view.
- All `NOT NULL` columns without defaults in the base table must be included in the view (for INSERTs to succeed).

For more complex views (multi-table joins, aggregates) that need to support DML, use an **`INSTEAD OF` trigger** to define custom logic:

```sql
CREATE TRIGGER trg_vw_OrderSummary_Insert
ON dbo.vw_OrderSummary
INSTEAD OF INSERT
AS
BEGIN
    INSERT INTO dbo.Orders (CustomerID, TotalAmount)
    SELECT CustomerID, TotalAmount FROM inserted;
END;
```

---

## 8. Indexed Views (Materialized Views)

Unlike a normal view, an **indexed view** physically stores the result set, because you create a **unique clustered index** on it. SQL Server then keeps it automatically in sync with the underlying tables (similar to how a table's data is maintained).

### Requirements
- View must be created `WITH SCHEMABINDING`.
- Must use two-part table names.
- No `*`, only fully qualified column names.
- No `UNION`, `OUTER JOIN`, subqueries, `TOP`, `DISTINCT` (with some exceptions), non-deterministic functions.
- If using `GROUP BY`, must include `COUNT_BIG(*)` in the SELECT list.
- First index created on the view must be a **unique clustered index**.
- Certain SET options must be correctly configured on the session creating/modifying data (e.g., `ANSI_NULLS ON`, `QUOTED_IDENTIFIER ON`).

```sql
CREATE VIEW dbo.vw_SalesByProduct
WITH SCHEMABINDING
AS
SELECT ProductID, COUNT_BIG(*) AS OrderCount, SUM(Quantity) AS TotalQty
FROM dbo.OrderDetails
GROUP BY ProductID;
GO

CREATE UNIQUE CLUSTERED INDEX IX_vw_SalesByProduct
ON dbo.vw_SalesByProduct (ProductID);
```

### Benefits
- Turns expensive aggregations/joins into pre-computed, indexed data — huge performance win for read-heavy reporting queries.
- The optimizer can automatically substitute an indexed view for a matching query, **even if the query doesn't reference the view directly** (Enterprise Edition does this automatically; other editions need `WITH (NOEXPAND)` hint).

### Costs
- Maintenance overhead — every INSERT/UPDATE/DELETE on base tables must also update the indexed view's stored data, similar to maintaining an extra index/table.
- Storage overhead (the result set is physically stored).
- Best suited for **read-heavy, relatively static** data (data warehousing, reporting) — not ideal for highly transactional (OLTP) source tables.

---

## 9. Partitioned Views

Combines data from multiple structurally identical tables (often split by date range or region) into a single logical view, typically using `UNION ALL`.

```sql
CREATE VIEW dbo.vw_Orders_All AS
SELECT * FROM dbo.Orders_2024
UNION ALL
SELECT * FROM dbo.Orders_2025
UNION ALL
SELECT * FROM dbo.Orders_2026;
```

- **Local partitioned view** — all member tables on the same server.
- **Distributed partitioned view** — member tables spread across multiple servers (used in older scale-out architectures; largely superseded by native table partitioning and modern distributed database features).
- Largely superseded by SQL Server's built-in **table partitioning** feature for single-server scenarios, but still used for cross-server sharding patterns.

---

## 10. System / Catalog Views

SQL Server itself exposes metadata through built-in system views, e.g.:

```sql
SELECT * FROM sys.views;
SELECT * FROM sys.objects WHERE type = 'V';
SELECT * FROM INFORMATION_SCHEMA.VIEWS;
```

Useful system views/functions for working with views:
```sql
EXEC sp_helptext 'dbo.vw_ActiveCustomers';   -- shows the view's SQL definition (if not encrypted)
SELECT OBJECT_DEFINITION(OBJECT_ID('dbo.vw_ActiveCustomers'));
EXEC sp_depends 'dbo.vw_ActiveCustomers';    -- shows dependent/depended-on objects (legacy)
SELECT * FROM sys.dm_sql_referenced_entities('dbo.vw_ActiveCustomers', 'OBJECT');
```

---

## 11. Views vs. Other Objects

| Object | Stores Data? | Parameters? | Best For |
|---|---|---|---|
| **View** | No (unless indexed) | No | Reusable SELECT logic, security abstraction |
| **Indexed View** | Yes (physically) | No | Precomputed aggregates for reporting/performance |
| **Table-Valued Function (TVF)** | No | Yes | Parameterized, reusable query logic |
| **Stored Procedure** | No | Yes | Multi-statement logic, DML, control flow |
| **CTE (Common Table Expression)** | No | No (scoped to one query) | One-off, query-scoped reusable subquery |
| **Temp Table** | Yes (physically, in tempdb) | N/A | Intermediate result sets within a session/batch |

Key distinction: a **view has no parameters** — if you need to pass in filter values dynamically with proper optimization, an **inline table-valued function** is usually a better choice (it behaves like a parameterized view).

```sql
CREATE FUNCTION dbo.fn_OrdersByCustomer(@CustomerID INT)
RETURNS TABLE
AS
RETURN (SELECT * FROM dbo.Orders WHERE CustomerID = @CustomerID);

SELECT * FROM dbo.fn_OrdersByCustomer(5);
```

---

## 12. Performance Considerations

- A **regular (non-indexed) view is not "compiled" or cached** — its definition is essentially expanded and merged into the outer query at execution time. It's a convenience/abstraction, not a performance feature by itself.
- Nesting views many layers deep (views calling views calling views) can confuse the optimizer and hurt performance — check the actual execution plan rather than assuming it's fine.
- `SELECT *` inside a view definition is risky: if the base table's columns change, the view can silently return wrong/missing data until refreshed with `sp_refreshview`.
- Use `sp_refreshview 'dbo.ViewName'` after altering underlying table structures to refresh cached metadata.
- For real performance gains from a view-like object, use an **indexed view** (materialized) or a **TVF** with good predicate pushdown.

---

## 13. Security Use Case Example

```sql
-- Base table has sensitive columns
CREATE TABLE dbo.Employees (
    EmployeeID INT PRIMARY KEY,
    Name NVARCHAR(100),
    Salary MONEY,
    SSN CHAR(11)
);

-- View exposes only non-sensitive columns
CREATE VIEW dbo.vw_EmployeeDirectory AS
SELECT EmployeeID, Name
FROM dbo.Employees;

-- Grant access to the view only, not the base table
GRANT SELECT ON dbo.vw_EmployeeDirectory TO [SomeRole];
DENY SELECT ON dbo.Employees TO [SomeRole];
```

This lets a role query employee names without ever being able to see `Salary` or `SSN`, even through ad-hoc queries.

---

## 14. Common Pitfalls

1. **Assuming views improve performance** — they don't, unless indexed. A view over a bad query is still a bad query.
2. **`SELECT *` in view definitions** — breaks silently when the base table schema changes; always list columns explicitly.
3. **Deeply nested views** — hard to debug, hard for the optimizer to simplify.
4. **Forgetting `sp_refreshview`** after schema changes — leads to stale metadata errors.
5. **Trying to update through complex views** without `INSTEAD OF` triggers — DML will simply fail with an error.
6. **Using `WITH ENCRYPTION`** thinking it's real security — it only obscures the definition, doesn't prevent all forms of reverse engineering, and prevents Enterprise Edition's automatic script generation/documentation tools from working on it.

---

## 15. Quick Command Reference

```sql
-- Create / Replace
CREATE OR ALTER VIEW dbo.MyView AS SELECT ... ;

-- Drop
DROP VIEW dbo.MyView;

-- Refresh metadata after base table changes
EXEC sp_refreshview 'dbo.MyView';

-- View definition
EXEC sp_helptext 'dbo.MyView';

-- List all views in the database
SELECT name, create_date, modify_date FROM sys.views;

-- Check if a view is indexed
SELECT v.name, i.name AS IndexName, i.type_desc
FROM sys.views v
JOIN sys.indexes i ON v.object_id = i.object_id
WHERE i.index_id > 0;
```

---

## 16. Summary

| Feature | Purpose |
|---|---|
| Basic View | Reusable, saved SELECT statement; simplifies and secures access |
| `SCHEMABINDING` | Locks underlying schema; required for indexed views |
| `CHECK OPTION` | Prevents DML through the view from violating its WHERE clause |
| `ENCRYPTION` | Obfuscates the view definition (not true security) |
| Indexed View | Physically materializes results; big performance gains for aggregates/joins on read-heavy data |
| Partitioned View | Unions multiple similarly-structured tables into one logical object |
| `INSTEAD OF` Trigger | Enables DML through views that aren't naturally updatable |

