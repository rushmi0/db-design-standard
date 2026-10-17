# Database Design Principles

These practices support the mandatory rules. Apart from the mandatory audit and active columns, apply each one only when a requirement needs it: no column, table, constraint or abstraction goes in without a stated reason.

## 1. Audit columns (D1)

Every table, with no exceptions, has these six columns, all NOT NULL:

| Column | Logical type | Set on insert | Set on update |
|---|---|---|---|
| `i_created_at` | `timestamp` | current Unix time | never changed |
| `i_created_func_nm` | `string(100) ascii` | name of the program function that inserted the row | never changed |
| `i_created_by` | `string(50) ascii` | user ID, or process name for batch jobs | never changed |
| `i_updated_at` | `timestamp` | same value as `i_created_at` | current Unix time, by **every** UPDATE |
| `i_updated_func_nm` | `string(100) ascii` | same value as `i_created_func_nm` | name of the function that made the change |
| `i_updated_by` | `string(50) ascii` | same value as `i_created_by` | user ID or process name |

- **Who sets them.** The data-access layer sets all six on every insert and the three `updated` columns on every update. It knows the user and the function; the database does not. A database default for `i_created_at` / `i_updated_at` is allowed as a safety net. A trigger is optional; if one is used, document it and don't also set the value in code.
- **Function names.** Use one stable naming scheme for `_func_nm`, such as `<module>.<function>` (`invoice.create`, `batch.nightly_close`) or a screen/API code. Values are ASCII; the application never writes other characters there.
- **Users and processes.** `_by` holds a user ID (`usr000001`) or a fixed process name (`batch_import`, `system`). Because it can hold a process name, `_by` is **not** an FK to `m_usr`. It is still K1-compliant: it is not a key column.
- **"Updated" on insert.** The `updated` columns equal the `created` columns until the first update, so they are never NULL.
- **Setting `i_active`.** Deactivating or reactivating a row is an update and sets the three `updated` columns.
- **History.** When old values must be kept, use a separate `t_<entity>_hist` table with the same columns plus `(i_<entity>_id, i_internal_no)` as its key. Do not add history columns to the main table.

## 2. Active flag and soft deletion (D2)

Every table has `i_active boolean NOT NULL DEFAULT true`.

| Value | Meaning |
|---|---|
| `true` (`1`) | Active: the row is part of normal operation |
| `false` (`0`) | Inactive or soft-deleted: hidden from normal operation, kept for history and references |

- **No physical deletes in normal operation.** "Delete" sets `i_active = false`. Physical deletes happen only outside normal operation: test data, a documented retention purge.
- **Standard queries return active rows only.** Every repository method, API and query filters `i_active = true` (`= 1` on vendors without a boolean literal) unless the caller explicitly asks for inactive rows. A view `v_<table>` that returns active rows is a convenient default for read paths.
- **Status is not replaced.** A transaction that is cancelled stays active with `i_status = 'cancelled'`: it is a real business document that must remain visible. `i_active = false` is for rows that should disappear from normal use, such as a discarded draft or a line removed from a document.
- **Parents and children.** Deactivating a header makes its document inactive; queries on details join the header and filter on the header's `i_active`. New rows must not reference an inactive parent.
- **Unique business codes.** Choose per table, and document the choice:
  - *Unique for all time* (default): a plain `uq_` constraint. An inactive row keeps its code, so it can be reactivated without a conflict. Works on every vendor.
  - *Unique among active rows*: the code can be reused after deactivation. Needs a filtered or function-based unique index; each vendor file shows the form. Reactivating a row then needs a check that its code is still free.
- **Indexes.** Don't index `i_active` alone; it has two values. Where most rows are active, the filter costs little. Where many rows are inactive and queries are slow, use a filtered index or add `i_active` as the last column of the index the query already uses.

## 3. Booleans (D3)

- Logical type `boolean`, NOT NULL, mapped per vendor (`logical-types.md`).
- Name it as a yes/no question: `i_is_taxable`, `i_has_serial_no`. The only exception is `i_active`.
- Avoid three-state flags. If a third state exists, it is a status code.

## 4. Normalization and redundancy
- Start in third normal form: every non-key column depends on the key, the whole key, and nothing but the key.
- Some redundancy is allowed if it is recorded in the design notes, together with the rule that keeps it correct:
  - **Snapshots** (H3): values copied onto a transaction because they must not change later, such as a price on a detail line.
  - **Stored totals**: header totals read far more often than lines change. Recalculate them in the same transaction as any line change, counting active lines only.
  - **Measured hot paths**: denormalization proven necessary by query plans or load tests.
- Never store a value that can be derived cheaply and is never frozen. Age is calculated from `i_birth_date` and is not stored.

