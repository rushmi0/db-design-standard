# PostgreSQL Implementation

Requires PostgreSQL **12 or later**, and the database must be created with `ENCODING 'UTF8'`. This file maps the logical standard to PostgreSQL. It never changes a logical rule.

## 1. Type mapping

| Logical | PostgreSQL | Notes |
|---|---|---|
| `string(n)` | `VARCHAR(n)` | Unicode because the database is UTF8; `n` counts characters. Database default collation |
| `string(n) ascii` | `VARCHAR(n) COLLATE "C"` | Byte-order comparison and sorting (K2); prefix `LIKE` can use the index. Characters limited by CHECK |
| `int` | `INTEGER` | |
| `long` | `BIGINT` | |
| `boolean` | `BOOLEAN` | Literals `TRUE` / `FALSE` |
| `decimal(p,s)` | `NUMERIC(p,s)` | Never `REAL`, `DOUBLE PRECISION` or `MONEY` |
| `timestamp` | `BIGINT` | Unix seconds. Never the SQL `TIMESTAMP` / `TIMESTAMPTZ` types |

Do not use `SERIAL`, `BIGSERIAL` or `GENERATED … AS IDENTITY` for key columns (K1).

## 2. Names and quoting
- **Lowercase project:** write names unquoted. PostgreSQL folds unquoted names to lowercase, so `m_customer` is stored and matched as written.
- **Uppercase project:** PostgreSQL would fold `M_CUSTOMER` to `m_customer`. **Quote every identifier in DDL and in every query** (`"M_CUSTOMER"."I_CUSTOMER_ID"`) and configure the ORM to quote always.
- Index and constraint names must be unique in the schema; the `ix_<table>_…` patterns ensure this.
- The name limit is 63 bytes, which matches N6.
- Keyword list: Appendix C of the PostgreSQL documentation. Names built with the `m_`/`t_`/`i_` prefixes are never keywords.

## 3. Unix time (seconds)

| Task | Expression |
|---|---|
| Now (start of the transaction) | `FLOOR(EXTRACT(EPOCH FROM CURRENT_TIMESTAMP))::BIGINT` |
| Now (actual clock time) | `FLOOR(EXTRACT(EPOCH FROM clock_timestamp()))::BIGINT` |
| Unix → `timestamptz` | `to_timestamp(u)` |
| `timestamptz` → Unix | `FLOOR(EXTRACT(EPOCH FROM ts))::BIGINT` |
| `DATE` → `_date` value | `EXTRACT(EPOCH FROM d::timestamp)::BIGINT` (a `timestamp` without zone is read as UTC) |
| `_date` value → `DATE` | `(to_timestamp(u) AT TIME ZONE 'UTC')::date` |
| Start of a local business day | `EXTRACT(EPOCH FROM (d::timestamp AT TIME ZONE 'Asia/Bangkok'))::BIGINT` |

`EXTRACT(EPOCH …)` returns seconds with a fraction; `FLOOR` drops it, so values are always whole seconds. PostgreSQL uses IANA time-zone names such as `Asia/Bangkok`. For a local day, query with `i_x_at >= start AND i_x_at < start + 86400`.

## 4. CHECK patterns
Use regular expressions with `^…$` anchors, which reject spaces and wrong-case letters:

```sql
CHECK (i_customer_id ~ '^cus[0-9]{6}$')        -- fixed-format ID
CHECK (i_id_seq_id   ~ '^[a-z]{2,4}([0-9]{6}|[0-9]{8})?$')
CHECK (i_sku         ~ '^[A-Za-z0-9_-]+$')     -- business code
CHECK (i_due_date % 86400 = 0)                 -- calendar date
```

## 5. Example schema (compliant)

The data-access layer supplies the `func_nm` and `by` audit values on every insert; the timestamps also have database defaults as a safety net.

