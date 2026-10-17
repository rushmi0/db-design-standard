# SQLite Implementation

Requires **SQLite 3.38 or later** (`STRICT` tables, `unixepoch()`, `RETURNING`). Check with `SELECT sqlite_version()`; language runtimes often bundle their own copy. This file maps the logical standard to SQLite. It never changes a logical rule.

SQLite is an embedded, single-writer database. It has no stored procedures, sequences, time-zone database, exact decimal type or schema-level users. The sections below give the compatible solution for each.

## 1. Type mapping

Every table is declared `STRICT, WITHOUT ROWID`. `STRICT` makes SQLite enforce the declared column types; without it, any value fits in any column. `WITHOUT ROWID` stores rows in PK order and removes the hidden integer row ID, so the string PK is the only key. `STRICT` tables accept only `INTEGER`, `TEXT`, `REAL`, `BLOB` and `ANY`.

| Logical | SQLite | Notes |
|---|---|---|
| `string(n)` | `TEXT` + CHECK `length(col) <= n` | Stored as UTF-8; `length()` counts characters |
| `string(n) ascii` | `TEXT` + CHECK on length and characters | Default `BINARY` collation: exact, case-sensitive (K2). Never declare `COLLATE NOCASE` on these columns |
| `int` | `INTEGER` | |
| `long` | `INTEGER` | 64-bit signed |
| `boolean` | `INTEGER` + CHECK `IN (0,1)` | `1` = true |
| `decimal(p,s)` | `INTEGER` holding the value × 10^s | See below |
| `timestamp` | `INTEGER` | Unix seconds |

**Decimals (documented vendor conflict).** SQLite has no exact decimal type: `NUMERIC` and `REAL` store fractions as floating point, which D4 forbids. A `decimal(p,s)` column therefore stores the value multiplied by 10^s as an `INTEGER`:

| Logical | Value | Stored |
|---|---|---|
| `decimal(19,4)` | `150.0000` | `1500000` |
| `decimal(9,6)` | `0.070000` | `70000` |

- The column keeps its logical name and type in the design; only the physical form changes. Mark each such column in the DDL with a comment (`-- decimal(19,4), x10^4`).
- The data-access layer converts on every read and write. Use the language's decimal type (Python `Decimal`, Java `BigDecimal`, C# `decimal`), never floating point.
- `SUM` and comparisons are exact. Multiplying two scaled values must divide by the second scale and round: `qty × price` = `(i_qty * i_unit_price + 5000) / 10000` for positive values at scale 4.
- The largest storable value is 922,337,203,685,477.5807 at scale 4.

## 2. Names and quoting
- Names are matched without regard to case and stored as written. Write them unquoted in the project case.
- Index names must be unique in the database; the `ix_<table>_…` pattern ensures this.
- SQLite keeps constraint names only in the `CREATE TABLE` text: they appear in error messages but can't be queried. Write them anyway (N6); the catalog audit checks index names only.
- Define unique business codes with `CREATE UNIQUE INDEX uq_…`, not an inline `UNIQUE` constraint. An inline constraint creates an index named `sqlite_autoindex_…` and can't be changed later without rebuilding the table.
- SQLite has no `ALTER TABLE … ADD CONSTRAINT`. Changing a constraint means creating a new table, copying the rows and renaming; plan constraints before the first release.

## 3. Unix time (seconds)

| Task | Expression |
|---|---|
| Now | `unixepoch()` |
| Unix → UTC text | `datetime(u, 'unixepoch')` → `'2026-10-17 07:30:15'` |
| UTC text → Unix | `unixepoch('2026-10-17 07:30:15')` |
| Date text → `_date` value | `unixepoch('2026-10-17')` |
| `_date` value → date text | `date(u, 'unixepoch')` |
| Start of a local business day (Bangkok, fixed +07:00) | `unixepoch('2026-10-17') - 7 * 3600` |
| Business date for an ID | `strftime('%Y%m%d', 'now', '+7 hours')` |

- `unixepoch()` returns whole seconds.
- SQLite has no time-zone database. Its `'localtime'` modifier uses the server's zone, which the standard never relies on. Use a fixed offset for zones without daylight saving, or calculate local-day bounds in the application.

## 4. CHECK patterns
SQLite has no built-in `REGEXP`. `GLOB` is case-sensitive and must match the whole value, so it replaces anchored regular expressions:

```sql
CHECK (length(i_customer_id) = 9 AND i_customer_id GLOB 'cus[0-9][0-9][0-9][0-9][0-9][0-9]')  -- fixed-format ID
CHECK (length(i_sku) BETWEEN 1 AND 30 AND i_sku NOT GLOB '*[^A-Za-z0-9_-]*')                 -- business code
CHECK (length(i_customer_nm) <= 200)                                                          -- Unicode text
CHECK (i_active IN (0, 1))                                                                    -- boolean
CHECK (i_due_date % 86400 = 0)                                                                -- calendar date
```

