# Keys and Relationships

## 1. String keys (K1–K4)

Every PK and FK column is a variable-length string, including composite key parts and `I_INTERNAL_NO`. Integer, numeric, date and timestamp key columns are not allowed, and neither are database-generated integer keys (`IDENTITY`, `SERIAL`, `AUTO_INCREMENT`). A database sequence **may** be used to produce the number inside a string ID (section 3). The sequence is a generator; the key column itself is still a string.

| Rule | Requirement | Why |
|---|---|---|
| Type | Variable-length string (`VARCHAR`), never fixed `CHAR` | DBMSs disagree on whether trailing spaces in `CHAR` are significant |
| Characters | `A–Z`, `0–9` only (K2), enforced by CHECK | With no lowercase or spaces, case-insensitive and case-sensitive DBMSs compare IDs the same way |
| Collation | Binary or byte (`Latin1_General_100_BIN2`, `"C"`) | Exact comparison and the same sort order everywhere |
| Length | `ID(20)` for entity IDs; `LINENO` for line numbers | One width keeps FK types matched without effort |
| FK match | FK type, length and collation identical to the PK (K3) | Some DBMSs reject a mismatch; others silently skip the index |
| Stability | Never updated, never reused (K4) | References in other systems, logs and printed documents stay valid |

A business code that can change (a SKU, a tax number, a product code chosen by users) is **not** the PK. Store it in its own `CODE` column with a `UQ_` constraint. The PK stays as a generated ID.

String keys cost a little more index space and comparison time than integers. Keep them short, ASCII and binary-collated, and the difference is small. This standard does not allow integer keys as a performance exception.

## 2. ID formats

Each table documents its ID format. The defaults are:

| Kind | Format | Example | Length |
|---|---|---|---|
| Master | `<PREFIX><6 digits>`, with a 2–4 letter prefix | `CUS000001` | 9–10 |
| Document (header or standalone transaction) | `<PREFIX 2 letters><YYYYMMDD><N digits>` | `IV202605030001` | 14 for N=4 |
| Line number | 6 digits | `000001` | 6 |

- **Prefix.** Each table has its own prefix, unique within the database: `CUS`, `PRD`, `WH`, `IV`, `SM`.
- **Date part.** `YYYYMMDD` is the **business date** in the system's documented business time zone (for example Asia/Bangkok). It is only a readable label. The exact instant of the document is stored separately in its `_AT` column.
- **Width.** Choose the digit count `N` with room to grow: at least ten times the highest expected count per day, or per lifetime for masters. When the counter runs out it must raise an error, never wrap around or truncate. Widening the format later changes the ID length, so all FK columns that reference it must be widened too.
- **Pattern check.** The PK column gets a CHECK on its exact pattern, for example `CUS` followed by 6 digits. Its FK columns rely on the FK constraint and need no pattern check.

Random IDs (UUID or ULID written as uppercase hex or Crockford base32) are allowed only for an entity that must be created without central coordination, such as offline or multi-site entry. Document the reason. They are longer and not readable, and fully random values fragment clustered indexes.

## 3. Generating IDs

| Need | Method |
|---|---|
| Master ID | DB sequence (`SQ_<TABLE>`) with `MAXVALUE` equal to the largest number the digits allow and `NO CYCLE`, formatted with leading zeros in the INSERT |
| Document ID with a daily counter | Counter table `M_DOC_SEQ`, keyed by prefix plus date and updated atomically **in the same transaction** as the document insert |
| `I_INTERNAL_NO` | Assigned by the application in line order when the document is saved. When lines are added to an existing document, lock the header row, then use the highest existing number plus one |

The counter table holds one row per prefix and business date:

| Column | Type | Note |
|---|---|---|
| `I_DOC_SEQ_ID` | `ID(20)` PK | prefix plus date, e.g. `IV20260503` |
| `I_LAST_VAL` | `COUNT` | last number issued |
| `I_CREATED_AT`, `I_UPDATED_AT` | `UNIXTIME` | audit |