```sql
-- Counter for IDs (configuration -> master)
CREATE TABLE m_id_seq (
    i_id_seq_id        VARCHAR(20)  COLLATE "C" NOT NULL,
    i_last_val         BIGINT       NOT NULL,
    i_created_at       BIGINT       NOT NULL DEFAULT FLOOR(EXTRACT(EPOCH FROM CURRENT_TIMESTAMP))::BIGINT,
    i_created_func_nm  VARCHAR(100) COLLATE "C" NOT NULL,
    i_created_by       VARCHAR(50)  COLLATE "C" NOT NULL,
    i_updated_at       BIGINT       NOT NULL DEFAULT FLOOR(EXTRACT(EPOCH FROM CURRENT_TIMESTAMP))::BIGINT,
    i_updated_func_nm  VARCHAR(100) COLLATE "C" NOT NULL,
    i_updated_by       VARCHAR(50)  COLLATE "C" NOT NULL,
    i_active           BOOLEAN      NOT NULL DEFAULT TRUE,
    CONSTRAINT pk_m_id_seq PRIMARY KEY (i_id_seq_id),
    CONSTRAINT ck_m_id_seq_id_seq_id CHECK (i_id_seq_id ~ '^[a-z]{2,4}([0-9]{6}|[0-9]{8})?$'),
    CONSTRAINT ck_m_id_seq_last_val CHECK (i_last_val >= 0)
);

-- Master, ID without a date part: cus + 6 digits
CREATE TABLE m_customer (
    i_customer_id      VARCHAR(20)  COLLATE "C" NOT NULL,
    i_customer_nm      VARCHAR(200) NOT NULL,
    i_tax_no           VARCHAR(30)  COLLATE "C" NULL,  -- NULL = not registered for tax
    i_created_at       BIGINT       NOT NULL DEFAULT FLOOR(EXTRACT(EPOCH FROM CURRENT_TIMESTAMP))::BIGINT,
    i_created_func_nm  VARCHAR(100) COLLATE "C" NOT NULL,
    i_created_by       VARCHAR(50)  COLLATE "C" NOT NULL,
    i_updated_at       BIGINT       NOT NULL DEFAULT FLOOR(EXTRACT(EPOCH FROM CURRENT_TIMESTAMP))::BIGINT,
    i_updated_func_nm  VARCHAR(100) COLLATE "C" NOT NULL,
    i_updated_by       VARCHAR(50)  COLLATE "C" NOT NULL,
    i_active           BOOLEAN      NOT NULL DEFAULT TRUE,
    CONSTRAINT pk_m_customer PRIMARY KEY (i_customer_id),
    CONSTRAINT ck_m_customer_customer_id CHECK (i_customer_id ~ '^cus[0-9]{6}$'),
    CONSTRAINT ck_m_customer_tax_no CHECK (i_tax_no ~ '^[A-Za-z0-9_-]+$')
);

-- Master, monthly ID: prd + yyyymm + 4 digits; changeable business code in its own column
CREATE TABLE m_product (
    i_product_id       VARCHAR(20)  COLLATE "C" NOT NULL,
    i_sku              VARCHAR(30)  COLLATE "C" NOT NULL,
    i_product_nm       VARCHAR(200) NOT NULL,
    i_unit_price       NUMERIC(19,4) NOT NULL,
    i_is_taxable       BOOLEAN      NOT NULL DEFAULT TRUE,
    i_created_at       BIGINT       NOT NULL DEFAULT FLOOR(EXTRACT(EPOCH FROM CURRENT_TIMESTAMP))::BIGINT,
    i_created_func_nm  VARCHAR(100) COLLATE "C" NOT NULL,
    i_created_by       VARCHAR(50)  COLLATE "C" NOT NULL,
    i_updated_at       BIGINT       NOT NULL DEFAULT FLOOR(EXTRACT(EPOCH FROM CURRENT_TIMESTAMP))::BIGINT,
    i_updated_func_nm  VARCHAR(100) COLLATE "C" NOT NULL,
    i_updated_by       VARCHAR(50)  COLLATE "C" NOT NULL,
    i_active           BOOLEAN      NOT NULL DEFAULT TRUE,
    CONSTRAINT pk_m_product PRIMARY KEY (i_product_id),
    CONSTRAINT uq_m_product_sku UNIQUE (i_sku),   -- unique for all time (default)
    CONSTRAINT ck_m_product_product_id CHECK (i_product_id ~ '^prd[0-9]{10}$'),
    CONSTRAINT ck_m_product_sku CHECK (i_sku ~ '^[A-Za-z0-9_-]+$'),
    CONSTRAINT ck_m_product_unit_price CHECK (i_unit_price >= 0)
);

-- Master with times of day: wh + 4 digits
CREATE TABLE m_warehouse (
    i_warehouse_id     VARCHAR(20)  COLLATE "C" NOT NULL,
    i_warehouse_nm     VARCHAR(200) NOT NULL,
    i_open_time        INTEGER      NOT NULL,  -- seconds after local midnight
    i_close_time       INTEGER      NOT NULL,
    i_created_at       BIGINT       NOT NULL DEFAULT FLOOR(EXTRACT(EPOCH FROM CURRENT_TIMESTAMP))::BIGINT,
    i_created_func_nm  VARCHAR(100) COLLATE "C" NOT NULL,
    i_created_by       VARCHAR(50)  COLLATE "C" NOT NULL,
    i_updated_at       BIGINT       NOT NULL DEFAULT FLOOR(EXTRACT(EPOCH FROM CURRENT_TIMESTAMP))::BIGINT,
    i_updated_func_nm  VARCHAR(100) COLLATE "C" NOT NULL,
    i_updated_by       VARCHAR(50)  COLLATE "C" NOT NULL,
    i_active           BOOLEAN      NOT NULL DEFAULT TRUE,
    CONSTRAINT pk_m_warehouse PRIMARY KEY (i_warehouse_id),
    CONSTRAINT ck_m_warehouse_warehouse_id CHECK (i_warehouse_id ~ '^wh[0-9]{4}$'),
    CONSTRAINT ck_m_warehouse_open_time CHECK (i_open_time BETWEEN 0 AND 86399),
    CONSTRAINT ck_m_warehouse_close_time CHECK (i_close_time BETWEEN 0 AND 86399)
);

-- Header, daily ID: inv + yyyymmdd + 6 digits
CREATE TABLE t_invoice_h (
    i_invoice_id       VARCHAR(20)  COLLATE "C" NOT NULL,
    i_invoice_at       BIGINT       NOT NULL,
    i_due_date         BIGINT       NULL,  -- NULL = payable immediately
    i_customer_id      VARCHAR(20)  COLLATE "C" NOT NULL,
    i_status           VARCHAR(20)  COLLATE "C" NOT NULL DEFAULT 'draft',
    i_ccy              VARCHAR(3)   COLLATE "C" NOT NULL,
    i_total_amt        NUMERIC(19,4) NOT NULL DEFAULT 0,  -- stored total: sum of active lines
    i_note             VARCHAR(2000) NULL,               -- NULL = no note
    i_created_at       BIGINT       NOT NULL DEFAULT FLOOR(EXTRACT(EPOCH FROM CURRENT_TIMESTAMP))::BIGINT,
    i_created_func_nm  VARCHAR(100) COLLATE "C" NOT NULL,
    i_created_by       VARCHAR(50)  COLLATE "C" NOT NULL,
    i_updated_at       BIGINT       NOT NULL DEFAULT FLOOR(EXTRACT(EPOCH FROM CURRENT_TIMESTAMP))::BIGINT,
    i_updated_func_nm  VARCHAR(100) COLLATE "C" NOT NULL,
    i_updated_by       VARCHAR(50)  COLLATE "C" NOT NULL,
    i_active           BOOLEAN      NOT NULL DEFAULT TRUE,
    CONSTRAINT pk_t_invoice_h PRIMARY KEY (i_invoice_id),
    CONSTRAINT fk_t_invoice_h_customer FOREIGN KEY (i_customer_id)
        REFERENCES m_customer (i_customer_id) ON DELETE NO ACTION ON UPDATE NO ACTION,
    CONSTRAINT ck_t_invoice_h_invoice_id CHECK (i_invoice_id ~ '^inv[0-9]{14}$'),
    CONSTRAINT ck_t_invoice_h_due_date CHECK (i_due_date % 86400 = 0),
    CONSTRAINT ck_t_invoice_h_status CHECK (i_status IN ('draft', 'completed', 'cancelled')),
    CONSTRAINT ck_t_invoice_h_ccy CHECK (i_ccy ~ '^[A-Z]{3}$'),
    CONSTRAINT ck_t_invoice_h_total_amt CHECK (i_total_amt >= 0)
);
-- FK index; also serves "a customer's invoices by date"
CREATE INDEX ix_t_invoice_h_customer_id_invoice_at ON t_invoice_h (i_customer_id, i_invoice_at);

-- Detail: composite PK (header ID, i_internal_no)
CREATE TABLE t_invoice_d (
    i_invoice_id       VARCHAR(20)  COLLATE "C" NOT NULL,
    i_internal_no      VARCHAR(6)   COLLATE "C" NOT NULL,
    i_product_id       VARCHAR(20)  COLLATE "C" NOT NULL,
    i_desc             VARCHAR(500) NOT NULL,   -- snapshot of the product name
    i_qty              NUMERIC(19,4) NOT NULL,
    i_unit_price       NUMERIC(19,4) NOT NULL,  -- snapshot at sale time
    i_tax_rate         NUMERIC(9,6)  NOT NULL,  -- snapshot at sale time
    i_line_amt         NUMERIC(19,4) NOT NULL,
    i_created_at       BIGINT       NOT NULL DEFAULT FLOOR(EXTRACT(EPOCH FROM CURRENT_TIMESTAMP))::BIGINT,
    i_created_func_nm  VARCHAR(100) COLLATE "C" NOT NULL,
    i_created_by       VARCHAR(50)  COLLATE "C" NOT NULL,
    i_updated_at       BIGINT       NOT NULL DEFAULT FLOOR(EXTRACT(EPOCH FROM CURRENT_TIMESTAMP))::BIGINT,
    i_updated_func_nm  VARCHAR(100) COLLATE "C" NOT NULL,
    i_updated_by       VARCHAR(50)  COLLATE "C" NOT NULL,
    i_active           BOOLEAN      NOT NULL DEFAULT TRUE,
    CONSTRAINT pk_t_invoice_d PRIMARY KEY (i_invoice_id, i_internal_no),
    CONSTRAINT fk_t_invoice_d_invoice_h FOREIGN KEY (i_invoice_id)
        REFERENCES t_invoice_h (i_invoice_id) ON DELETE NO ACTION ON UPDATE NO ACTION,
    CONSTRAINT fk_t_invoice_d_product FOREIGN KEY (i_product_id)
        REFERENCES m_product (i_product_id) ON DELETE NO ACTION ON UPDATE NO ACTION,
    CONSTRAINT ck_t_invoice_d_internal_no CHECK (i_internal_no ~ '^[0-9]{6}$' AND i_internal_no <> '000000'),
    CONSTRAINT ck_t_invoice_d_qty CHECK (i_qty > 0),
    CONSTRAINT ck_t_invoice_d_unit_price CHECK (i_unit_price >= 0),
    CONSTRAINT ck_t_invoice_d_tax_rate CHECK (i_tax_rate BETWEEN 0 AND 1)
);
-- FK to the header is covered by the PK's leading column; FK to the product needs its own index
CREATE INDEX ix_t_invoice_d_product_id ON t_invoice_d (i_product_id);

-- Standalone transaction: one record per event, role FKs to the same master
CREATE TABLE t_stock_move (
    i_stock_move_id      VARCHAR(20)  COLLATE "C" NOT NULL,
    i_move_at            BIGINT       NOT NULL,
    i_product_id         VARCHAR(20)  COLLATE "C" NOT NULL,
    i_from_warehouse_id  VARCHAR(20)  COLLATE "C" NULL,  -- NULL = received from outside
    i_to_warehouse_id    VARCHAR(20)  COLLATE "C" NULL,  -- NULL = shipped out
    i_qty                NUMERIC(19,4) NOT NULL,
    i_created_at         BIGINT       NOT NULL DEFAULT FLOOR(EXTRACT(EPOCH FROM CURRENT_TIMESTAMP))::BIGINT,
    i_created_func_nm    VARCHAR(100) COLLATE "C" NOT NULL,
    i_created_by         VARCHAR(50)  COLLATE "C" NOT NULL,
    i_updated_at         BIGINT       NOT NULL DEFAULT FLOOR(EXTRACT(EPOCH FROM CURRENT_TIMESTAMP))::BIGINT,
    i_updated_func_nm    VARCHAR(100) COLLATE "C" NOT NULL,
    i_updated_by         VARCHAR(50)  COLLATE "C" NOT NULL,
    i_active             BOOLEAN      NOT NULL DEFAULT TRUE,
    CONSTRAINT pk_t_stock_move PRIMARY KEY (i_stock_move_id),
    CONSTRAINT fk_t_stock_move_product FOREIGN KEY (i_product_id)
        REFERENCES m_product (i_product_id) ON DELETE NO ACTION ON UPDATE NO ACTION,
    CONSTRAINT fk_t_stock_move_from_warehouse FOREIGN KEY (i_from_warehouse_id)
        REFERENCES m_warehouse (i_warehouse_id) ON DELETE NO ACTION ON UPDATE NO ACTION,
    CONSTRAINT fk_t_stock_move_to_warehouse FOREIGN KEY (i_to_warehouse_id)
        REFERENCES m_warehouse (i_warehouse_id) ON DELETE NO ACTION ON UPDATE NO ACTION,
    CONSTRAINT ck_t_stock_move_stock_move_id CHECK (i_stock_move_id ~ '^smv[0-9]{14}$'),
    CONSTRAINT ck_t_stock_move_qty CHECK (i_qty > 0),
    CONSTRAINT ck_t_stock_move_warehouse CHECK (
        (i_from_warehouse_id IS NOT NULL OR i_to_warehouse_id IS NOT NULL)
        AND (i_from_warehouse_id IS NULL OR i_to_warehouse_id IS NULL OR i_from_warehouse_id <> i_to_warehouse_id))
);
CREATE INDEX ix_t_stock_move_product_id_move_at ON t_stock_move (i_product_id, i_move_at);
CREATE INDEX ix_t_stock_move_from_warehouse_id ON t_stock_move (i_from_warehouse_id);
CREATE INDEX ix_t_stock_move_to_warehouse_id ON t_stock_move (i_to_warehouse_id);

-- Default read path: active rows only (D2)
CREATE VIEW v_m_product AS SELECT * FROM m_product WHERE i_active;
```

