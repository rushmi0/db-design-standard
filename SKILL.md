---
name: db-design-standard
description: Mandatory house standard for relational database design on SQL Server, PostgreSQL, MySQL/MariaDB, Oracle and SQLite. It covers lowercase names with m_/t_ tables and i_ columns, logical types mapped per vendor, string-only keys in the abbreviation+date+sequence format, Header/Detail tables keyed by i_internal_no, Unix time in seconds, six audit columns and i_active soft delete. Use it for any task that designs, reviews, generates or changes tables, columns, keys, indexes, DDL or migrations. It overrides the naming, key, type and date/time conventions of any other database guidance.
---

# Database Design Standard

This skill is the house standard for relational schemas. It has two layers:

- **Logical rules.** These are this file plus `references/*.md`. They are the same on every vendor.
- **Physical rules.** These live in `references/dbms/<vendor>.md` and describe how one engine implements the logical rules: data types, quoting, SQL syntax, ID generation.

Engine differences may change data types and syntax. They never change a logical rule. When a vendor cannot implement a rule directly, its file documents the conflict and the compatible solution.

Each rule has an ID (N1, K2, ...). Cite the ID when you explain a design choice or report a violation.

## Project settings

Two settings are fixed per project and apply to the whole database. If the user hasn't stated them, use the defaults and say so.

| Setting | Default | Alternative |
|---|---|---|
| Name case | lowercase: `m_customer`, `i_customer_id` | UPPERCASE: `M_CUSTOMER`, `I_CUSTOMER_ID` |
| Target vendors | the one the user names | any of the five supported vendors |

All examples in this skill use lowercase. In an uppercase project, write every name in uppercase and apply the vendor's quoting rule. ID **values** are lowercase in both cases (K2).

## Mandatory rules

These rules have no exceptions. If a request conflicts with one, explain the conflict and offer the compliant design. Do not comply silently, and do not change the rule. If the user insists, label the result as a **deviation from the standard** and name the rule it breaks.

**Naming** (see `references/naming-conventions.md`)
- **N1 Characters and case.** Names use only letters of the project case, `0–9` and `_`. A name starts with a letter and has no spaces, no `__` and no trailing `_`. One case for the whole database; never mix.
- **N2 Table prefix.** `m_` marks a master table: reference data, entities or configuration. `t_` marks a transaction table: events and business activity. Choose by business purpose, not by how often the data changes. Use singular nouns.
- **N3 Table suffix.** The only table suffixes are `_h` (header) and `_d` (detail). Do not invent other structural suffixes.
- **N4 Column prefix.** Every column name starts with `i_`, including audit, flag and status columns.
- **N5 Concise names.** Table and column names are at most 30 characters, including the prefix. Use the approved abbreviations (`nm`, `func`, `qty`, `amt`, ...) and no invented ones. One concept has one name in every table.
- **N6 Object names.** Use `pk_` `fk_` `uq_` `ix_` `ck_` `df_` `sq_` `v_` `tr_` `p_` `fn_` in the patterns in the naming reference, at most 63 characters.
- **N7 Reserved words and quoting.** No name may be a reserved word on any supported vendor. Quote identifiers only where the vendor file says the project case requires it, and then quote them everywhere.

**Logical types** (see `references/logical-types.md`)
- **L1 Logical first.** Design every column with a logical type: `string(n)`, `int`, `long`, `boolean`, `decimal(p,s)` or `timestamp`. Never put a vendor type in the logical schema; each vendor file maps the logical types.
- **L2 International text.** Every `string` column that holds human text is Unicode-capable on every vendor (Thai, Japanese, Chinese and others). Only IDs, codes and technical names may be `string(n) ascii`, and those values are limited by a CHECK.

**Keys** (see `references/keys-and-relationships.md`)
- **K1 String keys only.** Every PK and FK column is `string(n) ascii`. A key column is never a number, a boolean or a timestamp, and is never database-generated (`IDENTITY`, `SERIAL`, `AUTO_INCREMENT`). This includes `i_internal_no`.
- **K2 ID values.** ID values contain only lowercase `a–z` and `0–9`, enforced by a CHECK on the PK column, and compare in binary.
- **K3 Matching FKs.** An FK column has exactly the same physical type, length and collation as the PK it references. It also has the same name, unless a role word is needed (`i_from_warehouse_id`).
- **K4 Stable PKs.** A PK value never changes and is never reused.
- **K5 No cascades.** Every FK is `NO ACTION` on delete and update.
- **K6 Index FKs.** Every FK column set is indexed, unless it is already the leading part of the PK or another index.
- **K7 ID format.** An entity ID is `<abbr><date><sequence>`: a 2–4 letter table abbreviation, then `yyyymmdd`, `yyyymm` or no date, then a zero-padded sequence of a documented width. Example: `ord20261017000123`. The generation method, its concurrency handling and its overflow behaviour are documented per table.