## 5. Relationships and keys
See `keys-and-relationships.md`. In short: string PKs and FKs in the `<abbr><date><sequence>` format, composite keys for detail, sub-detail and junction tables, `NO ACTION` on every FK, an index on every FK column set, and business codes in separate `uq_` columns.

## 6. Nullability and defaults (D5)
- A column is NOT NULL by default. NULL means "not applicable" or "not happened yet", and that meaning is written next to the column: `i_paid_at NULL` = not paid yet, `i_from_warehouse_id NULL` = received from outside.
- **Never store an empty string.** Oracle stores `''` as NULL, so a design that tells `''` and NULL apart cannot work on every vendor. An optional text column is nullable, and "blank" is NULL. Codes and IDs are never blank.
- Never use NULL to mean zero. An amount with nothing in it is `0`.
- Use a default only when the value really is the default: `i_active = true`, the initial status (`'draft'`), audit timestamps. Never add a default just to get around NOT NULL.

## 7. Status and lifecycle
- `i_status` is `string(20) ascii` holding a lowercase word: `draft`, `completed`, `cancelled`.
- **Where the allowed values live:**
  - When the list belongs to the application and rarely changes, use a CHECK listing the values.
  - When users manage the list, or it needs labels or translations, use a master `m_<entity>_status` plus an FK.
- **Allowed transitions:** write them in the design notes as a table, e.g. `draft → completed`, `draft → cancelled`, `completed → cancelled`. Enforce them in the application in the same transaction as the change.
- When the time of a transition matters, add `_at` columns for it (`i_completed_at`, `i_cancelled_at`). Add a status-history table only if every transition must be audited.

## 8. Numeric precision (D4)

| Data | Logical type | Note |
|---|---|---|
| Money: amounts, prices, costs, taxes | `decimal(19,4)` | Round to the currency's decimals (THB 2, JPY 0) when storing **final** document totals; lines may keep 4 |
| Quantity | `decimal(19,4)` | Covers fractional units (kg, m) |
| Rate: tax, discount, exchange | `decimal(9,6)` | `0.070000` = 7% |
| Counters | `int` / `long` | Never a key (K1) |

- **Never use** float, real, double or money types for business numbers. On SQLite, `decimal` is a scaled integer (`logical-types.md` §5).
- **Multi-currency amounts** need `i_ccy` (`string(3) ascii`, ISO 4217) on the row or its header.
- **Rounding.** Document the rounding method (half-up, or half-even for some tax rules) and where it runs: per line or per document.
- **Sign.** Constrain sign where it matters, e.g. `i_qty > 0` and `i_unit_price >= 0`. Represent a return or credit as a separate document type or status, not a hidden negative quantity, unless the business requires negatives.

## 9. Strings and international text (L2)

- **All human text is Unicode** on every vendor, including Thai, Japanese, Chinese and other non-Latin scripts. Never assume English-only storage is enough. Each vendor file states the Unicode type and the database character set it requires.
- **Lengths count characters,** not bytes. Use the standard sizes in `logical-types.md`.
- **ASCII strings** (IDs, codes, statuses, technical audit names) are limited by a CHECK, except the audit `_by` / `_func_nm` columns, which the data-access layer controls.
- **Trim before storing.** Trim surrounding spaces from codes and IDs in the application. CHECKs then reject anything else.
- **Email** is stored as entered, as `string(254)`; compare in lowercase in the application.
- **Verify international text** (step 5 of the implementation checklist): insert and read back a Thai, Japanese and Chinese sample through the real driver, and compare the strings exactly. A `?` or mojibake means the column, connection or driver is not Unicode.

## 10. Indexing and performance
- **Always index:** the PK (automatic), every FK column set (K6), every `uq_`.
- **Index when there is a real query** for it: columns in frequent `WHERE`, `JOIN` and `ORDER BY` clauses, e.g. `(i_customer_id, i_invoice_at)` for "a customer's invoices by date".
- **Composite index column order:** columns compared with `=` first, then the range or sort column.
- **Check before adding indexes.** Look at the execution plan first, and remove indexes nobody uses. Every index makes writes slower.
- **Range queries on `_at`** use `>= start AND < end` on the integer. Wrapping the column in a conversion function stops the index being used.
- **Prefer sequential IDs.** IDs that increase in insert order (abbreviation + date + counter) keep B-tree inserts at the end of the index. Random IDs scatter them.

## 11. Transactions and consistency
- **Keep a document in one transaction.** A header, its details, the counter update, stock effects and total recalculation all go in one transaction.
- **Keep transactions short.** No user interaction or network calls inside one.
- **Lock in a fixed order:** counter, then header, then details, and tables in the same order in every code path. This avoids deadlocks.
- **Lock what a decision reads.** Lock the header row before assigning `i_internal_no` or checking stock; each vendor file shows the locking syntax.
- **Optional conflict check.** `i_version` (`long`) is incremented on each update and checked in the `WHERE`. Use it when users edit the same record concurrently in a UI.