If the counter update runs in the same transaction as the document insert, a rollback also rolls back the counter, so numbers have no gaps. The cost is that two documents with the same prefix and date are created one at a time. That is fine for short transactions. If you accept gaps, use a sequence instead and document that choice.

Each DBMS file has the exact SQL for these methods.

## 4. Composite keys

Composite PKs are allowed when the data model needs them, and every part is a string. Use them for:

| Case | PK | FKs |
|---|---|---|
| Detail | `(I_<HDR>_ID, I_INTERNAL_NO)` | `I_<HDR>_ID` → header |
| Sub-detail (detail of a detail) | `(I_<HDR>_ID, I_INTERNAL_NO, I_SUB_NO)` | `(I_<HDR>_ID, I_INTERNAL_NO)` → detail |
| Junction | `(I_<A>_ID, I_<B>_ID)` | one FK to each side |

Write the columns of a composite FK in the same order as the parent PK. Any table that references a detail row needs both key columns.

## 5. Header/Detail (H1–H3)

**Use Header/Detail when** one transaction has shared data plus many related rows: an invoice and its lines, a goods receipt and its items.

**Use one `T_` table when** each record stands alone: a stock movement, a login event, a payment that settles one invoice.

### Header
- PK is `I_<EVENT>_ID`, using the document ID format.
- Holds everything shared by the whole transaction: the instant (`_AT`), parties, status, currency and totals.
- Stored totals such as `I_TOTAL_AMT` are a documented exception to normalization (see the design principles reference). They must equal the sum of the lines when the document reaches a final status.

### Detail
- PK is `(I_<EVENT>_ID, I_INTERNAL_NO)`. The FK `FK_<DETAIL>_<EVENT>_H` points to the header, and the PK's leading column already serves as its index.
- `I_INTERNAL_NO` is `LINENO`: 6 digits, zero-padded, so that text order equals number order. It is unique within its header only, so the same `000001` appears under every invoice. It is never renumbered, and gaps after a deletion are allowed.
- Copy business values at the time of the transaction (H3): description, unit price, tax rate and discount. Changing `M_PRODUCT` must never change a past invoice.
- **Display order.** `I_INTERNAL_NO` is the line order. Add `I_SORT_ORDER` (`COUNT`) only if users can reorder lines.

### Second child collection and sub-detail
- A header with two kinds of children gets a second detail table with the same key shape: `T_INVOICE_D` for lines and `T_INVOICE_PAY_D` for payments. Each has its own `I_INTERNAL_NO` sequence.
- A detail of a detail adds `I_SUB_NO` (`LINENO`): `T_INVOICE_LOT_D (I_INVOICE_ID, I_INTERNAL_NO, I_SUB_NO)`.
- Both names end in `_D`, so they look alike. The FK name shows the parent: `FK_T_INVOICE_LOT_D_INVOICE_D`.

### Integrity the DB cannot enforce alone
- **"A header has at least one detail."** Insert the header and its lines in one transaction, and check the line count before the status leaves `DRAFT`.
- **"Totals equal the sum of lines."** Recalculate the totals in the same transaction that changes the lines.

## 6. Relationships and referential integrity (K5, K6)

| Relationship | Implementation |
|---|---|
| One-to-many | FK on the "many" side |
| One-to-one | FK on the dependent side with a `UQ_` on it; or the same PK value in both tables, when the dependent row cannot exist alone |
| Many-to-many | Junction table with a composite PK; add a `UQ_` or index on the reverse order `(B, A)` if lookups run from B |
| Self-reference (a hierarchy) | Nullable role FK, e.g. `I_PARENT_CATEGORY_ID` → same table |

- **Every FK uses `NO ACTION`.** Never use `CASCADE` or `SET NULL`.
  - PKs never change, so an update never needs to cascade.
  - Deleting is a business action: cancel the document through its status, or soft-delete the master row.
  - When a real delete is needed (draft data, test data), delete the children explicitly, then the parent, in one transaction.
- **Index every FK column set** unless an existing PK or index already starts with those columns.
- **Never use a timestamp in a PK or FK**, not even as part of a composite key.
