# SQL Server Stored Procedures — Complete Guide (+ EF Core Integration)

## 1. What Is a Stored Procedure?

A **stored procedure (SP)** is a precompiled, named collection of one or more T-SQL statements stored in the database. Unlike a view (a single SELECT), a stored procedure can contain multiple statements, control-of-flow logic (`IF`, `WHILE`, `TRY/CATCH`), parameters, variables, and can perform `SELECT`, `INSERT`, `UPDATE`, `DELETE`, transaction control, and even call other procedures.

```sql
CREATE PROCEDURE dbo.usp_GetCustomerOrders
    @CustomerID INT
AS
BEGIN
    SET NOCOUNT ON;
    SELECT OrderID, OrderDate, TotalAmount
    FROM dbo.Orders
    WHERE CustomerID = @CustomerID;
END;
```

```sql
EXEC dbo.usp_GetCustomerOrders @CustomerID = 5;
```

---

## 2. Why Use Stored Procedures?

1. **Performance** — execution plans are compiled and cached, reused across calls (avoids recompiling the same query logic repeatedly).
2. **Reduced network traffic** — client sends one short `EXEC` call instead of a large SQL string.
3. **Security** — grant `EXECUTE` permission on the procedure without granting direct table access; parameters help prevent SQL injection when used properly.
4. **Encapsulation / abstraction** — business logic lives in one place; underlying schema changes can be absorbed inside the procedure without changing calling code.
5. **Maintainability** — centralizes complex logic (multi-step transactions, business rules) instead of duplicating it across application code.
6. **Reduced SQL injection risk** — parameters are strongly typed and passed separately from the command text (as long as you don't build dynamic SQL by concatenating strings inside the procedure).

---

## 3. Syntax Essentials

```sql
CREATE OR ALTER PROCEDURE [schema.]ProcName
    @Param1 INT,
    @Param2 NVARCHAR(50) = NULL,      -- default value
    @Param3 INT OUTPUT                 -- output parameter
[WITH ENCRYPTION | RECOMPILE | EXECUTE AS <user>]
AS
BEGIN
    SET NOCOUNT ON;
    -- statements
END;
```

- `ALTER PROCEDURE` — modify existing.
- `DROP PROCEDURE ProcName` — remove.
- `CREATE OR ALTER PROCEDURE` — create-or-replace shorthand (2016 SP1+).
- `EXEC` / `EXECUTE` — run it.

### Parameters
```sql
CREATE PROCEDURE dbo.usp_UpdateStock
    @ProductID INT,
    @QtyChange INT,
    @NewQty INT OUTPUT
AS
BEGIN
    UPDATE dbo.Products SET Stock = Stock + @QtyChange WHERE ProductID = @ProductID;
    SELECT @NewQty = Stock FROM dbo.Products WHERE ProductID = @ProductID;
END;
GO

DECLARE @Result INT;
EXEC dbo.usp_UpdateStock @ProductID = 1, @QtyChange = -5, @NewQty = @Result OUTPUT;
SELECT @Result;
```

### Return Values
```sql
CREATE PROCEDURE dbo.usp_CheckStock @ProductID INT
AS
BEGIN
    IF EXISTS (SELECT 1 FROM dbo.Products WHERE ProductID = @ProductID AND Stock > 0)
        RETURN 1;
    RETURN 0;
END;
GO

DECLARE @RC INT;
EXEC @RC = dbo.usp_CheckStock @ProductID = 1;
```
`RETURN` sends back a single integer status code (traditionally used for success/failure, not data). Use `OUTPUT` parameters or `SELECT` result sets for actual data.

---

## 4. Control-of-Flow & Error Handling

```sql
CREATE PROCEDURE dbo.usp_TransferFunds
    @FromAccount INT, @ToAccount INT, @Amount MONEY
AS
BEGIN
    SET NOCOUNT ON;
    BEGIN TRY
        BEGIN TRANSACTION;

        UPDATE dbo.Accounts SET Balance = Balance - @Amount WHERE AccountID = @FromAccount;
        IF (SELECT Balance FROM dbo.Accounts WHERE AccountID = @FromAccount) < 0
            THROW 50001, 'Insufficient funds.', 1;

        UPDATE dbo.Accounts SET Balance = Balance + @Amount WHERE AccountID = @ToAccount;

        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        IF XACT_STATE() <> 0
            ROLLBACK TRANSACTION;

        THROW;  -- re-raise the original error to the caller
    END CATCH
END;
```

Key elements: `TRY/CATCH`, `BEGIN/COMMIT/ROLLBACK TRANSACTION`, `THROW` (2012+, preferred) vs `RAISERROR` (legacy, more formatting options), `XACT_STATE()` to check transaction status before rollback.

---

## 5. Execution Plan Caching & Recompilation

SQL Server compiles a query plan the first time a procedure runs and **caches** it for reuse — this is the biggest performance advantage over ad-hoc SQL.

### Parameter Sniffing
The optimizer builds the cached plan based on the **parameter values used on the first call**. If those values are atypical (e.g., a highly selective vs. a very common value), subsequent calls with different data distributions can get a suboptimal plan reused. This is called **parameter sniffing** and is one of the most common real-world SP performance issues.

**Mitigations:**
```sql
-- Force a fresh plan every execution (avoids sniffing but adds compile overhead)
CREATE PROCEDURE dbo.usp_Example @X INT
WITH RECOMPILE
AS ...

-- Force recompiling just a specific statement
SELECT * FROM dbo.Orders WHERE CustomerID = @X OPTION (RECOMPILE);

-- Use local variables to defeat sniffing (optimizer uses average density instead)
CREATE PROCEDURE dbo.usp_Example @X INT
AS
BEGIN
    DECLARE @LocalX INT = @X;
    SELECT * FROM dbo.Orders WHERE CustomerID = @LocalX;
END;

-- OPTIMIZE FOR hint — compile assuming a specific/typical value
SELECT * FROM dbo.Orders WHERE CustomerID = @X
OPTION (OPTIMIZE FOR (@X = 100));

-- OPTIMIZE FOR UNKNOWN — use average statistical distribution, ignore actual param
OPTION (OPTIMIZE FOR UNKNOWN);
```

Other cache-invalidating events: statistics updates, index rebuilds, schema changes, or explicit `sp_recompile`.

---

## 6. Security

```sql
GRANT EXECUTE ON dbo.usp_GetCustomerOrders TO [AppRole];
DENY SELECT ON dbo.Orders TO [AppRole];  -- app can call SP but not query the table directly
```

- **Ownership chaining** — if the SP and the underlying table have the same owner, SQL Server skips checking the caller's permission on the table itself (only checks EXECUTE on the SP). This is the classic way to expose controlled data access.
- `EXECUTE AS` — run the procedure under a different security context (e.g., `EXECUTE AS OWNER`) — useful for cross-schema/cross-database access without granting broad rights to end users.
- Parameterized inputs (native SP parameters) are inherently safer against SQL injection than string concatenation. **However**, if you build dynamic SQL inside the procedure via `EXEC(@sql)` or `sp_executesql` with concatenated strings, you can still introduce injection — always use `sp_executesql` with parameters for dynamic SQL, never raw concatenation.

```sql
-- SAFE dynamic SQL
EXEC sp_executesql
    N'SELECT * FROM dbo.Orders WHERE CustomerID = @CustID',
    N'@CustID INT',
    @CustID = @CustomerID;
```

---

## 7. Stored Procedures vs. Other Objects

| Object | Can Modify Data? | Parameters? | Compiled Plan Cached? | Can Be Used in a `SELECT ... FROM`? |
|---|---|---|---|---|
| Stored Procedure | Yes | Yes (in/out) | Yes | No — called via `EXEC` |
| View | No (unless via `INSTEAD OF` trigger) | No | Not separately (merged into caller's plan) | Yes |
| Scalar/Table-Valued Function | No (generally, no side effects allowed) | Yes | Yes | Yes (TVFs) |
| Ad-hoc / dynamic SQL | Yes | Via `sp_executesql` | Plan cached but easily bloats cache with "one-off" plans unless parameterized | No |

---

## 8. Common Best Practices

1. Always `SET NOCOUNT ON` at the top — suppresses the "N rows affected" messages, reducing network chatter and improving performance for procedures with loops/multiple statements.
2. Use `TRY/CATCH` with proper transaction rollback for any multi-statement write logic.
3. Avoid `SELECT *` — return only needed columns for predictable output shape (important for EF Core mapping, see below).
4. Be explicit about parameter data types/lengths matching the underlying column definitions to avoid implicit conversions that block index usage.
5. Keep procedures focused (single responsibility) rather than giant multi-purpose "mega-procs" with many optional parameters and branching logic — those are especially prone to bad parameter-sniffing plans.
6. Avoid excessive dynamic SQL unless necessary (e.g., dynamic search filters) — and when necessary, always parameterize with `sp_executesql`.
7. Use schema qualification (`dbo.ProcName`) to avoid unnecessary plan cache lookups/metadata resolution overhead.

---

## 9. Relation to EF Core

Entity Framework Core is primarily built around **LINQ-to-SQL translation** — you write LINQ queries in C#, and EF Core's query pipeline generates parameterized SQL automatically (typically ad-hoc, auto-parameterized statements rather than calling stored procedures). But EF Core has first-class support for calling **existing** stored procedures where needed.

### 9.1 Calling Stored Procedures for Queries

```csharp
var orders = await context.Orders
    .FromSqlInterpolated($"EXEC dbo.usp_GetCustomerOrders @CustomerID = {customerId}")
    .ToListAsync();
```

- `FromSqlInterpolated` / `FromSqlRaw` map the SP's result set to an entity type (or a keyless entity type, see below).
- **Must return columns matching the target entity type's shape** (or a subset that EF Core can map, if configured).
- `FromSqlInterpolated` automatically parameterizes interpolated values — safe against SQL injection (avoid `FromSqlRaw` with manual string concatenation).
- Composing further LINQ (`.Where()`, `.OrderBy()`) on top of `FromSql*` results is limited — EF Core treats the SP call as an opaque composed subquery in some cases, but not all LINQ operators can be pushed after a raw SQL/SP call, and you cannot generally re-shape columns after the fact.

### 9.2 Keyless Entity Types (for SP results that don't map to a table)

If a stored procedure returns a shape that doesn't correspond to any tracked entity (e.g., a custom report), define a **keyless entity type**:

```csharp
public class CustomerOrderSummary
{
    public int CustomerID { get; set; }
    public int OrderCount { get; set; }
    public decimal TotalSpent { get; set; }
}

// In OnModelCreating:
modelBuilder.Entity<CustomerOrderSummary>().HasNoKey();
```

```csharp
var results = await context.Set<CustomerOrderSummary>()
    .FromSqlInterpolated($"EXEC dbo.usp_CustomerSummary")
    .ToListAsync();
```

### 9.3 Executing Non-Query Stored Procedures (INSERT/UPDATE/DELETE logic)

For SPs that don't return rows (e.g., perform an update and return only a row count or output param):

```csharp
var rowsAffected = await context.Database
    .ExecuteSqlInterpolatedAsync($"EXEC dbo.usp_UpdateStock @ProductID = {productId}, @QtyChange = {qty}");
```

For **output parameters**, drop to `DbCommand` directly (EF Core's high-level `ExecuteSql*` methods don't have a clean built-in way to retrieve OUTPUT params):

```csharp
var conn = context.Database.GetDbConnection();
using var cmd = conn.CreateCommand();
cmd.CommandText = "dbo.usp_UpdateStock";
cmd.CommandType = CommandType.StoredProcedure;

cmd.Parameters.Add(new SqlParameter("@ProductID", productId));
cmd.Parameters.Add(new SqlParameter("@QtyChange", qtyChange));
var outputParam = new SqlParameter("@NewQty", SqlDbType.Int) { Direction = ParameterDirection.Output };
cmd.Parameters.Add(outputParam);

await context.Database.OpenConnectionAsync();
await cmd.ExecuteNonQueryAsync();
int newQty = (int)outputParam.Value;
```

### 9.4 Mapping Entity CRUD Operations to Stored Procedures (EF Core 7+)

Since **EF Core 7**, you can configure entities so that EF Core's own `SaveChanges()` uses your stored procedures instead of auto-generated INSERT/UPDATE/DELETE statements — useful when DBAs require all writes to go through SPs (common in enterprise/regulated environments), or when you need custom audit logic embedded server-side.

```csharp
modelBuilder.Entity<Product>()
    .InsertUsingStoredProcedure("usp_Product_Insert",
        s => s.HasParameter(p => p.Name)
              .HasParameter(p => p.Price)
              .HasResultColumn(p => p.ProductID))   // e.g., returns the new IDENTITY value
    .UpdateUsingStoredProcedure("usp_Product_Update",
        s => s.HasParameter(p => p.ProductID)
              .HasParameter(p => p.Name)
              .HasParameter(p => p.Price)
              .HasRowsAffectedParameter())
    .DeleteUsingStoredProcedure("usp_Product_Delete",
        s => s.HasParameter(p => p.ProductID)
              .HasRowsAffectedParameter());
```

Now `context.Products.Add(newProduct); await context.SaveChangesAsync();` transparently calls `usp_Product_Insert` under the hood instead of generating a raw `INSERT`. This gives you the ORM convenience of change tracking + the governance/performance benefits of stored procedures.

### 9.5 Why Teams Combine EF Core + Stored Procedures

| Scenario | Typical Approach |
|---|---|
| Simple CRUD, straightforward filters | Plain EF Core LINQ (auto-generated parameterized SQL) — no SP needed |
| Complex reporting query, heavy joins/aggregation | Stored procedure + `FromSqlInterpolated` into a keyless entity |
| DBA governance requires all writes via SP (auditing, security policy) | `InsertUsingStoredProcedure` / `UpdateUsingStoredProcedure` / `DeleteUsingStoredProcedure` (EF Core 7+) |
| Multi-step transactional business logic best kept server-side | Stored procedure via `ExecuteSqlInterpolatedAsync` |
| Legacy database with existing SP-based data access layer | `FromSqlRaw`/`FromSqlInterpolated` wrapping existing SPs |

### 9.6 Caveats When Using SPs with EF Core

- EF Core's **change tracking** doesn't automatically understand what a raw SP call does to the database — if the SP updates related rows or triggers cascading effects, EF Core's in-memory tracked graph won't automatically reflect that; you may need to re-query or manually update tracked entities.
- Composability with LINQ after `FromSql*` is limited — you generally cannot arbitrarily reshape/project further with `.Select()` on top of many raw SQL/SP calls without EF Core producing a `SELECT * FROM (EXEC ...)` style composition (which SQL Server does **not** actually support directly for `EXEC` — this is why full LINQ composability after an SP call is restricted; `FromSqlRaw`/`Interpolated` for **stored procedure calls specifically** doesn't support further server-side query composition the way it does for plain SQL statements).
- Column names/order returned by the SP must exactly line up with the mapped type's properties (or use explicit column-to-property mapping via Fluent API).
- Migrations don't manage stored procedures automatically — you typically maintain SP scripts separately (raw SQL migrations, or a versioned SQL script folder) rather than through EF Core's model-based migration system, though you can embed `CREATE PROCEDURE` scripts inside a migration's `Up()`/`Down()` via `migrationBuilder.Sql(...)`.

```csharp
// Example: creating a stored procedure via an EF Core migration
protected override void Up(MigrationBuilder migrationBuilder)
{
    migrationBuilder.Sql(@"
        CREATE PROCEDURE dbo.usp_GetCustomerOrders
            @CustomerID INT
        AS
        BEGIN
            SET NOCOUNT ON;
            SELECT OrderID, OrderDate, TotalAmount
            FROM dbo.Orders WHERE CustomerID = @CustomerID;
        END");
}
```

---

## 10. Quick Command Reference

```sql
-- Create / Replace
CREATE OR ALTER PROCEDURE dbo.usp_Name @P1 INT AS BEGIN ... END;

-- Execute
EXEC dbo.usp_Name @P1 = 1;

-- Drop
DROP PROCEDURE dbo.usp_Name;

-- View definition
EXEC sp_helptext 'dbo.usp_Name';

-- Force recompile next execution
EXEC sp_recompile 'dbo.usp_Name';

-- List all procedures
SELECT name, create_date, modify_date FROM sys.procedures;
```

---

## 11. Summary

| Feature | Purpose |
|---|---|
| Parameters (IN/OUT) | Pass and retrieve scalar values |
| `RETURN` | Single integer status code |
| Cached execution plan | Performance benefit vs. ad-hoc SQL |
| Parameter sniffing | Common pitfall — plan reused for atypical parameter values |
| `TRY/CATCH` + transactions | Robust multi-statement error handling |
| Ownership chaining / `EXECUTE AS` | Security — grant execute without direct table access |
| EF Core `FromSqlInterpolated` | Query an SP's result set into entities/keyless types |
| EF Core `ExecuteSqlInterpolatedAsync` | Run a non-query SP (INSERT/UPDATE/DELETE style) |
| EF Core 7+ `...UsingStoredProcedure` | Route `SaveChanges()` CRUD through your own SPs |