## 6. ID generation and documents

### ID from the counter table
A function always runs inside the caller's transaction, so the counter rolls back together with the row that uses the ID:

```sql
CREATE FUNCTION fn_next_id(p_abbr VARCHAR, p_date VARCHAR, p_digits INTEGER,
                           p_func_nm VARCHAR, p_by VARCHAR)
RETURNS VARCHAR
LANGUAGE plpgsql
AS $$
DECLARE
    v_seq_id VARCHAR(20) := p_abbr || p_date;   -- p_date: 'yyyymmdd', 'yyyymm' or ''
    v_now    BIGINT      := FLOOR(EXTRACT(EPOCH FROM clock_timestamp()))::BIGINT;
    v_n      BIGINT;
BEGIN
    -- Atomic insert-or-increment; the row lock is held until the caller commits
    INSERT INTO m_id_seq (i_id_seq_id, i_last_val,
                          i_created_at, i_created_func_nm, i_created_by,
                          i_updated_at, i_updated_func_nm, i_updated_by)
    VALUES (v_seq_id, 1, v_now, p_func_nm, p_by, v_now, p_func_nm, p_by)
    ON CONFLICT (i_id_seq_id) DO UPDATE
        SET i_last_val        = m_id_seq.i_last_val + 1,
            i_updated_at      = EXCLUDED.i_updated_at,
            i_updated_func_nm = EXCLUDED.i_updated_func_nm,
            i_updated_by      = EXCLUDED.i_updated_by
    RETURNING i_last_val INTO v_n;

    IF v_n >= power(10, p_digits) THEN
        RAISE EXCEPTION 'ID counter exhausted for %', v_seq_id;
    END IF;

    RETURN v_seq_id || LPAD(v_n::TEXT, p_digits, '0');
END;
$$;
```