## 5. Example schema (compliant)

Foreign keys are enforced only when each connection runs `PRAGMA foreign_keys = ON`. Set it in the connection factory, every time. The data-access layer supplies the `func_nm` and `by` audit values on every insert.

```sql
PRAGMA foreign_keys = ON;

-- Counter for IDs (configuration -> master)
CREATE TABLE m_id_seq (
    i_id_seq_id        TEXT    NOT NULL,
    i_last_val         INTEGER NOT NULL,
    i_created_at       INTEGER NOT NULL DEFAULT (unixepoch()),
    i_created_func_nm  TEXT    NOT NULL CHECK (length(i_created_func_nm) <= 100),
    i_created_by       TEXT    NOT NULL CHECK (length(i_created_by) <= 50),
    i_updated_at       INTEGER NOT NULL DEFAULT (unixepoch()),
    i_updated_func_nm  TEXT    NOT NULL CHECK (length(i_updated_func_nm) <= 100),
    i_updated_by       TEXT    NOT NULL CHECK (length(i_updated_by) <= 50),
    i_active           INTEGER NOT NULL DEFAULT 1,
    CONSTRAINT pk_m_id_seq PRIMARY KEY (i_id_seq_id),
    CONSTRAINT ck_m_id_seq_id_seq_id CHECK (length(i_id_seq_id) BETWEEN 2 AND 12
        AND i_id_seq_id GLOB '[a-z][a-z]*' AND i_id_seq_id NOT GLOB '*[^a-z0-9]*'),
    CONSTRAINT ck_m_id_seq_last_val CHECK (i_last_val >= 0),
    CONSTRAINT ck_m_id_seq_active CHECK (i_active IN (0, 1))
) STRICT, WITHOUT ROWID;

-- Master, ID without a date part: cus + 6 digits
CREATE TABLE m_customer (
    i_customer_id      TEXT    NOT NULL,
    i_customer_nm      TEXT    NOT NULL,
    i_tax_no           TEXT    NULL,  -- NULL = not registered for tax
    i_created_at       INTEGER NOT NULL DEFAULT (unixepoch()),
    i_created_func_nm  TEXT    NOT NULL CHECK (length(i_created_func_nm) <= 100),
    i_created_by       TEXT    NOT NULL CHECK (length(i_created_by) <= 50),
    i_updated_at       INTEGER NOT NULL DEFAULT (unixepoch()),
    i_updated_func_nm  TEXT    NOT NULL CHECK (length(i_updated_func_nm) <= 100),
    i_updated_by       TEXT    NOT NULL CHECK (length(i_updated_by) <= 50),
    i_active           INTEGER NOT NULL DEFAULT 1,
    CONSTRAINT pk_m_customer PRIMARY KEY (i_customer_id),
    CONSTRAINT ck_m_customer_customer_id CHECK (length(i_customer_id) = 9
        AND i_customer_id GLOB 'cus[0-9][0-9][0-9][0-9][0-9][0-9]'),
    CONSTRAINT ck_m_customer_customer_nm CHECK (length(i_customer_nm) BETWEEN 1 AND 200),
    CONSTRAINT ck_m_customer_tax_no CHECK (length(i_tax_no) BETWEEN 1 AND 30 AND i_tax_no NOT GLOB '*[^A-Za-z0-9_-]*'),
    CONSTRAINT ck_m_customer_active CHECK (i_active IN (0, 1))
) STRICT, WITHOUT ROWID;

-- Master, monthly ID: prd + yyyymm + 4 digits; changeable business code in its own column
CREATE TABLE m_product (
    i_product_id       TEXT    NOT NULL,
    i_sku              TEXT    NOT NULL,
    i_product_nm       TEXT    NOT NULL,
    i_unit_price       INTEGER NOT NULL,  -- decimal(19,4), x10^4
    i_is_taxable       INTEGER NOT NULL DEFAULT 1,
    i_created_at       INTEGER NOT NULL DEFAULT (unixepoch()),
    i_created_func_nm  TEXT    NOT NULL CHECK (length(i_created_func_nm) <= 100),
    i_created_by       TEXT    NOT NULL CHECK (length(i_created_by) <= 50),
    i_updated_at       INTEGER NOT NULL DEFAULT (unixepoch()),
    i_updated_func_nm  TEXT    NOT NULL CHECK (length(i_updated_func_nm) <= 100),
    i_updated_by       TEXT    NOT NULL CHECK (length(i_updated_by) <= 50),
    i_active           INTEGER NOT NULL DEFAULT 1,
    CONSTRAINT pk_m_product PRIMARY KEY (i_product_id),
    CONSTRAINT ck_m_product_product_id CHECK (length(i_product_id) = 13
        AND i_product_id GLOB 'prd[0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9]'),
    CONSTRAINT ck_m_product_sku CHECK (length(i_sku) BETWEEN 1 AND 30 AND i_sku NOT GLOB '*[^A-Za-z0-9_-]*'),
    CONSTRAINT ck_m_product_product_nm CHECK (length(i_product_nm) BETWEEN 1 AND 200),
    CONSTRAINT ck_m_product_unit_price CHECK (i_unit_price >= 0),
    CONSTRAINT ck_m_product_is_taxable CHECK (i_is_taxable IN (0, 1)),
    CONSTRAINT ck_m_product_active CHECK (i_active IN (0, 1))
) STRICT, WITHOUT ROWID;
CREATE UNIQUE INDEX uq_m_product_sku ON m_product (i_sku);   -- unique for all time (default)

-- Master with times of day: wh + 4 digits
CREATE TABLE m_warehouse (
    i_warehouse_id     TEXT    NOT NULL,
    i_warehouse_nm     TEXT    NOT NULL,
    i_open_time        INTEGER NOT NULL,  -- seconds after local midnight
    i_close_time       INTEGER NOT NULL,
    i_created_at       INTEGER NOT NULL DEFAULT (unixepoch()),
    i_created_func_nm  TEXT    NOT NULL CHECK (length(i_created_func_nm) <= 100),
    i_created_by       TEXT    NOT NULL CHECK (length(i_created_by) <= 50),
    i_updated_at       INTEGER NOT NULL DEFAULT (unixepoch()),
    i_updated_func_nm  TEXT    NOT NULL CHECK (length(i_updated_func_nm) <= 100),
    i_updated_by       TEXT    NOT NULL CHECK (length(i_updated_by) <= 50),
    i_active           INTEGER NOT NULL DEFAULT 1,
    CONSTRAINT pk_m_warehouse PRIMARY KEY (i_warehouse_id),
    CONSTRAINT ck_m_warehouse_warehouse_id CHECK (length(i_warehouse_id) = 6
        AND i_warehouse_id GLOB 'wh[0-9][0-9][0-9][0-9]'),
    CONSTRAINT ck_m_warehouse_warehouse_nm CHECK (length(i_warehouse_nm) BETWEEN 1 AND 200),
    CONSTRAINT ck_m_warehouse_open_time CHECK (i_open_time BETWEEN 0 AND 86399),
    CONSTRAINT ck_m_warehouse_close_time CHECK (i_close_time BETWEEN 0 AND 86399),
    CONSTRAINT ck_m_warehouse_active CHECK (i_active IN (0, 1))
) STRICT, WITHOUT ROWID;

-- Header, daily ID: inv + yyyymmdd + 6 digits
CREATE TABLE t_invoice_h (
    i_invoice_id       TEXT    NOT NULL,
    i_invoice_at       INTEGER NOT NULL,
    i_due_date         INTEGER NULL,  -- NULL = payable immediately
    i_customer_id      TEXT    NOT NULL,
    i_status           TEXT    NOT NULL DEFAULT 'draft',
    i_ccy              TEXT    NOT NULL,
    i_total_amt        INTEGER NOT NULL DEFAULT 0,  -- decimal(19,4), x10^4; stored total: sum of active lines
    i_note             TEXT    NULL,               -- NULL = no note
    i_created_at       INTEGER NOT NULL DEFAULT (unixepoch()),
    i_created_func_nm  TEXT    NOT NULL CHECK (length(i_created_func_nm) <= 100),
    i_created_by       TEXT    NOT NULL CHECK (length(i_created_by) <= 50),
    i_updated_at       INTEGER NOT NULL DEFAULT (unixepoch()),
    i_updated_func_nm  TEXT    NOT NULL CHECK (length(i_updated_func_nm) <= 100),
    i_updated_by       TEXT    NOT NULL CHECK (length(i_updated_by) <= 50),
    i_active           INTEGER NOT NULL DEFAULT 1,
    CONSTRAINT pk_t_invoice_h PRIMARY KEY (i_invoice_id),
    CONSTRAINT fk_t_invoice_h_customer FOREIGN KEY (i_customer_id)
        REFERENCES m_customer (i_customer_id) ON DELETE NO ACTION ON UPDATE NO ACTION,
    CONSTRAINT ck_t_invoice_h_invoice_id CHECK (length(i_invoice_id) = 17
        AND i_invoice_id GLOB 'inv[0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9]'),
    CONSTRAINT ck_t_invoice_h_due_date CHECK (i_due_date % 86400 = 0),
    CONSTRAINT ck_t_invoice_h_status CHECK (i_status IN ('draft', 'completed', 'cancelled')),
    CONSTRAINT ck_t_invoice_h_ccy CHECK (length(i_ccy) = 3 AND i_ccy GLOB '[A-Z][A-Z][A-Z]'),
    CONSTRAINT ck_t_invoice_h_total_amt CHECK (i_total_amt >= 0),
    CONSTRAINT ck_t_invoice_h_note CHECK (length(i_note) BETWEEN 1 AND 2000),
    CONSTRAINT ck_t_invoice_h_active CHECK (i_active IN (0, 1))
) STRICT, WITHOUT ROWID;
-- FK index; also serves "a customer's invoices by date"
CREATE INDEX ix_t_invoice_h_customer_id_invoice_at ON t_invoice_h (i_customer_id, i_invoice_at);

-- Detail: composite PK (header ID, i_internal_no)
CREATE TABLE t_invoice_d (
    i_invoice_id       TEXT    NOT NULL,
    i_internal_no      TEXT    NOT NULL,
    i_product_id       TEXT    NOT NULL,
    i_desc             TEXT    NOT NULL,  -- snapshot of the product name
    i_qty              INTEGER NOT NULL,  -- decimal(19,4), x10^4
    i_unit_price       INTEGER NOT NULL,  -- decimal(19,4), x10^4; snapshot at sale time
    i_tax_rate         INTEGER NOT NULL,  -- decimal(9,6), x10^6; snapshot at sale time
    i_line_amt         INTEGER NOT NULL,  -- decimal(19,4), x10^4
    i_created_at       INTEGER NOT NULL DEFAULT (unixepoch()),
    i_created_func_nm  TEXT    NOT NULL CHECK (length(i_created_func_nm) <= 100),
    i_created_by       TEXT    NOT NULL CHECK (length(i_created_by) <= 50),
    i_updated_at       INTEGER NOT NULL DEFAULT (unixepoch()),
    i_updated_func_nm  TEXT    NOT NULL CHECK (length(i_updated_func_nm) <= 100),
    i_updated_by       TEXT    NOT NULL CHECK (length(i_updated_by) <= 50),
    i_active           INTEGER NOT NULL DEFAULT 1,
    CONSTRAINT pk_t_invoice_d PRIMARY KEY (i_invoice_id, i_internal_no),
    CONSTRAINT fk_t_invoice_d_invoice_h FOREIGN KEY (i_invoice_id)
        REFERENCES t_invoice_h (i_invoice_id) ON DELETE NO ACTION ON UPDATE NO ACTION,
    CONSTRAINT fk_t_invoice_d_product FOREIGN KEY (i_product_id)
        REFERENCES m_product (i_product_id) ON DELETE NO ACTION ON UPDATE NO ACTION,
    CONSTRAINT ck_t_invoice_d_internal_no CHECK (length(i_internal_no) = 6
        AND i_internal_no GLOB '[0-9][0-9][0-9][0-9][0-9][0-9]' AND i_internal_no <> '000000'),
    CONSTRAINT ck_t_invoice_d_desc CHECK (length(i_desc) BETWEEN 1 AND 500),
    CONSTRAINT ck_t_invoice_d_qty CHECK (i_qty > 0),
    CONSTRAINT ck_t_invoice_d_unit_price CHECK (i_unit_price >= 0),
    CONSTRAINT ck_t_invoice_d_tax_rate CHECK (i_tax_rate BETWEEN 0 AND 1000000),
    CONSTRAINT ck_t_invoice_d_active CHECK (i_active IN (0, 1))
) STRICT, WITHOUT ROWID;
-- FK to the header is covered by the PK's leading column; FK to the product needs its own index
CREATE INDEX ix_t_invoice_d_product_id ON t_invoice_d (i_product_id);

-- Standalone transaction: one record per event, role FKs to the same master
CREATE TABLE t_stock_move (
    i_stock_move_id      TEXT    NOT NULL,
    i_move_at            INTEGER NOT NULL,
    i_product_id         TEXT    NOT NULL,
    i_from_warehouse_id  TEXT    NULL,  -- NULL = received from outside
    i_to_warehouse_id    TEXT    NULL,  -- NULL = shipped out
    i_qty                INTEGER NOT NULL,  -- decimal(19,4), x10^4
    i_created_at         INTEGER NOT NULL DEFAULT (unixepoch()),
    i_created_func_nm    TEXT    NOT NULL CHECK (length(i_created_func_nm) <= 100),
    i_created_by         TEXT    NOT NULL CHECK (length(i_created_by) <= 50),
    i_updated_at         INTEGER NOT NULL DEFAULT (unixepoch()),
    i_updated_func_nm    TEXT    NOT NULL CHECK (length(i_updated_func_nm) <= 100),
    i_updated_by         TEXT    NOT NULL CHECK (length(i_updated_by) <= 50),
    i_active             INTEGER NOT NULL DEFAULT 1,
    CONSTRAINT pk_t_stock_move PRIMARY KEY (i_stock_move_id),
    CONSTRAINT fk_t_stock_move_product FOREIGN KEY (i_product_id)
        REFERENCES m_product (i_product_id) ON DELETE NO ACTION ON UPDATE NO ACTION,
    CONSTRAINT fk_t_stock_move_from_warehouse FOREIGN KEY (i_from_warehouse_id)
        REFERENCES m_warehouse (i_warehouse_id) ON DELETE NO ACTION ON UPDATE NO ACTION,
    CONSTRAINT fk_t_stock_move_to_warehouse FOREIGN KEY (i_to_warehouse_id)
        REFERENCES m_warehouse (i_warehouse_id) ON DELETE NO ACTION ON UPDATE NO ACTION,
    CONSTRAINT ck_t_stock_move_stock_move_id CHECK (length(i_stock_move_id) = 17
        AND i_stock_move_id GLOB 'smv[0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9]'),
    CONSTRAINT ck_t_stock_move_qty CHECK (i_qty > 0),
    CONSTRAINT ck_t_stock_move_warehouse CHECK (
        (i_from_warehouse_id IS NOT NULL OR i_to_warehouse_id IS NOT NULL)
        AND (i_from_warehouse_id IS NULL OR i_to_warehouse_id IS NULL OR i_from_warehouse_id <> i_to_warehouse_id)),
    CONSTRAINT ck_t_stock_move_active CHECK (i_active IN (0, 1))
) STRICT, WITHOUT ROWID;
CREATE INDEX ix_t_stock_move_product_id_move_at ON t_stock_move (i_product_id, i_move_at);
CREATE INDEX ix_t_stock_move_from_warehouse_id ON t_stock_move (i_from_warehouse_id);
CREATE INDEX ix_t_stock_move_to_warehouse_id ON t_stock_move (i_to_warehouse_id);

-- Default read path: active rows only (D2)
CREATE VIEW v_m_product AS SELECT * FROM m_product WHERE i_active = 1;
```

