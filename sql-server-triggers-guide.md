# SQL Server Triggers — Complete Guide

## 1. What Is a Trigger?

A **trigger** is a special kind of stored procedure that automatically executes ("fires") in response to an event — a data modification (`INSERT`/`UPDATE`/`DELETE`), a schema change (`CREATE`/`ALTER`/`DROP`), or a server/login-level event — rather than being called explicitly by name.

Unlike normal stored procedures, you never `EXEC` a trigger directly; SQL Server invokes it automatically when the triggering event occurs.

```sql
CREATE TRIGGER trg_Orders_AfterInsert
ON dbo.Orders
AFTER INSERT
AS
BEGIN
    SET NOCOUNT ON;
    INSERT INTO dbo.OrderAudit (OrderID, Action, ActionDate)
    SELECT OrderID, 'INSERTED', GETDATE() FROM inserted;
END;
```

---

## 2. Categories of Triggers

| Category | Fires On | Typical Use |
|---|---|---|
| **DML Trigger** | `INSERT`, `UPDATE`, `DELETE` on a table/view | Auditing, enforcing complex business rules, cascading logic, maintaining denormalized data |
| **DDL Trigger** | `CREATE`, `ALTER`, `DROP` and other schema-level statements | Preventing schema changes, auditing structural changes, enforcing naming/governance policies |
| **Logon Trigger** | `LOGON` event (server-level) | Restricting logins, auditing connections, enforcing connection limits per user |

---

## 3. DML Triggers: AFTER vs INSTEAD OF

### AFTER Triggers (a.k.a. FOR triggers)
Fire **after** the triggering action has already happened (data is already modified, but still inside the same transaction).

```sql
CREATE TRIGGER trg_Name ON dbo.Table AFTER INSERT, UPDATE, DELETE AS ...
```

