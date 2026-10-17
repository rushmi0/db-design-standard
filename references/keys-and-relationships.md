# Keys and Relationships

## 1. String keys (K1–K4)

Every PK and FK column is `string(n) ascii`, including composite key parts and `i_internal_no`. Number, boolean and timestamp key columns are not allowed, and neither are database-generated keys (`IDENTITY`, `SERIAL`, `AUTO_INCREMENT`, SQLite row IDs). A database sequence or counter **may** produce the number inside a string ID (section 3); the key column itself is still a string.

| Rule | Requirement | Why |
|---|---|---|
| Type | `string(n) ascii`: variable length, never fixed `CHAR` | Vendors disagree on whether trailing spaces in `CHAR` count |
| Characters | Lowercase `a–z` and `0–9` only (K2), enforced by a CHECK | With one case and no spaces, case-sensitive and case-insensitive vendors compare IDs the same way |
| Collation | Binary (the vendor file names it) | Exact comparison and the same sort order everywhere |
| Length | `string(20) ascii` for entity IDs; `string(6) ascii` for line numbers | One width keeps FK types matched without effort |
| FK match | FK type, length and collation identical to the PK (K3) | Some vendors reject a mismatch; others silently skip the index |
| Stability | Never updated, never reused (K4) | References in other systems, logs and printed documents stay valid |

A business code that can change (a SKU, a tax number, a code chosen by users) is **not** the PK. Store it in its own `string(n) ascii` column with a `uq_` constraint. The PK stays a generated ID.

String keys cost a little more index space and comparison time than integers. Short, ASCII, binary-collated keys keep that cost small. This standard does not allow integer keys as a performance exception.

## 2. ID format (K7)

An entity ID is:

```
<abbr><date><sequence>
```

| Part | Rule | Examples |
|---|---|---|
| `abbr` | 2–4 lowercase letters that identify the table. Unique within the database and documented per table | `usr`, `ord`, `pkg`, `cus`, `inv` |
| `date` | `yyyymmdd` for a sequence that restarts daily, `yyyymm` for monthly, or nothing when the entity doesn't need a date | `20261017`, `202610`, — |
| `sequence` | Zero-padded number with a documented width `N` | `000001` |

| Entity | Format | Example | Length |
|---|---|---|---|
| User (master, no date) | `usr` + 6 digits | `usr000001` | 9 |
| Package (monthly) | `pkg` + `yyyymm` + 4 digits | `pkg2026100001` | 13 |
| Order (daily) | `ord` + `yyyymmdd` + 6 digits | `ord20261017000123` | 17 |

- **Date part.** The date is the **business date** in the system's documented business time zone (for example Asia/Bangkok). It is a readable label and the reset point of the counter. The exact instant is stored separately in the entity's `_at` column.
- **Width.** Choose `N` with room to grow: at least ten times the highest expected count per period (day, month, or lifetime when there is no date). The whole ID must fit in 20 characters. When the counter runs out, generation must raise an error, never wrap around or truncate. Widening the format later changes the ID length, so every FK column that references it must be widened too.
- **Pattern check.** The PK column gets a CHECK on its exact pattern, for example `ord` + 14 digits. FK columns rely on the FK constraint and need no pattern check.
- **FKs store the referenced ID as it is.** An FK column never gets its own date or sequence part.

Random IDs (UUID or ULID written in lowercase hex or base32) are allowed only for an entity that must be created without central coordination, such as offline or multi-site entry. Document the reason. They are longer, not readable, and scatter inserts in clustered indexes.

## 3. Generating IDs

| Need | Method | Concurrency and uniqueness |
|---|---|---|
| ID with a date part, or any ID on MySQL and SQLite | Counter table `m_id_seq`, keyed by `abbr` + date (or `abbr` alone), incremented atomically **in the same transaction** as the insert | The counter row is locked until the transaction ends, so two sessions can never get the same number. A rollback also rolls back the counter, so there are no gaps |
| ID without a date part, on vendors with sequences (SQL Server, PostgreSQL, MariaDB, Oracle) | Optional: a database sequence `sq_<table>` with `MAXVALUE` equal to the largest number the digits allow and `NO CYCLE` | The sequence never returns the same number twice. It may leave gaps after a rollback, which is acceptable for IDs without a date |
| `i_internal_no` | Assigned in line order when the document is saved. When lines are added later, lock the header row, then use the highest existing number plus one | The header lock serializes writers of one document |

The PK's CHECK constraint is the final uniqueness and format guarantee: a bad or duplicate value fails the insert instead of being stored.