## 6. ID generation and documents

### ID from the counter table
SQLite has no procedures or sequences, so the counter is incremented with an upsert, and the new number is read back in the same transaction. Start every write transaction with `BEGIN IMMEDIATE`: it takes the database write lock at once, so two connections can never read the same counter value, and nothing deadlocks halfway through.

In application code, run the upsert with `RETURNING i_last_val` and format the ID in the application. The scripts below read the counter back with `INSERT … SELECT` instead, so they also run as plain SQL:

```sql
BEGIN IMMEDIATE;

-- Atomic insert-or-increment of the counter for 'cus' (no date part)
INSERT INTO m_id_seq (i_id_seq_id, i_last_val, i_created_func_nm, i_created_by, i_updated_func_nm, i_updated_by)
VALUES ('cus', 1, 'customer.create', 'usr000001', 'customer.create', 'usr000001')
ON CONFLICT (i_id_seq_id) DO UPDATE
   SET i_last_val = i_last_val + 1, i_updated_at = unixepoch(),
       i_updated_func_nm = excluded.i_updated_func_nm, i_updated_by = excluded.i_updated_by;

INSERT INTO m_customer (i_customer_id, i_customer_nm,
                        i_created_func_nm, i_created_by, i_updated_func_nm, i_updated_by)
SELECT 'cus' || printf('%06d', i_last_val), 'บริษัท ตัวอย่าง จำกัด',
       'customer.create', 'usr000001', 'customer.create', 'usr000001'
  FROM m_id_seq WHERE i_id_seq_id = 'cus';

-- Monthly counter for products: prd + yyyymm (Bangkok business date)
INSERT INTO m_id_seq (i_id_seq_id, i_last_val, i_created_func_nm, i_created_by, i_updated_func_nm, i_updated_by)
VALUES ('prd' || strftime('%Y%m', 'now', '+7 hours'), 1, 'product.create', 'usr000001', 'product.create', 'usr000001')
ON CONFLICT (i_id_seq_id) DO UPDATE
   SET i_last_val = i_last_val + 1, i_updated_at = unixepoch(),
       i_updated_func_nm = excluded.i_updated_func_nm, i_updated_by = excluded.i_updated_by;

INSERT INTO m_product (i_product_id, i_sku, i_product_nm, i_unit_price,
                       i_created_func_nm, i_created_by, i_updated_func_nm, i_updated_by)
SELECT i_id_seq_id || printf('%04d', i_last_val), 'SKU-001', '商品 A', 1000000,   -- 100.0000
       'product.create', 'usr000001', 'product.create', 'usr000001'
  FROM m_id_seq WHERE i_id_seq_id = 'prd' || strftime('%Y%m', 'now', '+7 hours');

INSERT INTO m_id_seq (i_id_seq_id, i_last_val, i_created_func_nm, i_created_by, i_updated_func_nm, i_updated_by)
VALUES ('wh', 1, 'warehouse.create', 'usr000001', 'warehouse.create', 'usr000001')
ON CONFLICT (i_id_seq_id) DO UPDATE
   SET i_last_val = i_last_val + 1, i_updated_at = unixepoch(),
       i_updated_func_nm = excluded.i_updated_func_nm, i_updated_by = excluded.i_updated_by;

INSERT INTO m_warehouse (i_warehouse_id, i_warehouse_nm, i_open_time, i_close_time,
                         i_created_func_nm, i_created_by, i_updated_func_nm, i_updated_by)
SELECT 'wh' || printf('%04d', i_last_val), '東京倉庫', 30600, 64800,
       'warehouse.create', 'usr000001', 'warehouse.create', 'usr000001'
  FROM m_id_seq WHERE i_id_seq_id = 'wh';

COMMIT;
```