### Master rows
```sql
BEGIN;
INSERT INTO m_customer (i_customer_id, i_customer_nm,
                        i_created_func_nm, i_created_by, i_updated_func_nm, i_updated_by)
VALUES (fn_next_id('cus', '', 6, 'customer.create', 'usr000001'), 'บริษัท ตัวอย่าง จำกัด',
        'customer.create', 'usr000001', 'customer.create', 'usr000001');

INSERT INTO m_product (i_product_id, i_sku, i_product_nm, i_unit_price,
                       i_created_func_nm, i_created_by, i_updated_func_nm, i_updated_by)
VALUES (fn_next_id('prd', to_char(CURRENT_TIMESTAMP AT TIME ZONE 'Asia/Bangkok', 'YYYYMM'), 4,
                   'product.create', 'usr000001'),
        'SKU-001', '商品 A', 100, 'product.create', 'usr000001', 'product.create', 'usr000001');

INSERT INTO m_warehouse (i_warehouse_id, i_warehouse_nm, i_open_time, i_close_time,
                         i_created_func_nm, i_created_by, i_updated_func_nm, i_updated_by)
VALUES (fn_next_id('wh', '', 4, 'warehouse.create', 'usr000001'), '東京倉庫', 30600, 64800,
        'warehouse.create', 'usr000001', 'warehouse.create', 'usr000001');
COMMIT;
```

