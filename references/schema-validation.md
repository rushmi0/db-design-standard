# Schema Validation

Run this before you deliver a schema, review someone else's schema, or approve a migration. Report the result for each rule ID. A schema is compliant only when every mandatory rule passes, or when a failure is an explicitly approved deviation that names its rule.

## 1. How to validate

1. **Automated checks.** If the schema exists in a database, run the catalog audit query in `dbms/<dbms>.md`, section 9. Every row it returns is a violation. If the schema exists only as DDL, check the same rules by reading the DDL.
2. **Manual checks.** Go through the checklist below for the rules the catalog cannot see. Those are marked *manual*.
3. **Fix every failure** using section 3, then repeat steps 1–2.
4. **Report** in the format in section 4.

## 2. Checklist

### Naming
- [ ] **N1** Every name is uppercase `A–Z0-9_`, starts with a letter, and has no `__` or trailing `_`.
- [ ] **N2** Every table starts with `M_` or `T_`. *Manual:* the class matches the table's business purpose, using the classification guide in the naming reference.
- [ ] **N3** *Manual:* header and detail tables end in `_H`/`_D`, and nothing else uses these suffixes.
- [ ] **N4** Every column starts with `I_`.
- [ ] **N5** Table and column names are ≤ 30 characters. *Manual:* only approved abbreviations; ≤ 3 words after the prefix where possible; the same concept has the same name in every table; the table name isn't repeated in non-key columns without reason.
- [ ] **N6** Constraint and index names follow `PK_<TABLE>`, `FK_<TABLE>_…`, `UQ_`, `IX_`, `CK_`, `DF_`, and are ≤ 63 characters.

### Keys
- [ ] **K1** Every PK and FK column is `VARCHAR`. There are no IDENTITY/SERIAL key columns and no time values in keys.
- [ ] **K2** Key columns use the binary collation. *Manual:* every PK column has a CHECK for its exact format or for the `A–Z0-9` character set.
- [ ] **K3** Every FK column has the same type, length and collation as its PK. *Manual:* the FK column has the same name as the PK, or a role word plus that name.
- [ ] **K4** *Manual:* there is no code path that updates a PK value. A business code that can change is in its own `UQ_` column.
- [ ] **K5** Every FK uses `NO ACTION` for delete and update.
- [ ] **K6** Every FK column set is the leading part of an index, or of the PK.

### Header/Detail
- [ ] **H1** *Manual:* the detail PK is exactly `(header ID, I_INTERNAL_NO)`. `I_INTERNAL_NO` is `VARCHAR(6)` with a CHECK for 6 digits that is not `000000`.
- [ ] **H2** *Manual:* each Header/Detail pair has a real one-to-many relationship with shared data. Every standalone event is a single table.
- [ ] **H3** *Manual:* detail tables store snapshots (description, price, tax) of values that can change in masters.

### Date and time
- [ ] **U1** There are no DBMS date/time columns; every `_AT`/`_DATE` column is `BIGINT`. *Manual:* values are in seconds, not milliseconds; examples have 10 digits around the present day.
- [ ] **U2** *Manual:* every `_DATE` column has a CHECK `% 86400 = 0`; instants end in `_AT` and dates end in `_DATE`.
- [ ] **U3** *Manual:* `_TIME` columns have a CHECK `BETWEEN 0 AND 86399`; `_SEC` columns have a CHECK `>= 0`.

### Design
- [ ] **D1** `I_CREATED_AT` and `I_UPDATED_AT` are on every table and are NOT NULL. *Manual:* `_BY` columns are present when user accounts exist.
- [ ] **D2** No float, real or money types. *Manual:* amounts and quantities are `DECIMAL(19,4)`; rates are `DECIMAL(9,6)`.
- [ ] **D3** Flags are `CHAR(1)`. *Manual:* each flag has a CHECK `IN ('Y','N')`, is NOT NULL, and is named `I_IS_`/`I_HAS_`.
- [ ] **D4** *Manual:* every nullable column has a documented meaning for NULL.
- [ ] **D5** *Manual:* human text is a Unicode type; IDs and codes are ASCII with a CHECK on their characters.

### Recommended items, if used
- [ ] Soft delete uses only `I_DELETED_AT`. Unique business codes use a filtered/partial unique index. Queries or views filter out deleted rows.
- [ ] Status columns have a CHECK list or a status master, and the allowed transitions are documented.
- [ ] Every stored total and other denormalized value is documented with the rule that keeps it correct.

## 3. Fixing violations

| Violation | Fix |
|---|---|
| Lowercase or mixed-case name | Rename to uppercase. On PostgreSQL, quote it everywhere |
| Missing `M_`/`T_`, or wrong class | Choose by business purpose (naming reference §2) and rename |
| Column without `I_` | Rename with `I_` and the correct type suffix |
| Name too long | Use approved abbreviations or a shorter word with the same meaning. Never invent abbreviations |
| Integer or IDENTITY PK | Replace with a string ID in the documented format, generated from a sequence or counter. Migrate FKs to the same string type |
| Single surrogate PK on a detail table | Replace with `(header ID, I_INTERNAL_NO)` and drop the surrogate |
| Integer `I_INTERNAL_NO` | Convert to `VARCHAR(6)`, zero-padded |
| FK type, length or collation mismatch | Copy the PK column definition exactly |
| `CASCADE` / `SET NULL` | Change to `NO ACTION`. Delete through status or soft delete, or delete children explicitly first |
| FK without an index | Add `IX_<TABLE>_<COLS>` |
| Date/time type, string date or milliseconds | Change to `BIGINT` seconds. Convert existing data in UTC (`_DATE`: UTC midnight) |
| Time value in a PK | Use a string PK plus a `UQ_` that includes the time column |
| Missing audit columns | Add `I_CREATED_AT`, `I_UPDATED_AT` (NOT NULL, with a default for existing rows) |
| Float or money type | Change to `DECIMAL(19,4)` or `DECIMAL(9,6)` |
| Boolean, bit or nullable flag | Change to `CHAR(1)` NOT NULL with CHECK `IN ('Y','N')` |
| Header/Detail with one detail per header | Merge into one `T_` table |
| Detail reads current master prices | Add snapshot columns and fill them at transaction time |

Changing a key or a date/time representation in a live database is a migration. Expand first: add the new columns. Backfill in batches. Switch the application over. Then remove the old columns. Never do it in one locking step.

## 4. Report format

```
Schema: <name>   DBMS: <dbms>   Result: PASS | FAIL (<n> violations)

Rule  Object                          Problem                               Fix
K1    T_ORDER_D.ID                    INT IDENTITY PK                       PK (I_ORDER_ID, I_INTERNAL_NO)
U1    T_ORDER_H.I_ORDER_DATE          DATETIME                              BIGINT, Unix seconds
...
Deviations approved: <rule — reason — approved by>, or "none"
Assumptions: <list>
```