When a counter passes its width, `printf` produces one digit more than the format, and the PK's length CHECK rejects the row. The counter can't overflow silently.

### Creating a Header/Detail document
```sql
BEGIN IMMEDIATE;

INSERT INTO m_id_seq (i_id_seq_id, i_last_val, i_created_func_nm, i_created_by, i_updated_func_nm, i_updated_by)
VALUES ('inv' || strftime('%Y%m%d', 'now', '+7 hours'), 1, 'invoice.create', 'usr000001', 'invoice.create', 'usr000001')
ON CONFLICT (i_id_seq_id) DO UPDATE
   SET i_last_val = i_last_val + 1, i_updated_at = unixepoch(),
       i_updated_func_nm = excluded.i_updated_func_nm, i_updated_by = excluded.i_updated_by;

INSERT INTO t_invoice_h (i_invoice_id, i_invoice_at, i_customer_id, i_ccy,
                         i_created_func_nm, i_created_by, i_updated_func_nm, i_updated_by)
SELECT s.i_id_seq_id || printf('%06d', s.i_last_val), unixepoch(), c.i_customer_id, 'THB',
       'invoice.create', 'usr000001', 'invoice.create', 'usr000001'
  FROM m_id_seq s, m_customer c
 WHERE s.i_id_seq_id = 'inv' || strftime('%Y%m%d', 'now', '+7 hours')
   AND c.i_customer_id = 'cus000001' AND c.i_active = 1;   -- never reference an inactive parent

-- Lines are numbered 000001, 000002, ...; price, tax and name are copied (H3).
-- Quantity 2 = 20000 at scale 4; line amount = qty x price, rescaled.
INSERT INTO t_invoice_d (i_invoice_id, i_internal_no, i_product_id, i_desc,
                         i_qty, i_unit_price, i_tax_rate, i_line_amt,
                         i_created_func_nm, i_created_by, i_updated_func_nm, i_updated_by)
SELECT (SELECT MAX(i_invoice_id) FROM t_invoice_h), '000001', p.i_product_id, p.i_product_nm,
       20000, p.i_unit_price, 70000, (20000 * p.i_unit_price + 5000) / 10000,
       'invoice.create', 'usr000001', 'invoice.create', 'usr000001'
  FROM v_m_product p
 ORDER BY p.i_product_id
 LIMIT 1;

-- Recalculate the stored total from active lines, in the same transaction
UPDATE t_invoice_h
   SET i_total_amt = COALESCE((SELECT SUM(d.i_line_amt) FROM t_invoice_d d
                                WHERE d.i_invoice_id = t_invoice_h.i_invoice_id AND d.i_active = 1), 0),
       i_updated_at = unixepoch(), i_updated_func_nm = 'invoice.create', i_updated_by = 'usr000001'
 WHERE i_invoice_id = (SELECT MAX(i_invoice_id) FROM t_invoice_h);

COMMIT;
```

