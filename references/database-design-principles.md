# Database Design Principles

These practices support the mandatory rules. Apply each one only when a requirement needs it: no column, table, constraint or abstraction goes in without a stated reason.

## 1. Normalization and redundancy
- Start in third normal form. Every non-key column depends on the key, the whole key, and nothing but the key.
- Some redundancy is allowed if it is recorded in a comment or the design notes, together with the rule that keeps it correct. Allowed kinds:
  - **Snapshots** (H3): values copied onto a transaction because they must not change later, such as a price or tax rate on a detail line.
  - **Stored totals**: header totals that are read far more often than lines change. Recalculate them in the same transaction as any line change.
  - **Measured hot paths**: denormalization proven necessary by query plans or load tests.
- Never store a value that can be derived cheaply and is never frozen. Age is calculated from `I_BIRTH_DATE` and is not stored.

## 2. Relationships and keys
See `keys-and-relationships.md`. In short:
- string PKs and FKs
- composite keys for detail, sub-detail and junction tables
- `NO ACTION` on every FK
- an index on every FK column set
- business codes in separate `UQ_` columns

## 3. Nullability and defaults (D4)
- A column is NOT NULL by default. NULL means "not applicable" or "not happened yet", and that meaning is stated: `I_DELETED_AT NULL` = not deleted, `I_FROM_WAREHOUSE_ID NULL` = goods received from outside.
- Never use NULL to mean "empty" or "zero". Use `0` for an amount and `''` is not allowed for a code.
- Use a default only when the value really is the default:
  - audit timestamps
  - the initial status (`'DRAFT'`)
  - `I_IS_ACTIVE = 'Y'`

  Never set a default just to get around NOT NULL.

## 4. Audit columns (D1)

| Column | Type | Null | Default / maintained by |
|---|---|---|---|
| `I_CREATED_AT` | `UNIXTIME` | NOT NULL | Current Unix time on insert; never updated |
| `I_UPDATED_AT` | `UNIXTIME` | NOT NULL | Same as created on insert; set to the current time by **every** UPDATE statement |
| `I_CREATED_BY` | `ID(20)` | NOT NULL | Required when user accounts exist |
| `I_UPDATED_BY` | `ID(20)` | NOT NULL | Required when user accounts exist |

- **Who sets `I_UPDATED_AT`.** The data-access layer sets it. A trigger is optional; if one is used, document it and don't also set the value in code.
- **When `_BY` is an FK.** Declare `_BY` as an FK to `M_USER` only when `M_USER` is in the same database. Otherwise store the external user ID as an `ID` string with no FK. That is still K1-compliant, since it is a string, and it is documented as an external reference.
- **History.** When old values must be kept, use a separate `T_<ENTITY>_HIST` table with the same columns plus `(I_<ENTITY>_ID, I_INTERNAL_NO)` as its key. Do not add history columns to the main table.

## 5. Soft deletion (recommended, only where needed)
- **Use soft delete for masters** that are referenced by transactions or must stay restorable. Add `I_DELETED_AT UNIXTIME NULL`. A row counts as deleted when the column is not NULL.
- **Use a status for transactions instead** (`CANCELLED`, `VOID`). Do not soft-delete financial documents.
- **Unique business codes on soft-deleted tables** need a unique index that covers active rows only. That is a filtered or partial index; see the DBMS file. On a DBMS without one, either keep codes unique for all time or put the uniqueness check in the application, and document which.
- **Every query on the table must filter `I_DELETED_AT IS NULL`.** Use a view `V_<ENTITY>` for active rows when many queries need it.
- **Never mix two delete markers.** Don't add an `I_IS_DELETED` flag alongside `I_DELETED_AT`.

## 6. Status and lifecycle
- `I_STATUS` is a `CODE(20)` column holding an uppercase word: `DRAFT`, `COMPLETED`, `CANCELLED`.
- **Where the allowed values live:**
  - When the list belongs to the application and rarely changes, use a CHECK listing the values.
  - When users manage the list, or it needs labels or translations, use a master table `M_<ENTITY>_STATUS` plus an FK.