`LPAD` would cut off digits if a number were too long; the exhaustion check raises an error first, so that never happens.

**Optional sequence** for an ID without a date part (gaps after a rollback are acceptable):

```sql
CREATE SEQUENCE sq_m_customer AS INTEGER MINVALUE 1 MAXVALUE 999999 NO CYCLE;
-- 'cus' || LPAD(nextval('sq_m_customer')::TEXT, 6, '0')
```

### Creating a Header/Detail document
The application normally runs these statements in one transaction and keeps the ID in a variable. The `DO` block shows the same steps in plain SQL:

```sql
DO $$
DECLARE
    v_func VARCHAR(100) := 'invoice.create';
    v_by   VARCHAR(50)  := 'usr000001';
    v_id   VARCHAR(20)  := fn_next_id('inv', to_char(CURRENT_TIMESTAMP AT TIME ZONE 'Asia/Bangkok', 'YYYYMMDD'),
                                      6, 'invoice.create', 'usr000001');
    v_now  BIGINT       := FLOOR(EXTRACT(EPOCH FROM CURRENT_TIMESTAMP))::BIGINT;
BEGIN
    INSERT INTO t_invoice_h (i_invoice_id, i_invoice_at, i_customer_id, i_ccy,
                             i_created_at, i_created_func_nm, i_created_by,
                             i_updated_at, i_updated_func_nm, i_updated_by)
    SELECT v_id, v_now, c.i_customer_id, 'THB', v_now, v_func, v_by, v_now, v_func, v_by
      FROM m_customer c
     WHERE c.i_customer_id = 'cus000001' AND c.i_active;   -- never reference an inactive parent

    -- Lines are numbered 000001, 000002, ...; price, tax and name are copied (H3)
    INSERT INTO t_invoice_d (i_invoice_id, i_internal_no, i_product_id, i_desc,
                             i_qty, i_unit_price, i_tax_rate, i_line_amt,
                             i_created_at, i_created_func_nm, i_created_by,
                             i_updated_at, i_updated_func_nm, i_updated_by)
    SELECT v_id, '000001', p.i_product_id, p.i_product_nm,
           2, p.i_unit_price, 0.07, 2 * p.i_unit_price,
           v_now, v_func, v_by, v_now, v_func, v_by
      FROM v_m_product p
     ORDER BY p.i_product_id
     LIMIT 1;

    -- Recalculate the stored total from active lines, in the same transaction
    UPDATE t_invoice_h
       SET i_total_amt = COALESCE((SELECT SUM(i_line_amt) FROM t_invoice_d
                                    WHERE i_invoice_id = v_id AND i_active), 0),
           i_updated_at = v_now, i_updated_func_nm = v_func, i_updated_by = v_by
     WHERE i_invoice_id = v_id;
END;
$$;
```