### Adding a line to an existing document
`BEGIN IMMEDIATE` already serializes all writers, so no row lock is needed. The status check stays:

```sql
BEGIN IMMEDIATE;

-- Next line number = highest + 1, counting inactive lines too (numbers are never reused).
-- Past 999999, printf gives 7 digits and the length CHECK stops the insert.
INSERT INTO t_invoice_d (i_invoice_id, i_internal_no, i_product_id, i_desc,
                         i_qty, i_unit_price, i_tax_rate, i_line_amt,
                         i_created_func_nm, i_created_by, i_updated_func_nm, i_updated_by)
SELECT h.i_invoice_id,
       printf('%06d', COALESCE((SELECT MAX(CAST(d.i_internal_no AS INTEGER)) FROM t_invoice_d d
                                 WHERE d.i_invoice_id = h.i_invoice_id), 0) + 1),
       (SELECT MIN(i_product_id) FROM v_m_product), 'Product B', 10000, 1500000, 70000, 1500000,
       'invoice.add_line', 'usr000001', 'invoice.add_line', 'usr000001'
  FROM t_invoice_h h
 WHERE h.i_invoice_id = (SELECT MAX(i_invoice_id) FROM t_invoice_h)
   AND h.i_status = 'draft' AND h.i_active = 1;
-- The application checks that one row was inserted (changes() = 1).

COMMIT;
```

