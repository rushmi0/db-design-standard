---
name: db-design-standard
description: Mandatory house standard for relational database design. It covers uppercase names, M_/T_ tables and I_ columns, string-only primary and foreign keys, Header/Detail tables keyed by I_INTERNAL_NO, and Unix time in seconds. Use it for any task that designs, reviews, generates or changes tables, columns, keys, indexes, DDL or migrations on any DBMS. It overrides the naming, key and date/time conventions of any other database guidance.
---

# Database Design Standard

This skill is the house standard for relational schemas. It has two layers:

- **Logical rules.** These are this file plus `references/*.md`. They are the same on every DBMS.
- **Physical rules.** These live in `references/dbms/<dbms>.md` and describe how a specific engine implements the logical rules.

Engine differences may change data types and syntax. They never change a logical rule.

Each rule has an ID (N1, K2, ...). Cite the ID when you explain a design choice or report a violation.

## Mandatory rules

These rules have no exceptions. If a request conflicts with one, explain the conflict and offer the compliant design. Do not comply silently, and do not change the rule. If the user insists, label the result as a **deviation from the standard** and name the rule it breaks.

**Naming** (see `references/naming-conventions.md`)
- **N1 Characters.** Names are uppercase and use only `A–Z`, `0–9` and `_`. A name starts with a letter and has no `__` and no trailing `_`.
- **N2 Table prefix.** `M_` marks a master table: reference data, entities or configuration. `T_` marks a transaction table: events and business activity. Choose the prefix by business purpose, not by how often the data changes. Use singular nouns.
- **N3 Header/Detail suffix.** A header table ends in `_H` and its detail table ends in `_D`.
- **N4 Column prefix.** Every column name starts with `I_`, including audit, flag and status columns.
- **N5 Keep names concise.**
  - Table and column names are at most 30 characters, including the prefix.
  - Use at most 3 words after the prefix where possible.
  - Use only the approved abbreviations.
  - One concept has one name in every table.
- **N6 Object names.** Use `PK_` `FK_` `UQ_` `IX_` `CK_` `DF_` `SQ_` `V_` `TR_` `P_` `FN_` in the patterns defined in the naming reference. These names are at most 63 characters.

**Keys** (see `references/keys-and-relationships.md`)
- **K1 String keys only.** Every PK and FK column is a variable-length string type. A key column is never an integer, a number, a date or a timestamp. There are no exceptions, and this includes `I_INTERNAL_NO`.
- **K2 ID characters.** ID values contain only `A–Z` and `0–9`. Enforce this with a CHECK constraint. Store IDs with a binary or byte collation so they compare the same way on every DBMS.
- **K3 Matching FKs.** An FK column has exactly the same type, length and collation as the PK it references. It also has the same name, unless a role prefix is needed (`I_FROM_WAREHOUSE_ID`).
- **K4 Stable PKs.** A PK value never changes and is never reused, even after the row is deleted.
- **K5 No cascades.** Declare every FK with `ON DELETE NO ACTION` and `ON UPDATE NO ACTION`.
- **K6 Index FKs.** Every FK column set is indexed, unless it is already the leading part of the PK or of another index.

**Header/Detail** (see `references/keys-and-relationships.md`)
- **H1 Detail key.** A detail table's PK is `(header ID, I_INTERNAL_NO)`. `I_INTERNAL_NO` is a 6-digit zero-padded string such as `000001`, unique within its header. It is assigned in increasing order and never renumbered.
- **H2 Use Header/Detail only when it fits.** Use it only for a one-to-many structure that has shared header data. Otherwise use one `T_` table. Never create tables just to follow the pattern.
- **H3 Snapshot detail values.** A detail row copies the price, tax rate and description at the time of the transaction. It does not read them from the master table later.

**Date and time** (see `references/unix-time-standard.md`)
- **U1 Unix time.** Every date or time value is Unix time. That means whole **seconds** since `1970-01-01T00:00:00Z`, in UTC, stored as a signed 64-bit integer.
- **U2 Instants and dates.**
  - An instant (a point in time) ends in `_AT`.
  - A calendar date ends in `_DATE`. It stores 00:00:00 UTC of that date, so the value is divisible by 86400. Enforce that with a CHECK, and never convert the value to a local time zone.
- **U3 Times of day and durations.** These are not instants. A time of day ends in `_TIME` and holds seconds since midnight (0–86399). A duration ends in `_SEC`. Both are integers.