### Adding a line to an existing document
```sql
BEGIN;

-- 1. Lock the header first (fixed lock order: header, then details)
SELECT i_invoice_id
  FROM t_invoice_h
 WHERE i_invoice_id = (SELECT MIN(i_invoice_id) FROM t_invoice_h)
   AND i_status = 'draft' AND i_active
   FOR UPDATE;
-- The application stops if no row is returned.

-- 2. Next line number = highest + 1, counting inactive lines too (numbers are never reused)
INSERT INTO t_invoice_d (i_invoice_id, i_internal_no, i_product_id, i_desc,
                         i_qty, i_unit_price, i_tax_rate, i_line_amt,
                         i_created_func_nm, i_created_by, i_updated_func_nm, i_updated_by)
SELECT h.i_invoice_id,
       CASE WHEN COALESCE(MAX(d.i_internal_no)::INTEGER, 0) < 999999
            THEN LPAD((COALESCE(MAX(d.i_internal_no)::INTEGER, 0) + 1)::TEXT, 6, '0') END,
       (SELECT MIN(i_product_id) FROM v_m_product), 'Product B', 1, 150, 0.07, 150,
       'invoice.add_line', 'usr000001', 'invoice.add_line', 'usr000001'
  FROM t_invoice_h h
  LEFT JOIN t_invoice_d d ON d.i_invoice_id = h.i_invoice_id
 WHERE h.i_invoice_id = (SELECT MIN(i_invoice_id) FROM t_invoice_h)
 GROUP BY h.i_invoice_id;
-- LPAD cuts a longer number down to 6 digits, so the CASE returns NULL at the limit
-- and NOT NULL stops the insert instead of reusing a number.

COMMIT;
```

### Removing a line (soft delete)
```sql
UPDATE t_invoice_d
   SET i_active = FALSE,
       i_updated_at = FLOOR(EXTRACT(EPOCH FROM CURRENT_TIMESTAMP))::BIGINT,
       i_updated_func_nm = 'invoice.remove_line', i_updated_by = 'usr000001'
 WHERE i_invoice_id = (SELECT MIN(i_invoice_id) FROM t_invoice_h) AND i_internal_no = '000002';
```
Recalculate the header total in the same transaction, as above.

## 7. Unique codes among active rows only
To let a code be reused after its row is deactivated, replace the plain unique constraint with a partial unique index:

```sql
ALTER TABLE m_product DROP CONSTRAINT uq_m_product_sku;
CREATE UNIQUE INDEX uq_m_product_sku ON m_product (i_sku) WHERE i_active;
```

