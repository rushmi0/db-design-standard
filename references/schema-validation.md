# Schema Validation

Run this before you deliver a schema, review someone else's schema, or approve a migration. Report the result for each rule ID. A schema is compliant only when every mandatory rule passes, or when a failure is an explicitly approved deviation that names its rule.

## 1. How to validate

1. **Automated checks.** If the schema exists in a database, run the catalog audit query in `dbms/<vendor>.md`, section 9. Every row it returns is a violation. If the schema exists only as DDL, check the same rules by reading the DDL.
2. **Manual checks.** Go through the checklist below for the rules the catalog cannot see. Those are marked *manual*.
3. **Every target vendor.** When a schema targets several vendors, validate the DDL for each one.
4. **Fix every failure** using section 3, then repeat steps 1–3.
5. **Report** in the format in section 4.

## 2. Checklist

### Project settings
- [ ] *Manual:* the name case (lowercase by default) and the target vendors are stated, and every vendor has a file in `dbms/` or is marked as an unverified mapping.

### Naming
- [ ] **N1** Every name uses only the project case, `0-9` and `_`, starts with a letter, and has no spaces, `__` or trailing `_`. One case across the whole database.
- [ ] **N2** Every table starts with `m_` or `t_`. *Manual:* the class matches the table's business purpose, using the classification guide in the naming reference.
- [ ] **N3** *Manual:* only header tables end in `_h` and only detail tables end in `_d`; no other structural suffix is used.
- [ ] **N4** Every column starts with `i_`.
- [ ] **N5** Table and column names are ≤ 30 characters. *Manual:* only approved abbreviations; the same concept has the same name in every table; the table name isn't repeated in non-key columns without reason.
- [ ] **N6** Constraint and index names follow `pk_<table>`, `fk_<table>_…`, `uq_`, `ix_`, `ck_`, `df_`, and are ≤ 63 characters. (MySQL/MariaDB always name the PK `PRIMARY`; SQLite doesn't expose constraint names — check those in the DDL.)
- [ ] **N7** *Manual:* no name is a reserved word on a target vendor, and quoting follows the naming reference §5 (PostgreSQL quotes everything in an uppercase project; Oracle never quotes).

### Logical types
- [ ] **L1** *Manual:* the logical schema uses only `string(n)`, `string(n) ascii`, `int`, `long`, `boolean`, `decimal(p,s)` and `timestamp`, and the DDL uses the mapping in `logical-types.md`. SQLite: every table is `STRICT` (automated).
- [ ] **L2** *Manual:* every human-text column is the vendor's Unicode type (`NVARCHAR`, `utf8mb4`, `VARCHAR2 … CHAR` in `AL32UTF8`, UTF8 database). Only IDs, codes, statuses and the technical audit columns are ASCII. A Thai, Japanese and Chinese sample survives a write and read through the real driver.

### Keys
- [ ] **K1** Every PK and FK column is the vendor's ASCII string type. There are no `IDENTITY`/`SERIAL`/`AUTO_INCREMENT` columns and no time values in keys.
- [ ] **K2** Key columns use the binary collation (or BYTE semantics on Oracle). *Manual:* every PK column has a CHECK for its exact format, using lowercase `a–z` and `0–9` only.
- [ ] **K3** Every FK column has the same type, length and collation as its PK. *Manual:* the FK column has the same name as the PK, or a role word plus that name.
- [ ] **K4** *Manual:* no code path updates a PK value. A business code that can change is in its own `uq_` column.
- [ ] **K5** Every FK uses `NO ACTION` for delete and update (on Oracle: no `ON DELETE` clause).
- [ ] **K6** Every FK column set is the leading part of an index, or of the PK.
- [ ] **K7** *Manual:* each table's ID format (abbreviation, date part, sequence width), its generation method, concurrency handling and overflow behaviour are documented; abbreviations are unique in the database; IDs fit in 20 characters.

### Header/Detail
- [ ] **H1** *Manual:* the detail PK is exactly `(header ID, i_internal_no)`. `i_internal_no` is `string(6) ascii` with a CHECK for 6 digits that is not `000000`.
- [ ] **H2** *Manual:* each Header/Detail pair has a real one-to-many relationship with shared data. Every standalone event is a single table.
- [ ] **H3** *Manual:* detail tables store snapshots (description, price, tax) of values that can change in masters.

### Date and time
- [ ] **U1** There are no vendor date/time columns; every `_at`/`_date` column is the 64-bit integer type. *Manual:* values are in seconds, not milliseconds — present-day examples have 10 digits.
- [ ] **U2** *Manual:* every `_date` column has a CHECK `% 86400 = 0` (`MOD` on Oracle); instants end in `_at` and dates end in `_date`.
- [ ] **U3** *Manual:* `_time` columns have a CHECK `BETWEEN 0 AND 86399`; `_sec` columns have a CHECK `>= 0`.

### Design
- [ ] **D1** All six audit columns are on every table, NOT NULL, with the right types. *Manual:* the data-access layer fills them on every insert and update.
- [ ] **D2** `i_active` is on every table, boolean, NOT NULL, default true. *Manual:* no physical deletes in normal operation; standard queries, repositories and APIs return active rows only unless inactive rows are requested; the uniqueness choice (all time vs active only) is documented per business code.
- [ ] **D3** Every `i_is_…`/`i_has_…` column is boolean and NOT NULL (with a CHECK `IN (0,1)` where the vendor's boolean is a number).
- [ ] **D4** No float, real, double or money types. *Manual:* amounts and quantities are `decimal(19,4)`, rates `decimal(9,6)` (scaled integers on SQLite).
- [ ] **D5** *Manual:* every nullable column has a documented meaning for NULL, and no design depends on storing an empty string.

### Recommended items, if used
- [ ] Status columns have a CHECK list or a status master, and the allowed transitions are documented.
- [ ] Every stored total and other denormalized value is documented with the rule that keeps it correct.

## 3. Fixing violations

| Violation | Fix |
|---|---|
| Name in the wrong case, or mixed case | Rename to the project case. On PostgreSQL in an uppercase project, quote it everywhere; on Oracle, never quote |
| Missing `m_`/`t_`, or wrong class | Choose by business purpose (naming reference §2) and rename |
| Table ends in `_h`/`_d` but isn't a header/detail, or another suffix is used as a structure marker | Rename |
| Column without `i_` | Rename with `i_` and the correct type suffix |
| Name too long | Use approved abbreviations or a shorter word with the same meaning. Never invent abbreviations |
| Vendor type in the logical schema | Replace with the logical type, then map it with `logical-types.md` |
| Human text in a non-Unicode type (`VARCHAR` on SQL Server, `utf8mb3`, `VARCHAR2 BYTE`) | Change to the vendor's Unicode type; check that the connection is Unicode too |
| Integer or IDENTITY PK | Replace with a string ID in the `<abbr><date><sequence>` format, generated from `m_id_seq` or a sequence. Migrate FKs to the same string type |
| ID with uppercase letters or other characters | Convert values to lowercase `a–z0–9` in a migration and add the PK pattern CHECK |
| Single surrogate PK on a detail table | Replace with `(header ID, i_internal_no)` and drop the surrogate |
| Integer `i_internal_no` | Convert to `string(6) ascii`, zero-padded |
| FK type, length or collation mismatch | Copy the PK column definition exactly |
| `CASCADE` / `SET NULL` | Change to `NO ACTION`. Remove through `i_active` or status |
| FK without an index | Add `ix_<table>_<cols>` (on MySQL, inside `CREATE TABLE` or before the FK) |
| Date/time type, string date or milliseconds | Change to the 64-bit integer type holding seconds. Convert existing data in UTC (`_date`: UTC midnight); divide millisecond values by 1000 |
| Time value in a PK | Use a string PK plus a `uq_` that includes the time column |
| Missing audit columns | Add all six as NOT NULL; backfill existing rows (`created`/`updated` = migration time, `func_nm` = the migration name, `by` = `migration`) |
| Missing `i_active` | Add it NOT NULL with default true; change every read path to filter it |
| Physical `DELETE` in application code | Replace with an update that sets `i_active` to false and the `updated` audit columns |
| Float or money type | Change to `decimal(19,4)` or `decimal(9,6)`; on SQLite, a scaled integer |
| `CHAR(1) 'Y'/'N'`, nullable or three-state flag | Change to the vendor's boolean mapping, NOT NULL. A real third state belongs in `i_status` |
| Empty strings stored | Convert to NULL and make the column nullable, with a documented meaning |
| Header/Detail with one detail per header | Merge into one `t_` table |
| Detail reads current master prices | Add snapshot columns and fill them at transaction time |

Changing a key, type or date/time representation in a live database is a migration. Expand first: add the new columns. Backfill in batches. Switch the application over. Then remove the old columns. Never do it in one locking step.

## 4. Report format

```
Schema: <name>   Vendor(s): <list>   Case: lowercase   Result: PASS | FAIL (<n> violations)

Rule  Object                      Problem                         Fix
K1    t_order_d.id                INT IDENTITY PK                 PK (i_order_id, i_internal_no)
U1    t_order_h.i_order_date      DATETIME                        BIGINT, Unix seconds
D2    m_customer                  missing i_active                add i_active boolean NOT NULL DEFAULT true
...
Deviations approved: <rule — reason — approved by>, or "none"
Assumptions: <list>
```