### Removing a line (soft delete)
```sql
UPDATE t_invoice_d
   SET i_active = 0, i_updated_at = unixepoch(),
       i_updated_func_nm = 'invoice.remove_line', i_updated_by = 'usr000001'
 WHERE i_invoice_id = (SELECT MAX(i_invoice_id) FROM t_invoice_h) AND i_internal_no = '000002';
```
Recalculate the header total in the same transaction, as above.

## 7. Unique codes among active rows only
SQLite supports partial indexes. Because the default uniqueness is a separate index (section 2), switching is two statements:

```sql
DROP INDEX uq_m_product_sku;
CREATE UNIQUE INDEX uq_m_product_sku ON m_product (i_sku) WHERE i_active = 1;
```

## 8. Performance notes
- **One writer at a time.** SQLite serializes writes across the whole database. Keep write transactions short, start them with `BEGIN IMMEDIATE`, and set a busy timeout (`PRAGMA busy_timeout = 5000`) so waiting writers retry instead of failing.
- **WAL mode** (`PRAGMA journal_mode = WAL`, set once per database) lets readers run while a write is in progress.
- **`WITHOUT ROWID` tables** are clustered on the string PK, so lookups by ID read one B-tree.
- **Indexes are not created for FKs automatically.** Create the `ix_` indexes yourself (K6).
- **Range queries on `_at`** stay integer comparisons. Never wrap the column in `datetime()` inside a `WHERE`.