## 8. Performance notes
- **Indexes are not created for FKs automatically.** Create the `ix_` indexes yourself (K6).
- **`COLLATE "C"` on ASCII strings** makes comparison and sorting byte-based and fast, and prefix `LIKE` searches can use the index.
- **Range queries on `_at`** stay integer comparisons. Never wrap the column in `to_timestamp()` inside a `WHERE`.
- **Inactive rows.** Where most rows are inactive, a partial index `… WHERE i_active` keeps the index small for normal queries.
- **Table rows are not clustered** in PostgreSQL. Sequential IDs still keep the B-tree index compact.

## 9. Catalog audit query

This query lists violations of the rules that can be checked from the catalog: N1, N2, N4, N5, N6, K1, K2, K3, K5, K6, U1, D1, D2, D3 and D4. Run it in the schema to check (`SET search_path = <schema>` first). An empty result means every automated check passed; the remaining rules need the manual review in `../schema-validation.md`. The name pattern is for a lowercase project; in an uppercase project replace `a-z` with `A-Z` and the prefixes with their uppercase form.

```sql
WITH t AS (
    SELECT c.oid, c.relname::TEXT AS tbl
      FROM pg_class c JOIN pg_namespace n ON n.oid = c.relnamespace
     WHERE c.relkind IN ('r', 'p') AND n.nspname = current_schema()
), col AS (
    SELECT t.oid, t.tbl, a.attname::TEXT AS col, a.atttypid, a.attcollation, a.attnotnull,
           format_type(a.atttypid, a.atttypmod) AS typ,
           EXISTS (SELECT 1 FROM pg_constraint k
                    WHERE k.conrelid = t.oid AND k.contype IN ('p', 'f') AND a.attnum = ANY (k.conkey)) AS is_key
      FROM t JOIN pg_attribute a ON a.attrelid = t.oid AND a.attnum > 0 AND NOT a.attisdropped
), fk AS (
    SELECT k.*, t.tbl FROM pg_constraint k JOIN t ON t.oid = k.conrelid WHERE k.contype = 'f'
), audit(c, kind) AS (
    VALUES ('i_created_at', 't'), ('i_created_func_nm', 's'), ('i_created_by', 's'),
           ('i_updated_at', 't'), ('i_updated_func_nm', 's'), ('i_updated_by', 's'), ('i_active', 'b')
)
SELECT 'N1' AS rule_id, tbl AS object_name, 'name must be a-z0-9_, start with a letter, no __ or trailing _' AS problem
  FROM t WHERE tbl !~ '^[a-z][a-z0-9]*(_[a-z0-9]+)*$'
UNION ALL SELECT 'N2', tbl, 'table must start with m_ or t_' FROM t WHERE left(tbl, 2) NOT IN ('m_', 't_')
UNION ALL SELECT 'N5', tbl, 'table name longer than 30' FROM t WHERE length(tbl) > 30
UNION ALL SELECT 'N1', tbl || '.' || col, 'name must be a-z0-9_, start with a letter, no __ or trailing _'
  FROM col WHERE col !~ '^[a-z][a-z0-9]*(_[a-z0-9]+)*$'
UNION ALL SELECT 'N4', tbl || '.' || col, 'column must start with i_' FROM col WHERE left(col, 2) <> 'i_'
UNION ALL SELECT 'N5', tbl || '.' || col, 'column name longer than 30' FROM col WHERE length(col) > 30
UNION ALL SELECT 'K1', tbl || '.' || col, 'key column must be VARCHAR, is ' || typ
  FROM col WHERE is_key AND atttypid <> 'varchar'::regtype
UNION ALL SELECT 'K2', tbl || '.' || col, 'key column must use COLLATE "C"'
  FROM col WHERE is_key AND atttypid = 'varchar'::regtype
   AND attcollation <> (SELECT oid FROM pg_collation WHERE collname = 'C' AND collnamespace = 'pg_catalog'::regnamespace)
UNION ALL SELECT 'K3', fk.conname, 'FK column ' || ca.attname || ' differs from ' || pa.attname || ' in type, length or collation'
  FROM fk CROSS JOIN LATERAL unnest(fk.conkey, fk.confkey) AS u(ck, pk)
  JOIN pg_attribute ca ON ca.attrelid = fk.conrelid  AND ca.attnum = u.ck
  JOIN pg_attribute pa ON pa.attrelid = fk.confrelid AND pa.attnum = u.pk
 WHERE ca.atttypid <> pa.atttypid OR ca.atttypmod <> pa.atttypmod OR ca.attcollation <> pa.attcollation
UNION ALL SELECT 'K5', conname, 'FK actions must be NO ACTION' FROM fk WHERE confdeltype <> 'a' OR confupdtype <> 'a'
UNION ALL SELECT 'K6', conname, 'FK columns are not the leading columns of any index'
  FROM fk WHERE NOT EXISTS (
        SELECT 1 FROM pg_index i
         WHERE i.indrelid = fk.conrelid AND i.indpred IS NULL
           AND (i.indkey::int2[])[0:cardinality(fk.conkey) - 1] @> fk.conkey
           AND (i.indkey::int2[])[0:cardinality(fk.conkey) - 1] <@ fk.conkey)
UNION ALL SELECT 'U1', tbl || '.' || col, 'date/time type not allowed: ' || typ
  FROM col WHERE atttypid IN ('date'::regtype, 'timestamp'::regtype, 'timestamptz'::regtype,
                              'time'::regtype, 'timetz'::regtype, 'interval'::regtype)
UNION ALL SELECT 'U1', tbl || '.' || col, '_at/_date column must be BIGINT, is ' || typ
  FROM col WHERE (col LIKE '%\_at' OR col LIKE '%\_date') AND atttypid <> 'int8'::regtype
UNION ALL SELECT CASE WHEN audit.c = 'i_active' THEN 'D2' ELSE 'D1' END, t.tbl, 'missing ' || audit.c
  FROM t CROSS JOIN audit
 WHERE NOT EXISTS (SELECT 1 FROM col WHERE col.oid = t.oid AND col.col = audit.c)
UNION ALL SELECT CASE WHEN audit.kind = 'b' THEN 'D2' ELSE 'D1' END, col.tbl || '.' || col.col,
       'must be NOT NULL ' || CASE audit.kind WHEN 't' THEN 'BIGINT' WHEN 'b' THEN 'BOOLEAN' ELSE 'VARCHAR' END || ', is ' || col.typ
       || CASE WHEN col.attnotnull THEN '' ELSE ' NULL' END
  FROM col JOIN audit ON audit.c = col.col
 WHERE NOT col.attnotnull
    OR col.atttypid <> CASE audit.kind WHEN 't' THEN 'int8'::regtype WHEN 'b' THEN 'bool'::regtype ELSE 'varchar'::regtype END
UNION ALL SELECT 'D3', tbl || '.' || col, 'flag must be NOT NULL BOOLEAN, is ' || typ || CASE WHEN attnotnull THEN '' ELSE ' NULL' END
  FROM col WHERE (col LIKE 'i\_is\_%' OR col LIKE 'i\_has\_%') AND (atttypid <> 'bool'::regtype OR NOT attnotnull)
UNION ALL SELECT 'D4', tbl || '.' || col, 'floating-point or money type: ' || typ
  FROM col WHERE atttypid IN ('real'::regtype, 'double precision'::regtype, 'money'::regtype)
UNION ALL SELECT 'N6', k.conname, 'constraint name must start with '
       || CASE k.contype WHEN 'p' THEN 'pk_' WHEN 'f' THEN 'fk_' WHEN 'u' THEN 'uq_' ELSE 'ck_' END || t.tbl
  FROM pg_constraint k JOIN t ON t.oid = k.conrelid
 WHERE k.contype IN ('p', 'f', 'u', 'c')
   AND NOT (CASE k.contype WHEN 'p' THEN k.conname = 'pk_' || t.tbl
                           ELSE left(k.conname, length(t.tbl) + 4)
                                = CASE k.contype WHEN 'f' THEN 'fk_' WHEN 'u' THEN 'uq_' ELSE 'ck_' END || t.tbl || '_' END)
UNION ALL SELECT 'N6', ic.relname, 'index name must start with ix_' || t.tbl || '_ (or uq_ when unique)'
  FROM pg_index i JOIN pg_class ic ON ic.oid = i.indexrelid JOIN t ON t.oid = i.indrelid
 WHERE NOT EXISTS (SELECT 1 FROM pg_constraint k WHERE k.conindid = i.indexrelid AND k.contype IN ('p', 'u'))
   AND left(ic.relname, length(t.tbl) + 4) <> (CASE WHEN i.indisunique THEN 'uq_' ELSE 'ix_' END) || t.tbl || '_'
UNION ALL SELECT 'N6', conname, 'object name longer than 63' FROM pg_constraint k JOIN t ON t.oid = k.conrelid WHERE length(conname) > 63
ORDER BY 1, 2;
```