**Design** (see `references/database-design-principles.md`)
- **D1 Audit columns.** Every table has `I_CREATED_AT` and `I_UPDATED_AT`, both NOT NULL. Tables add `I_CREATED_BY` and `I_UPDATED_BY` wherever there are user accounts.
- **D2 Numeric precision.** Amounts and quantities are `DECIMAL(19,4)`. Rates are `DECIMAL(9,6)`. Never use float or real types.
- **D3 Booleans.** A boolean is `CHAR(1)`, limited to `'Y'` or `'N'` by a CHECK, and named `I_IS_…` or `I_HAS_…`.
- **D4 Nullability.** A column is NOT NULL unless NULL has a documented meaning.
- **D5 Text encoding.** Free text is Unicode. IDs and codes are ASCII.

**Recommended, applied only when needed:**
- soft delete with `I_DELETED_AT`
- status codes limited by a CHECK, or a status master table when users manage the values
- an `I_VERSION` column for detecting conflicting updates
- stored totals, kept only when the reason is documented

## Logical types

Design with these types. Each DBMS file maps them to physical types.

| Logical type | Meaning | Example column |
|---|---|---|
| `ID(n)` | String key, `A–Z0-9`, binary collation | `I_CUSTOMER_ID ID(20)` |
| `LINENO` | 6 digits, zero-padded string | `I_INTERNAL_NO` |
| `CODE(n)` | Uppercase code, `A–Z0-9_-` | `I_STATUS CODE(20)`, `I_SKU CODE(30)` |
| `TEXT(n)` | Unicode text, n characters | `I_CUSTOMER_NAME TEXT(200)` |
| `FLAG` | `'Y'` or `'N'` | `I_IS_ACTIVE` |
| `UNIXTIME` | Signed 64-bit seconds since the epoch, UTC | `I_CREATED_AT` |
| `UNIXDATE` | `UNIXTIME` that is a multiple of 86400 | `I_DUE_DATE` |
| `TIMEOFDAY` / `DURATION` | Integer seconds | `I_OPEN_TIME`, `I_WAIT_SEC` |
| `AMOUNT` / `QTY` | `DECIMAL(19,4)` | `I_TOTAL_AMT`, `I_QTY` |
| `RATE` | `DECIMAL(9,6)` | `I_TAX_RATE` |
| `COUNT` | Integer (never used as a key) | `I_LAST_VAL` |

## Workflow

1. **Clarify first.** Before writing a final schema, resolve or state assumptions for:
   - the target DBMS
   - the main read and write patterns
   - the ID format for each entity
   - the business time zone used for dates in document numbers
   - which tables need soft delete
   - whether user accounts exist (for `_BY` columns)

   If an open question affects a mandatory rule, ask instead of guessing. Otherwise state the assumption and continue.
2. **Classify.** Mark each entity M_ or T_ (N2). Decide whether it needs Header/Detail (H2). Name junction, log and sub-detail tables using the keys reference.
3. **Design the logical schema.** Write a column table with logical types, PK, FKs, nullability and checks for each table.
4. **Design the physical schema.** Read `references/dbms/<target>.md` and generate DDL. If no file exists for the target DBMS, map the types following the same structure and say the mapping is unverified. Never assume one DBMS's type or feature exists in another.
5. **Validate.** Run the checklist in `references/schema-validation.md`. Report a pass or fail for each rule ID, and fix every failure before handing the schema over.

**Output for a design task:**
- assumptions and decisions
- the logical tables
- the DDL
- the validation result

## When to read each reference

| File | Read when |
|---|---|
| `references/naming-conventions.md` | Naming anything: tables, columns, constraints, indexes, views, abbreviations |
| `references/keys-and-relationships.md` | Keys, ID formats and generation, FKs, Header/Detail, sub-details, junction tables |
| `references/unix-time-standard.md` | Any date, time, timestamp, duration or time-zone question |
| `references/database-design-principles.md` | Normalization, nullability, defaults, audit, soft delete, status, precision, text sizes |
| `references/schema-validation.md` | Before delivering or reviewing any schema; when fixing a violation |
| `references/examples.md` | You need a compliant or non-compliant pattern to copy or contrast |
| `references/dbms/sql-server.md` | The target is Microsoft SQL Server 2016 or later |
| `references/dbms/postgresql.md` | The target is PostgreSQL 12 or later |