- **Allowed transitions:** write them in the design notes as a table, e.g. `DRAFT → COMPLETED`, `DRAFT → CANCELLED`, `COMPLETED → CANCELLED`. Enforce them in the application in the same transaction as the change.
- When the time of a transition matters, add `_AT` columns for it (`I_COMPLETED_AT`, `I_CANCELLED_AT`). Add a status-history table only if every transition must be audited.

## 7. Booleans (D3)
- `FLAG`: `CHAR(1)`, NOT NULL, with CHECK `IN ('Y','N')`.
- Name it as a yes/no question: `I_IS_ACTIVE`, `I_HAS_TAX`.
- Avoid three-state flags. If a third state exists, it is a status code.

## 8. Numeric precision (D2)

| Data | Type | Note |
|---|---|---|
| Money: amounts, prices, costs, taxes | `DECIMAL(19,4)` | Round to the currency's decimals (THB 2, JPY 0) when storing **final** document totals; lines may keep 4 |
| Quantity | `DECIMAL(19,4)` | Covers fractional units (kg, m) |
| Rate: tax, discount, exchange | `DECIMAL(9,6)` | `0.070000` = 7% |
| Counters | Integer | Never a key (K1) |

- **Never use** `FLOAT`, `REAL`, `DOUBLE` or `MONEY`-style types for business numbers.
- **Multi-currency amounts** need `I_CCY` (`CODE(3)`, ISO 4217) on the row or its header.
- **Rounding.** Document the rounding method (half-up, or half-even for some tax rules) and the place it runs: per line or per document.
- **Sign.** Constrain sign where it matters, e.g. `I_QTY > 0` and `I_UNIT_PRICE >= 0`. Represent a return or credit as a separate document type or status, not as a hidden negative quantity, unless the business requires negatives.

## 9. Strings, lengths and encoding (D5)

| Logical type | Length guide | Characters |
|---|---|---|
| `ID(n)` | 20 | `A–Z0-9`, binary collation |
| `LINENO` | 6 | `0–9` |
| `CODE(n)` | 3 (currency) / 20 (status) / 30 (business code) | `A–Z0-9_-` uppercase |
| `TEXT(n)` names | 200 | Unicode |
| `TEXT(n)` descriptions, addresses | 500 | Unicode |
| `TEXT(n)` notes | 2000 | Unicode |
| Email | `TEXT(254)` | Store as entered; compare in lowercase in the application |

- **Size to real data plus margin, with one of these sizes.** Don't pick a different width for each column.
- **Unicode is required for any human text,** including Thai and other non-Latin scripts.
- **Lengths count characters,** not bytes. Each DBMS file says how to get character semantics on that engine.
- **Trim before storing.** Trim surrounding spaces from codes and IDs in the application. CHECKs then reject anything else.

## 10. Indexing and performance
- **Always index:**
  - the PK, which is automatic
  - every FK column set (K6)
  - every `UQ_`
- **Index when there is a real query** for it:
  - columns in frequent `WHERE`, `JOIN`, `ORDER BY` clauses, e.g. `(I_CUSTOMER_ID, I_INVOICE_AT)` for "a customer's invoices by date"
- **Composite index column order:** columns compared with `=` first, then the range or sort column.
- **Check before adding indexes.** Look at the execution plan before adding more indexes, and remove indexes nobody uses. Every index makes writes slower.
- **Range queries on `_AT`** use `>= start AND < end` on the integer. That works with an index, whereas wrapping the column in a conversion function stops the index being used.
- **Prefer sequential IDs.** IDs that increase in insert order (prefix + date + counter) keep B-tree inserts at the end of the index. Random IDs scatter them.

## 11. Transactions and consistency
- **Keep a document in one transaction.** A header, its details, the counter update, stock effects and total recalculation all go in one transaction.
- **Keep transactions short.** No user interaction or network calls inside one.
- **Lock in a fixed order:** header before details, and tables in the same order in every code path. This avoids deadlocks.
- **Lock what a decision reads.** Use a row lock (`UPDLOCK` / `FOR UPDATE`) on the header when assigning `I_INTERNAL_NO` or checking stock before writing.
- **Optional conflict check.** `I_VERSION` (`COUNT`) is incremented on each update and checked in the `WHERE`. Use it when users edit the same record concurrently in the UI.