- Can only be defined on **tables**, not views.
- Multiple `AFTER` triggers of the same type can exist on the same table (order controlled via `sp_settriggerorder`, though it's best to avoid relying on this).
- The action already succeeded before the trigger runs; if the trigger raises an error or issues `ROLLBACK`, the entire transaction (including the original DML) is rolled back.

### INSTEAD OF Triggers
Fire **in place of** the triggering action — the original `INSERT`/`UPDATE`/`DELETE` does **not** happen automatically; the trigger's code entirely defines what actually occurs.

```sql
CREATE TRIGGER trg_Name ON dbo.Table INSTEAD OF INSERT AS ...
```

- Can be defined on **tables or views** — this is the primary mechanism for making complex (multi-table, aggregate) **views updatable** (see the earlier Views guide).
- Only **one** `INSTEAD OF` trigger per action type (INSERT/UPDATE/DELETE) is allowed per table/view.
- You must manually perform the actual data modification inside the trigger body if you want the effect to happen — otherwise nothing is written.

```sql
CREATE TRIGGER trg_vw_ActiveCustomers_Insert
ON dbo.vw_ActiveCustomers
INSTEAD OF INSERT
AS
BEGIN
    INSERT INTO dbo.Customers (FirstName, LastName, IsActive)
    SELECT FirstName, LastName, 1 FROM inserted;
END;
```

---

## 4. The `inserted` and `deleted` Virtual Tables

Every DML trigger has access to two special, automatically-populated in-memory tables that mirror the affected rows' shape:

| Trigger Type | `inserted` | `deleted` |
|---|---|---|
| `INSERT` | New rows being added | (empty) |
| `DELETE` | (empty) | Rows being removed |
| `UPDATE` | New (post-update) values | Old (pre-update) values |

```sql
CREATE TRIGGER trg_Products_AuditPriceChange
ON dbo.Products
AFTER UPDATE
AS
BEGIN
    SET NOCOUNT ON;
    IF UPDATE(Price)   -- only act if the Price column specifically changed
    BEGIN
        INSERT INTO dbo.PriceChangeLog (ProductID, OldPrice, NewPrice, ChangedAt)
        SELECT d.ProductID, d.Price, i.Price, GETDATE()
        FROM inserted i
        JOIN deleted d ON i.ProductID = d.ProductID
        WHERE i.Price <> d.Price;
    END
END;
```

- `UPDATE(ColumnName)` — boolean check for whether a specific column was targeted by the UPDATE statement (part of the SET clause), regardless of whether the value actually changed.
- `COLUMNS_UPDATED()` — bitmask of all updated columns, for more advanced checks.
- **Triggers fire once per statement, not once per row** — a single `UPDATE` affecting 10,000 rows fires the trigger **once**, with `inserted`/`deleted` containing all 10,000 rows. Always write **set-based** logic (joins against `inserted`/`deleted`), never assume single-row semantics — this is the most common trigger bug for developers coming from row-by-row procedural mindsets.

---

## 5. Multiple Triggers & Execution Order

```sql
EXEC sp_settriggerorder
    @triggername = 'dbo.trg_Orders_First',
    @order = 'First',
    @stmttype = 'INSERT';
```

- You can mark one trigger as `First` and one as `Last` for a given event; any others fire in an undefined order in between.
- Best practice: avoid depending on multiple AFTER triggers per table/event at all — consolidate logic into a single trigger where possible for predictability.

---

## 6. Nested and Recursive Triggers

- **Nested triggers** — a trigger's own DML action can fire another trigger on a different (or the same) table. Controlled server-wide by the `nested triggers` server configuration option (enabled by default).
- **Recursive triggers** — a trigger causes a DML statement that fires itself again (directly, or indirectly through a nested trigger chain that loops back). Controlled at the database level via `ALTER DATABASE ... SET RECURSIVE_TRIGGERS ON/OFF` (off by default).
- Deep nesting (default limit: 32 levels) can cause performance issues or infinite loops if not carefully designed — usually best avoided by keeping trigger logic simple and using guard conditions.

---

## 7. DDL Triggers

Fire in response to schema-level (Data Definition Language) events — `CREATE_TABLE`, `ALTER_TABLE`, `DROP_TABLE`, `CREATE_PROCEDURE`, etc. — at either **database** or **server** scope.

```sql
CREATE TRIGGER trg_PreventTableDrop
ON DATABASE
FOR DROP_TABLE
AS
BEGIN
    PRINT 'Dropping tables is not allowed in this database.';
    ROLLBACK;
END;
```

```sql
CREATE TRIGGER trg_AuditLoginChanges
ON ALL SERVER
FOR CREATE_LOGIN, ALTER_LOGIN, DROP_LOGIN
AS
BEGIN
    INSERT INTO AdminDB.dbo.ServerAudit (EventType, EventData, EventTime)
    VALUES (EVENTDATA().value('(/EVENT_INSTANCE/EventType)[1]', 'NVARCHAR(100)'),
            CAST(EVENTDATA() AS NVARCHAR(MAX)), GETDATE());
END;
```

- `EVENTDATA()` returns an XML document describing the event (event type, object name, T-SQL command text, login, timestamp, etc.) — the primary way to introspect what actually happened.
- Common uses: **governance/compliance** (block risky DDL like `DROP TABLE` in production, prevent unauthorized schema drift), **auditing** structural changes.
- Scopes: `ON DATABASE` (fires for events within that DB) vs. `ON ALL SERVER` (fires server-wide, including login/server-level events).

---

## 8. Logon Triggers

Fire after a login authenticates but **before** the session is fully established — useful for connection governance.

```sql
CREATE TRIGGER trg_LimitConnections
ON ALL SERVER
FOR LOGON
AS
BEGIN
    IF ORIGINAL_LOGIN() = 'RestrictedUser'
       AND (SELECT COUNT(*) FROM sys.dm_exec_sessions WHERE login_name = 'RestrictedUser') > 3
        ROLLBACK;  -- rejects the connection attempt
END;
```

- Can reject a connection entirely (`ROLLBACK`) — e.g., enforce max concurrent sessions per login, restrict logon by time of day, or block logins from unexpected applications.
- Runs in a special limited context — be careful; a buggy logon trigger can lock out all logins, including sysadmin (mitigated by connecting with the **Dedicated Admin Connection (DAC)**, which bypasses logon triggers).

---

## 9. Common Use Cases

1. **Auditing** — track who changed what and when (often into a separate audit table, or via `EVENTDATA()` for schema changes).
2. **Enforcing complex business rules** that can't be expressed as a simple `CHECK` constraint (e.g., cross-table validation: "an order's total can't exceed the customer's credit limit").
3. **Maintaining denormalized/summary data** — e.g., updating a `Customers.TotalOrders` counter column whenever a row is inserted into `Orders`.
4. **Making complex views updatable** via `INSTEAD OF` triggers.
5. **Preventing unwanted schema drift** in production via DDL triggers.
6. **Replicating/synchronizing** changes to another table or system (though modern designs often prefer Change Data Capture (CDC) or Change Tracking for this instead — see below).

---

## 10. Downsides & Why Triggers Are Often Discouraged

1. **Hidden logic / "spooky action at a distance"** — a plain `INSERT` statement can silently cascade into a dozen side effects that aren't visible at the call site, making debugging and onboarding much harder.
2. **Performance overhead** — every affected DML statement now carries the extra cost of running the trigger's logic, inside the same transaction (extends lock duration).
3. **Hard-to-predict execution order** with multiple triggers, and risk of runaway recursion/nesting if not carefully guarded.
4. **Set-based logic requirement** — trigger bugs are extremely common when developers write row-by-row assumptions instead of correctly joining against `inserted`/`deleted`.
5. **Testing complexity** — triggers add implicit coupling that unit/integration tests must account for; ORMs like EF Core generally have **no visibility** into trigger side effects (see below).
6. Many teams prefer to push this kind of logic into the **application layer** or into an explicit, callable **stored procedure**, reserving triggers mainly for auditing or cases where you must guarantee behavior *regardless* of which client/tool performs the write (since triggers fire no matter what issued the DML — app code, ad-hoc query, bulk import, etc.).

**Modern alternatives to consider:**
- **Change Data Capture (CDC)** / **Change Tracking** — built-in, lower-overhead mechanisms for tracking what changed, better suited to ETL/auditing pipelines than triggers.
- **Computed columns** or **CHECK constraints** for simple derived-value/validation logic instead of a trigger.
- **Application-level logic / domain events** for business rules that are easier to test and reason about in code.

---

## 11. Trigger Management Commands

```sql
-- Disable / Enable
DISABLE TRIGGER trg_Name ON dbo.Orders;
ENABLE TRIGGER trg_Name ON dbo.Orders;

DISABLE TRIGGER ALL ON dbo.Orders;   -- disable all triggers on a table

-- Drop
DROP TRIGGER trg_Name;               -- DML trigger
DROP TRIGGER trg_Name ON DATABASE;   -- DDL trigger (database scope)
DROP TRIGGER trg_Name ON ALL SERVER; -- DDL/Logon trigger (server scope)

-- List triggers
SELECT name, is_disabled, is_instead_of_trigger
FROM sys.triggers
WHERE parent_id = OBJECT_ID('dbo.Orders');

SELECT * FROM sys.server_triggers;      -- server-scoped
SELECT * FROM sys.triggers WHERE parent_class = 0;  -- database-scoped DDL triggers

-- View definition
EXEC sp_helptext 'trg_Name';
```

---

## 12. Relation to EF Core

EF Core's change tracker has **no awareness** of what a trigger does — it only tracks the entity state changes it explicitly issued via its own generated `INSERT`/`UPDATE`/`DELETE` statements. If a trigger modifies additional columns or other tables as a side effect, EF Core won't automatically reflect that in your in-memory entity graph.

### Key issue: `OUTPUT` clause conflicts

By default, EF Core uses SQL Server's `OUTPUT INSERTED.*` clause to efficiently retrieve generated values (like an `IDENTITY` column) right after an `INSERT`, in the same round-trip:

```sql
INSERT INTO dbo.Products (Name, Price)
OUTPUT INSERTED.ProductID
VALUES (@p0, @p1);
```

**This fails** if the target table has an `AFTER INSERT`/`AFTER UPDATE`/`AFTER DELETE` trigger, because SQL Server disallows `OUTPUT ... INTO` combined with triggers on the same table in certain forms — EF Core historically threw errors like:
```
The target table 'X' of the INSERT statement cannot have any enabled triggers if the statement contains an OUTPUT clause without INTO clause.
```

**Fix (EF Core 7+):** you must explicitly tell EF Core the table has triggers, so it switches to a different strategy (e.g., re-querying instead of `OUTPUT`):

```csharp
modelBuilder.Entity<Product>()
    .ToTable("Products", tb => tb.HasTrigger("trg_Products_AfterInsert"));
```

Without this configuration, `SaveChanges()` can throw at runtime on tables that have triggers — this is one of the most common EF Core + SQL Server trigger gotchas in practice.

### Other implications

- Because triggers can modify rows beyond what EF Core's statement explicitly targeted, **you should re-query after `SaveChanges()`** if you need up-to-date values that a trigger might have changed (e.g., a computed audit column, a cascading update to another entity).
- If a trigger issues a `ROLLBACK` (business rule violation), EF Core will surface this as a `DbUpdateException` wrapping the SQL Server error — handle it like any other constraint violation.
- Bulk operations (`ExecuteUpdateAsync`/`ExecuteDeleteAsync` in EF Core 7+, which bypass change tracking and run direct set-based SQL) still fire triggers normally, since triggers operate at the SQL engine level regardless of what client issued the statement.
- If your team relies heavily on triggers for business logic that EF Core's app-side code also assumes, consider whether that logic would be more transparent, testable, and EF-Core-friendly as application code or an explicitly-called stored procedure instead.

---

## 13. Summary

| Trigger Type | Scope | Fires | Common Use |
|---|---|---|---|
| `AFTER` (DML) | Table | After INSERT/UPDATE/DELETE succeeds | Auditing, cascading updates, business rule enforcement |
| `INSTEAD OF` (DML) | Table or View | Replaces the DML action entirely | Making complex views updatable, custom validation before write |
| DDL Trigger | Database or Server | Schema-changing statements (CREATE/ALTER/DROP) | Governance, blocking risky schema changes, structural audit |
| Logon Trigger | Server | Login/connection attempt | Connection limits, login restrictions, connection auditing |

Key mental model: triggers are **implicit, automatic, and statement-scoped** (not row-scoped) — always write set-based logic against `inserted`/`deleted`, and remember that EF Core needs to be explicitly told a table has triggers (`HasTrigger`) or `SaveChanges()` can break.