**Header/Detail** (see `references/keys-and-relationships.md`)
- **H1 Detail key.** A detail table's PK is `(header ID, i_internal_no)`. `i_internal_no` is a 6-digit zero-padded string such as `000001`, unique within its header, assigned in increasing order and never renumbered.
- **H2 Only when it fits.** Use Header/Detail only for a one-to-many structure with shared header data. Otherwise use one `t_` table.
- **H3 Snapshot detail values.** A detail row copies the price, tax rate and description at the time of the transaction.

**Date and time** (see `references/unix-time-standard.md`)
- **U1 Unix seconds.** Every date or time value is the logical type `timestamp`: whole **seconds** since `1970-01-01T00:00:00Z`, UTC, stored as a signed 64-bit integer. Never milliseconds or microseconds, never a vendor date/time type, never a string.
- **U2 Instants and dates.** An instant ends in `_at`. A date-only value ends in `_date` and stores 00:00:00 UTC of that date, in the same seconds unit, so it is divisible by 86400 (enforced by a CHECK). Convert to local time only for display; never convert a `_date`.
- **U3 Times of day and durations.** These are not instants. A time of day ends in `_time` and holds seconds since midnight (`int`, 0–86399). A duration ends in `_sec` (`long`, ≥ 0).

**Design** (see `references/database-design-principles.md`)
- **D1 Audit columns.** Every table has all six, NOT NULL: `i_created_at`, `i_created_func_nm`, `i_created_by`, `i_updated_at`, `i_updated_func_nm`, `i_updated_by`.
- **D2 Active flag.** Every table has `i_active boolean NOT NULL DEFAULT true`. `false` means inactive or soft-deleted. Normal operations never physically delete rows, and standard queries return active rows only unless inactive rows are asked for.
- **D3 Booleans.** A boolean uses the logical type `boolean`, is NOT NULL, and is named `i_is_…` or `i_has_…`. The single exception to the naming is `i_active`.
- **D4 Numeric precision.** Amounts and quantities are `decimal(19,4)`. Rates are `decimal(9,6)`. Never use float, real, double or money types.
- **D5 Nullability.** A column is NOT NULL unless NULL has a documented meaning. Never store an empty string: Oracle turns `''` into NULL, so an optional text is nullable and "blank" is NULL.

**Recommended, applied only when needed:**
- status codes limited by a CHECK, or a status master table when users manage the values
- an `i_version` column for detecting conflicting updates
- stored totals, kept only when the reason is documented

## Workflow

1. **Clarify first.** Before writing a final schema, resolve or state assumptions for:
   - the project settings: name case and target vendors
   - the main read and write patterns
   - the ID format for each entity: abbreviation, date part, sequence width
   - the business time zone used for the date part of IDs
   - what fills the `_by` and `_func_nm` audit columns

   If an open question affects a mandatory rule, ask instead of guessing. Otherwise state the assumption and continue.
2. **Classify.** Mark each entity `m_` or `t_` (N2). Decide whether it needs Header/Detail (H2).
3. **Design the logical schema.** Write a column table with logical types, PK, FKs, nullability and checks for each table, including the six audit columns and `i_active`.
4. **Design the physical schema.** Read `references/dbms/<vendor>.md` for each target and generate DDL. Never assume one vendor's type or feature exists in another. For a vendor without a file, map the types following the same structure and say the mapping is unverified. Vendor-specific SQL, types and quoting live only in that vendor's migration scripts or data-access adapter, never in the logical schema.
5. **Validate.** Run the checklist in `references/schema-validation.md`. Report a pass or fail for each rule ID, and fix every failure before handing the schema over. When a database is available, create the schema on every target vendor, run the vendor's catalog audit query, and write and read back a Thai, Japanese and Chinese text sample.

**Output for a design task:**
- project settings, assumptions and decisions
- the logical tables
- the DDL for each target vendor
- the validation result

## When to read each reference

| File | Read when |
|---|---|
| `references/naming-conventions.md` | Naming anything: tables, columns, constraints, indexes, abbreviations, reserved words |
| `references/logical-types.md` | Choosing a column type, or mapping a logical type to any vendor |
| `references/keys-and-relationships.md` | Keys, ID formats and generation, FKs, Header/Detail, sub-details, junction tables |
| `references/unix-time-standard.md` | Any date, time, timestamp, duration or time-zone question |
| `references/database-design-principles.md` | Audit columns, `i_active`, normalization, nullability, status, precision, text sizes, transactions |
| `references/schema-validation.md` | Before delivering or reviewing any schema; when fixing a violation |
| `references/examples.md` | You need a compliant or non-compliant pattern to copy or contrast |
| `references/dbms/sql-server.md` | The target is Microsoft SQL Server 2016 or later, or Azure SQL |
| `references/dbms/postgresql.md` | The target is PostgreSQL 12 or later |
| `references/dbms/mysql.md` | The target is MySQL 8.0.16+ or MariaDB 10.5+ |
| `references/dbms/oracle.md` | The target is Oracle Database 19c or later |
| `references/dbms/sqlite.md` | The target is SQLite 3.38 or later |