### Counter table `m_id_seq`

| Column | Logical type | Note |
|---|---|---|
| `i_id_seq_id` | `string(20) ascii` PK | `abbr` + date, e.g. `ord20261017`, `pkg202610`, `usr` |
| `i_last_val` | `long` | last number issued |
| audit columns, `i_active` | | as on every table (D1, D2) |

Updating the counter in the same transaction as the insert means two inserts for the same `abbr` and date run one after the other. That is fine for short transactions. Each vendor file has the exact SQL; on SQL Server, PostgreSQL and Oracle it is a procedure or function `p_next_id` / `fn_next_id`.

## 4. Composite keys

Composite PKs are allowed when the data model needs them, and every part is a string. Use them for:

| Case | PK | FKs |
|---|---|---|
| Detail | `(i_<hdr>_id, i_internal_no)` | `i_<hdr>_id` → header |
| Sub-detail (detail of a detail) | `(i_<hdr>_id, i_internal_no, i_sub_no)` | `(i_<hdr>_id, i_internal_no)` → detail |
| Junction | `(i_<a>_id, i_<b>_id)` | one FK to each side |

Write the columns of a composite FK in the same order as the parent PK. Any table that references a detail row needs both key columns.

## 5. Header/Detail (H1–H3)

**Use Header/Detail when** one transaction has shared data plus many related rows: an invoice and its lines, a goods receipt and its items.

**Use one `t_` table when** each record stands alone: a stock movement, a login event, a payment that settles one invoice.

### Header
- PK is `i_<event>_id`, using the ID format with a date part.
- Holds everything shared by the whole transaction: the instant (`_at`), parties, status, currency and totals.
- Stored totals such as `i_total_amt` are a documented exception to normalization. They must equal the sum of the **active** lines when the document reaches a final status.

### Detail
- PK is `(i_<event>_id, i_internal_no)`. The FK `fk_<detail>_<event>_h` points to the header, and the PK's leading column already serves as its index.
- `i_internal_no` is `string(6) ascii`: 6 digits, zero-padded, so text order equals number order. It is unique within its header only, so `000001` appears under every invoice. It is never renumbered. A removed line is set to `i_active = false` and keeps its number.
- Copy business values at the time of the transaction (H3): description, unit price, tax rate and discount. Changing `m_product` must never change a past invoice.
- **Display order.** `i_internal_no` is the line order. Add `i_sort_order` (`int`) only if users can reorder lines.

### Second child collection and sub-detail
- A header with two kinds of children gets a second detail table with the same key shape: `t_invoice_d` for lines and `t_invoice_pay_d` for payments. Each has its own `i_internal_no` sequence.
- A detail of a detail adds `i_sub_no` (`string(6) ascii`): `t_invoice_lot_d (i_invoice_id, i_internal_no, i_sub_no)`.
- Both names end in `_d`, so they look alike. The FK name shows the parent: `fk_t_invoice_lot_d_invoice_d`.

### Integrity the database cannot enforce alone
- **"A header has at least one active detail."** Insert the header and its lines in one transaction, and check the line count before the status leaves `draft`.
- **"Totals equal the sum of active lines."** Recalculate the totals in the same transaction that changes the lines.
- **"Details of an inactive header are inactive."** Set the header's `i_active` to `false` and treat its details as inactive in queries (join the header and filter on its `i_active`). Deactivating every detail row as well is optional.

## 6. Relationships and referential integrity (K5, K6)

| Relationship | Implementation |
|---|---|
| One-to-many | FK on the "many" side |
| One-to-one | FK on the dependent side with a `uq_` on it; or the same PK value in both tables, when the dependent row cannot exist alone |
| Many-to-many | Junction table with a composite PK; add an index on the reverse order `(b, a)` if lookups run from B |
| Self-reference (a hierarchy) | Nullable role FK, e.g. `i_parent_category_id` → same table |

- **Every FK uses `NO ACTION`.** Never `CASCADE` or `SET NULL`.
  - PKs never change, so an update never needs to cascade.
  - Removing is a business action: set `i_active = false` (D2), or cancel a document through its status.
  - Rows are physically deleted only outside normal operation (test data, a retention purge). Delete the children explicitly, then the parent, in one transaction.
- **An inactive parent stays referenced.** FKs still point at it, so history stays complete. New rows must not reference an inactive parent; the application checks `i_active` when it validates the reference.
- **Index every FK column set** unless an existing PK or index already starts with those columns.
- **Never use a timestamp in a PK or FK**, not even as part of a composite key.