## 9. Catalog audit query

This query lists violations of the rules that can be checked through SQLite's `pragma_*` table functions: L1 (every table `STRICT`), N1, N2, N4, N5, N6 (index names), K1, K3, K5, K6, U1, D1, D2, D3 and D4. Constraint names and column collations are not queryable in SQLite; check them in the DDL. An empty result means every automated check passed. The name pattern is for a lowercase project; for uppercase, change the `GLOB` character classes.

```sql
WITH t AS (
    SELECT name AS tbl, strict FROM pragma_table_list
     WHERE schema = 'main' AND type = 'table' AND name NOT LIKE 'sqlite\_%' ESCAPE '\'
), col AS (
    SELECT t.tbl, c.name AS col, upper(c.type) AS typ, c."notnull" AS nn, c.pk
      FROM t, pragma_table_info(t.tbl) c
), fk AS (
    SELECT t.tbl, f.id, f.seq, f."table" AS ptbl, f."from" AS ccol, f."to" AS pcol, f.on_update, f.on_delete
      FROM t, pragma_foreign_key_list(t.tbl) f
), audit (c, kind) AS (
    VALUES ('i_created_at', 'INTEGER'), ('i_created_func_nm', 'TEXT'), ('i_created_by', 'TEXT'),
           ('i_updated_at', 'INTEGER'), ('i_updated_func_nm', 'TEXT'), ('i_updated_by', 'TEXT'),
           ('i_active', 'INTEGER')
)
SELECT 'L1' AS rule_id, tbl AS object_name, 'table must be STRICT' AS problem FROM t WHERE strict = 0
UNION ALL SELECT 'N1', tbl, 'name must be a-z0-9_, start with a letter, no __ or trailing _'
  FROM t WHERE tbl GLOB '*[^a-z0-9_]*' OR tbl NOT GLOB '[a-z]*' OR tbl GLOB '*__*' OR tbl GLOB '*_'
UNION ALL SELECT 'N2', tbl, 'table must start with m_ or t_' FROM t WHERE substr(tbl, 1, 2) NOT IN ('m_', 't_')
UNION ALL SELECT 'N5', tbl, 'table name longer than 30' FROM t WHERE length(tbl) > 30
UNION ALL SELECT 'N1', tbl || '.' || col, 'name must be a-z0-9_, start with a letter, no __ or trailing _'
  FROM col WHERE col GLOB '*[^a-z0-9_]*' OR col NOT GLOB '[a-z]*' OR col GLOB '*__*' OR col GLOB '*_'
UNION ALL SELECT 'N4', tbl || '.' || col, 'column must start with i_' FROM col WHERE substr(col, 1, 2) <> 'i_'
UNION ALL SELECT 'N5', tbl || '.' || col, 'column name longer than 30' FROM col WHERE length(col) > 30
UNION ALL SELECT 'K1', tbl || '.' || col, 'key column must be TEXT, is ' || typ
  FROM col WHERE typ <> 'TEXT'
   AND (pk > 0 OR EXISTS (SELECT 1 FROM fk WHERE fk.tbl = col.tbl AND fk.ccol = col.col))
UNION ALL SELECT 'K3', fk.tbl || '.' || fk.ccol, 'FK column type differs from ' || fk.ptbl || '.' || fk.pcol
  FROM fk JOIN col c ON c.tbl = fk.tbl AND c.col = fk.ccol
          JOIN col p ON p.tbl = fk.ptbl AND p.col = fk.pcol
 WHERE c.typ <> p.typ
UNION ALL SELECT 'K5', tbl || '.' || ccol, 'FK actions must be NO ACTION'
  FROM fk WHERE on_update <> 'NO ACTION' OR on_delete <> 'NO ACTION'
UNION ALL SELECT 'K6', tbl || ' -> ' || ptbl, 'FK columns are not the leading columns of any index'
  FROM (SELECT DISTINCT tbl, id, ptbl FROM fk) f
 WHERE NOT EXISTS (
        SELECT 1 FROM pragma_index_list(f.tbl) il
         WHERE il.partial = 0
           AND (SELECT COUNT(*) FROM fk x
                  JOIN pragma_index_info(il.name) ii ON ii.name = x.ccol AND ii.seqno = x.seq
                 WHERE x.tbl = f.tbl AND x.id = f.id)
             = (SELECT COUNT(*) FROM fk x WHERE x.tbl = f.tbl AND x.id = f.id))
UNION ALL SELECT 'U1', tbl || '.' || col, '_at/_date column must be INTEGER, is ' || typ
  FROM col WHERE (col LIKE '%\_at' ESCAPE '\' OR col LIKE '%\_date' ESCAPE '\') AND typ <> 'INTEGER'
UNION ALL SELECT CASE WHEN audit.c = 'i_active' THEN 'D2' ELSE 'D1' END, t.tbl, 'missing ' || audit.c
  FROM t CROSS JOIN audit
 WHERE NOT EXISTS (SELECT 1 FROM col WHERE col.tbl = t.tbl AND col.col = audit.c)
UNION ALL SELECT CASE WHEN audit.c = 'i_active' THEN 'D2' ELSE 'D1' END, col.tbl || '.' || col.col,
       'must be NOT NULL ' || audit.kind || ', is ' || col.typ || CASE WHEN col.nn = 1 THEN '' ELSE ' NULL' END
  FROM col JOIN audit ON audit.c = col.col
 WHERE col.nn = 0 OR col.typ <> audit.kind
UNION ALL SELECT 'D3', tbl || '.' || col, 'flag must be NOT NULL INTEGER (0/1), is ' || typ
  FROM col WHERE (col LIKE 'i\_is\_%' ESCAPE '\' OR col LIKE 'i\_has\_%' ESCAPE '\') AND (typ <> 'INTEGER' OR nn = 0)
UNION ALL SELECT 'D4', tbl || '.' || col, 'REAL/ANY not allowed; decimals are scaled INTEGER, is ' || typ
  FROM col WHERE typ IN ('REAL', 'ANY')
UNION ALL SELECT 'N6', t.tbl || '.' || il.name,
       CASE WHEN il.origin = 'u' THEN 'inline UNIQUE constraint; use CREATE UNIQUE INDEX uq_' || t.tbl || '_...'
            ELSE 'index name must start with ' || CASE WHEN il."unique" THEN 'uq_' ELSE 'ix_' END || t.tbl || '_' END
  FROM t, pragma_index_list(t.tbl) il
 WHERE il.origin <> 'pk'
   AND substr(il.name, 1, length(t.tbl) + 4) <> CASE WHEN il."unique" THEN 'uq_' ELSE 'ix_' END || t.tbl || '_'
ORDER BY 1, 2;
```
