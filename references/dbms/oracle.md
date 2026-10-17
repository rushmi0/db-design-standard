# Oracle Database Implementation

Requires **Oracle Database 19c or later** (128-byte names, `VARCHAR2(n CHAR)`), with database character set **`AL32UTF8`**, the default since 12.2. Check with `SELECT value FROM nls_database_parameters WHERE parameter = 'NLS_CHARACTERSET'`. This file maps the logical standard to Oracle. It never changes a logical rule.

## 1. Type mapping

| Logical | Oracle | Notes |
|---|---|---|
| `string(n)` | `VARCHAR2(n CHAR)` | Unicode in an `AL32UTF8` database; `CHAR` makes `n` count characters. For `n` > 1000, see below |
| `string(n) ascii` | `VARCHAR2(n BYTE)` | Compared in binary under the default `NLS_COMP = BINARY` (K2); never switch the database or sessions to `LINGUISTIC` |
| `int` | `NUMBER(10)` | |
| `long` | `NUMBER(19)` | |
| `boolean` | `NUMBER(1)` + CHECK `IN (0,1)` | `1` = true. On 23ai or later, `BOOLEAN` may be used instead; pick one form per database |
| `decimal(p,s)` | `NUMBER(p,s)` | Never `FLOAT`, `BINARY_FLOAT`, `BINARY_DOUBLE` or `NUMBER` without precision |
| `timestamp` | `NUMBER(19)` | Unix seconds. Never `DATE`, `TIMESTAMP` or `INTERVAL` |

- **Long text.** `VARCHAR2` holds at most 4000 bytes, about 1000 characters of 4-byte UTF-8 (1333 for Thai). For `string(n)` with `n` > 1000 use `VARCHAR2(n CHAR)` when the database has `MAX_STRING_SIZE = EXTENDED` (32767 bytes); otherwise use `CLOB` with CHECK `LENGTH(col) <= n`. The example below uses `CLOB` for the 2000-character note.
- **Empty strings are NULL.** Oracle stores `''` as NULL. D5 already forbids empty strings; a NOT NULL text column rejects `''` with an error.
- Never use `GENERATED … AS IDENTITY` (K1).

## 2. Names and quoting
- **Never quote names.** Oracle stores unquoted names in uppercase and matches them without regard to case, so `m_customer` in the DDL is `M_CUSTOMER` in the catalog and both spellings work in queries. This is the same name in either project case. A quoted lowercase name (`"m_customer"`) becomes case-sensitive and must then be quoted everywhere, so don't.
- Catalog views (`user_tables`, `user_tab_columns`, …) show names in uppercase; compare with uppercase literals there.
- Name limit: 128 bytes. Constraint and index names must be unique in the schema; the `pk_<table>` / `ix_<table>_…` patterns ensure this.
- Reserved words: `SELECT keyword FROM v$reserved_words WHERE reserved = 'Y'`. Names with the `m_`/`t_`/`i_` prefixes never collide.
- **FK actions.** Oracle's default (and only update) behaviour is NO ACTION, and it rejects `ON DELETE NO ACTION` and `ON UPDATE …` clauses. Declare FKs **without** any `ON DELETE`/`ON UPDATE` clause; that is K5-compliant. Never add `ON DELETE CASCADE` or `ON DELETE SET NULL`.

## 3. Unix time (seconds)

| Task | Expression |
|---|---|
| Now (rounded down) | `ROUND((TRUNC(SYS_EXTRACT_UTC(SYSTIMESTAMP), 'MI') - DATE '1970-01-01') * 86400) + FLOOR(EXTRACT(SECOND FROM SYS_EXTRACT_UTC(SYSTIMESTAMP)))` |
| Unix → UTC `TIMESTAMP` | `TIMESTAMP '1970-01-01 00:00:00' + NUMTODSINTERVAL(u, 'SECOND')` |
| UTC `DATE` → Unix | `ROUND((d - DATE '1970-01-01') * 86400)` |
| `DATE` → `_date` value | `ROUND((TRUNC(d) - DATE '1970-01-01') * 86400)` |
| `_date` value → `DATE` | `DATE '1970-01-01' + u / 86400` |
| Start of a local business day | `ROUND((CAST(FROM_TZ(CAST(DATE '2026-10-17' AS TIMESTAMP), 'Asia/Bangkok') AT TIME ZONE 'UTC' AS DATE) - DATE '1970-01-01') * 86400)` |
| Business date for an ID | `TO_CHAR(SYSTIMESTAMP AT TIME ZONE 'Asia/Bangkok', 'YYYYMMDD')` |

- Subtracting two `DATE`s gives days as a decimal fraction, so multiplying by 86400 can leave a tiny remainder; `ROUND` restores the exact whole second. The "now" expression truncates to the minute, then adds the whole seconds separately, so fractional seconds are always dropped rather than rounded.
- `SYSTIMESTAMP` returns the same value everywhere within one SQL statement, so the two uses in the "now" expression agree.
- Oracle has no `%` operator: the `_date` CHECK is `MOD(col, 86400) = 0`.
- Oracle uses IANA zone names (`Asia/Bangkok`). For a local business day, query with `i_x_at >= :start AND i_x_at < :start + 86400`.

## 4. CHECK patterns
Pass the `'c'` match parameter so the regular expression is case-sensitive whatever the session's `NLS_SORT` is:

```sql
CHECK (REGEXP_LIKE(i_customer_id, '^cus[0-9]{6}$', 'c'))     -- fixed-format ID
CHECK (REGEXP_LIKE(i_sku, '^[A-Za-z0-9_-]+$', 'c'))          -- business code
CHECK (i_active IN (0, 1))                                    -- boolean
CHECK (MOD(i_due_date, 86400) = 0)                            -- calendar date
```

## 5. Example schema (compliant)

The data-access layer supplies the `func_nm` and `by` audit values on every insert; the timestamps also have database defaults as a safety net.

```sql
-- Counter for IDs (configuration -> master)
CREATE TABLE m_id_seq (
    i_id_seq_id        VARCHAR2(20 BYTE)  NOT NULL,
    i_last_val         NUMBER(19)         NOT NULL,
    i_created_at       NUMBER(19) DEFAULT ROUND((TRUNC(SYS_EXTRACT_UTC(SYSTIMESTAMP), 'MI') - DATE '1970-01-01') * 86400) + FLOOR(EXTRACT(SECOND FROM SYS_EXTRACT_UTC(SYSTIMESTAMP))) NOT NULL,
    i_created_func_nm  VARCHAR2(100 BYTE) NOT NULL,
    i_created_by       VARCHAR2(50 BYTE)  NOT NULL,
    i_updated_at       NUMBER(19) DEFAULT ROUND((TRUNC(SYS_EXTRACT_UTC(SYSTIMESTAMP), 'MI') - DATE '1970-01-01') * 86400) + FLOOR(EXTRACT(SECOND FROM SYS_EXTRACT_UTC(SYSTIMESTAMP))) NOT NULL,
    i_updated_func_nm  VARCHAR2(100 BYTE) NOT NULL,
    i_updated_by       VARCHAR2(50 BYTE)  NOT NULL,
    i_active           NUMBER(1) DEFAULT 1 NOT NULL,
    CONSTRAINT pk_m_id_seq PRIMARY KEY (i_id_seq_id),
    CONSTRAINT ck_m_id_seq_id_seq_id CHECK (REGEXP_LIKE(i_id_seq_id, '^[a-z]{2,4}([0-9]{6}|[0-9]{8})?$', 'c')),
    CONSTRAINT ck_m_id_seq_last_val CHECK (i_last_val >= 0),
    CONSTRAINT ck_m_id_seq_active CHECK (i_active IN (0, 1))
);

-- Master, ID without a date part: cus + 6 digits
CREATE TABLE m_customer (
    i_customer_id      VARCHAR2(20 BYTE)  NOT NULL,
    i_customer_nm      VARCHAR2(200 CHAR) NOT NULL,
    i_tax_no           VARCHAR2(30 BYTE)  NULL,  -- NULL = not registered for tax
    i_created_at       NUMBER(19) DEFAULT ROUND((TRUNC(SYS_EXTRACT_UTC(SYSTIMESTAMP), 'MI') - DATE '1970-01-01') * 86400) + FLOOR(EXTRACT(SECOND FROM SYS_EXTRACT_UTC(SYSTIMESTAMP))) NOT NULL,
    i_created_func_nm  VARCHAR2(100 BYTE) NOT NULL,
    i_created_by       VARCHAR2(50 BYTE)  NOT NULL,
    i_updated_at       NUMBER(19) DEFAULT ROUND((TRUNC(SYS_EXTRACT_UTC(SYSTIMESTAMP), 'MI') - DATE '1970-01-01') * 86400) + FLOOR(EXTRACT(SECOND FROM SYS_EXTRACT_UTC(SYSTIMESTAMP))) NOT NULL,
    i_updated_func_nm  VARCHAR2(100 BYTE) NOT NULL,
    i_updated_by       VARCHAR2(50 BYTE)  NOT NULL,
    i_active           NUMBER(1) DEFAULT 1 NOT NULL,
    CONSTRAINT pk_m_customer PRIMARY KEY (i_customer_id),
    CONSTRAINT ck_m_customer_customer_id CHECK (REGEXP_LIKE(i_customer_id, '^cus[0-9]{6}$', 'c')),
    CONSTRAINT ck_m_customer_tax_no CHECK (REGEXP_LIKE(i_tax_no, '^[A-Za-z0-9_-]+$', 'c')),
    CONSTRAINT ck_m_customer_active CHECK (i_active IN (0, 1))
);

-- Master, monthly ID: prd + yyyymm + 4 digits; changeable business code in its own column
CREATE TABLE m_product (
    i_product_id       VARCHAR2(20 BYTE)  NOT NULL,
    i_sku              VARCHAR2(30 BYTE)  NOT NULL,
    i_product_nm       VARCHAR2(200 CHAR) NOT NULL,
    i_unit_price       NUMBER(19,4)       NOT NULL,
    i_is_taxable       NUMBER(1) DEFAULT 1 NOT NULL,
    i_created_at       NUMBER(19) DEFAULT ROUND((TRUNC(SYS_EXTRACT_UTC(SYSTIMESTAMP), 'MI') - DATE '1970-01-01') * 86400) + FLOOR(EXTRACT(SECOND FROM SYS_EXTRACT_UTC(SYSTIMESTAMP))) NOT NULL,
    i_created_func_nm  VARCHAR2(100 BYTE) NOT NULL,
    i_created_by       VARCHAR2(50 BYTE)  NOT NULL,
    i_updated_at       NUMBER(19) DEFAULT ROUND((TRUNC(SYS_EXTRACT_UTC(SYSTIMESTAMP), 'MI') - DATE '1970-01-01') * 86400) + FLOOR(EXTRACT(SECOND FROM SYS_EXTRACT_UTC(SYSTIMESTAMP))) NOT NULL,
    i_updated_func_nm  VARCHAR2(100 BYTE) NOT NULL,
    i_updated_by       VARCHAR2(50 BYTE)  NOT NULL,
    i_active           NUMBER(1) DEFAULT 1 NOT NULL,
    CONSTRAINT pk_m_product PRIMARY KEY (i_product_id),
    CONSTRAINT uq_m_product_sku UNIQUE (i_sku),   -- unique for all time (default)
    CONSTRAINT ck_m_product_product_id CHECK (REGEXP_LIKE(i_product_id, '^prd[0-9]{10}$', 'c')),
    CONSTRAINT ck_m_product_sku CHECK (REGEXP_LIKE(i_sku, '^[A-Za-z0-9_-]+$', 'c')),
    CONSTRAINT ck_m_product_unit_price CHECK (i_unit_price >= 0),
    CONSTRAINT ck_m_product_is_taxable CHECK (i_is_taxable IN (0, 1)),
    CONSTRAINT ck_m_product_active CHECK (i_active IN (0, 1))
);

-- Master with times of day: wh + 4 digits
CREATE TABLE m_warehouse (
    i_warehouse_id     VARCHAR2(20 BYTE)  NOT NULL,
    i_warehouse_nm     VARCHAR2(200 CHAR) NOT NULL,
    i_open_time        NUMBER(10)         NOT NULL,  -- seconds after local midnight
    i_close_time       NUMBER(10)         NOT NULL,
    i_created_at       NUMBER(19) DEFAULT ROUND((TRUNC(SYS_EXTRACT_UTC(SYSTIMESTAMP), 'MI') - DATE '1970-01-01') * 86400) + FLOOR(EXTRACT(SECOND FROM SYS_EXTRACT_UTC(SYSTIMESTAMP))) NOT NULL,
    i_created_func_nm  VARCHAR2(100 BYTE) NOT NULL,
    i_created_by       VARCHAR2(50 BYTE)  NOT NULL,
    i_updated_at       NUMBER(19) DEFAULT ROUND((TRUNC(SYS_EXTRACT_UTC(SYSTIMESTAMP), 'MI') - DATE '1970-01-01') * 86400) + FLOOR(EXTRACT(SECOND FROM SYS_EXTRACT_UTC(SYSTIMESTAMP))) NOT NULL,
    i_updated_func_nm  VARCHAR2(100 BYTE) NOT NULL,
    i_updated_by       VARCHAR2(50 BYTE)  NOT NULL,
    i_active           NUMBER(1) DEFAULT 1 NOT NULL,
    CONSTRAINT pk_m_warehouse PRIMARY KEY (i_warehouse_id),
    CONSTRAINT ck_m_warehouse_warehouse_id CHECK (REGEXP_LIKE(i_warehouse_id, '^wh[0-9]{4}$', 'c')),
    CONSTRAINT ck_m_warehouse_open_time CHECK (i_open_time BETWEEN 0 AND 86399),
    CONSTRAINT ck_m_warehouse_close_time CHECK (i_close_time BETWEEN 0 AND 86399),
    CONSTRAINT ck_m_warehouse_active CHECK (i_active IN (0, 1))
);

-- Header, daily ID: inv + yyyymmdd + 6 digits
CREATE TABLE t_invoice_h (
    i_invoice_id       VARCHAR2(20 BYTE)  NOT NULL,
    i_invoice_at       NUMBER(19)         NOT NULL,
    i_due_date         NUMBER(19)         NULL,  -- NULL = payable immediately
    i_customer_id      VARCHAR2(20 BYTE)  NOT NULL,
    i_status           VARCHAR2(20 BYTE) DEFAULT 'draft' NOT NULL,
    i_ccy              VARCHAR2(3 BYTE)   NOT NULL,
    i_total_amt        NUMBER(19,4) DEFAULT 0 NOT NULL,  -- stored total: sum of active lines
    i_note             CLOB               NULL,  -- string(2000); NULL = no note
    i_created_at       NUMBER(19) DEFAULT ROUND((TRUNC(SYS_EXTRACT_UTC(SYSTIMESTAMP), 'MI') - DATE '1970-01-01') * 86400) + FLOOR(EXTRACT(SECOND FROM SYS_EXTRACT_UTC(SYSTIMESTAMP))) NOT NULL,
    i_created_func_nm  VARCHAR2(100 BYTE) NOT NULL,
    i_created_by       VARCHAR2(50 BYTE)  NOT NULL,
    i_updated_at       NUMBER(19) DEFAULT ROUND((TRUNC(SYS_EXTRACT_UTC(SYSTIMESTAMP), 'MI') - DATE '1970-01-01') * 86400) + FLOOR(EXTRACT(SECOND FROM SYS_EXTRACT_UTC(SYSTIMESTAMP))) NOT NULL,
    i_updated_func_nm  VARCHAR2(100 BYTE) NOT NULL,
    i_updated_by       VARCHAR2(50 BYTE)  NOT NULL,
    i_active           NUMBER(1) DEFAULT 1 NOT NULL,
    CONSTRAINT pk_t_invoice_h PRIMARY KEY (i_invoice_id),
    CONSTRAINT fk_t_invoice_h_customer FOREIGN KEY (i_customer_id) REFERENCES m_customer (i_customer_id),
    CONSTRAINT ck_t_invoice_h_invoice_id CHECK (REGEXP_LIKE(i_invoice_id, '^inv[0-9]{14}$', 'c')),
    CONSTRAINT ck_t_invoice_h_due_date CHECK (MOD(i_due_date, 86400) = 0),
    CONSTRAINT ck_t_invoice_h_status CHECK (i_status IN ('draft', 'completed', 'cancelled')),
    CONSTRAINT ck_t_invoice_h_ccy CHECK (REGEXP_LIKE(i_ccy, '^[A-Z]{3}$', 'c')),
    CONSTRAINT ck_t_invoice_h_total_amt CHECK (i_total_amt >= 0),
    CONSTRAINT ck_t_invoice_h_note CHECK (LENGTH(i_note) <= 2000),
    CONSTRAINT ck_t_invoice_h_active CHECK (i_active IN (0, 1))
);
-- FK index; also serves "a customer's invoices by date"
CREATE INDEX ix_t_invoice_h_customer_id_invoice_at ON t_invoice_h (i_customer_id, i_invoice_at);

-- Detail: composite PK (header ID, i_internal_no)
CREATE TABLE t_invoice_d (
    i_invoice_id       VARCHAR2(20 BYTE)  NOT NULL,
    i_internal_no      VARCHAR2(6 BYTE)   NOT NULL,
    i_product_id       VARCHAR2(20 BYTE)  NOT NULL,
    i_desc             VARCHAR2(500 CHAR) NOT NULL,  -- snapshot of the product name
    i_qty              NUMBER(19,4)       NOT NULL,
    i_unit_price       NUMBER(19,4)       NOT NULL,  -- snapshot at sale time
    i_tax_rate         NUMBER(9,6)        NOT NULL,  -- snapshot at sale time
    i_line_amt         NUMBER(19,4)       NOT NULL,
    i_created_at       NUMBER(19) DEFAULT ROUND((TRUNC(SYS_EXTRACT_UTC(SYSTIMESTAMP), 'MI') - DATE '1970-01-01') * 86400) + FLOOR(EXTRACT(SECOND FROM SYS_EXTRACT_UTC(SYSTIMESTAMP))) NOT NULL,
    i_created_func_nm  VARCHAR2(100 BYTE) NOT NULL,
    i_created_by       VARCHAR2(50 BYTE)  NOT NULL,
    i_updated_at       NUMBER(19) DEFAULT ROUND((TRUNC(SYS_EXTRACT_UTC(SYSTIMESTAMP), 'MI') - DATE '1970-01-01') * 86400) + FLOOR(EXTRACT(SECOND FROM SYS_EXTRACT_UTC(SYSTIMESTAMP))) NOT NULL,
    i_updated_func_nm  VARCHAR2(100 BYTE) NOT NULL,
    i_updated_by       VARCHAR2(50 BYTE)  NOT NULL,
    i_active           NUMBER(1) DEFAULT 1 NOT NULL,
    CONSTRAINT pk_t_invoice_d PRIMARY KEY (i_invoice_id, i_internal_no),
    CONSTRAINT fk_t_invoice_d_invoice_h FOREIGN KEY (i_invoice_id) REFERENCES t_invoice_h (i_invoice_id),
    CONSTRAINT fk_t_invoice_d_product FOREIGN KEY (i_product_id) REFERENCES m_product (i_product_id),
    CONSTRAINT ck_t_invoice_d_internal_no CHECK (REGEXP_LIKE(i_internal_no, '^[0-9]{6}$', 'c') AND i_internal_no <> '000000'),
    CONSTRAINT ck_t_invoice_d_qty CHECK (i_qty > 0),
    CONSTRAINT ck_t_invoice_d_unit_price CHECK (i_unit_price >= 0),
    CONSTRAINT ck_t_invoice_d_tax_rate CHECK (i_tax_rate BETWEEN 0 AND 1),
    CONSTRAINT ck_t_invoice_d_active CHECK (i_active IN (0, 1))
);
-- FK to the header is covered by the PK's leading column; FK to the product needs its own index
CREATE INDEX ix_t_invoice_d_product_id ON t_invoice_d (i_product_id);

-- Standalone transaction: one record per event, role FKs to the same master
CREATE TABLE t_stock_move (
    i_stock_move_id      VARCHAR2(20 BYTE)  NOT NULL,
    i_move_at            NUMBER(19)         NOT NULL,
    i_product_id         VARCHAR2(20 BYTE)  NOT NULL,
    i_from_warehouse_id  VARCHAR2(20 BYTE)  NULL,  -- NULL = received from outside
    i_to_warehouse_id    VARCHAR2(20 BYTE)  NULL,  -- NULL = shipped out
    i_qty                NUMBER(19,4)       NOT NULL,
    i_created_at         NUMBER(19) DEFAULT ROUND((TRUNC(SYS_EXTRACT_UTC(SYSTIMESTAMP), 'MI') - DATE '1970-01-01') * 86400) + FLOOR(EXTRACT(SECOND FROM SYS_EXTRACT_UTC(SYSTIMESTAMP))) NOT NULL,
    i_created_func_nm    VARCHAR2(100 BYTE) NOT NULL,
    i_created_by         VARCHAR2(50 BYTE)  NOT NULL,
    i_updated_at         NUMBER(19) DEFAULT ROUND((TRUNC(SYS_EXTRACT_UTC(SYSTIMESTAMP), 'MI') - DATE '1970-01-01') * 86400) + FLOOR(EXTRACT(SECOND FROM SYS_EXTRACT_UTC(SYSTIMESTAMP))) NOT NULL,
    i_updated_func_nm    VARCHAR2(100 BYTE) NOT NULL,
    i_updated_by         VARCHAR2(50 BYTE)  NOT NULL,
    i_active             NUMBER(1) DEFAULT 1 NOT NULL,
    CONSTRAINT pk_t_stock_move PRIMARY KEY (i_stock_move_id),
    CONSTRAINT fk_t_stock_move_product FOREIGN KEY (i_product_id) REFERENCES m_product (i_product_id),
    CONSTRAINT fk_t_stock_move_from_warehouse FOREIGN KEY (i_from_warehouse_id) REFERENCES m_warehouse (i_warehouse_id),
    CONSTRAINT fk_t_stock_move_to_warehouse FOREIGN KEY (i_to_warehouse_id) REFERENCES m_warehouse (i_warehouse_id),
    CONSTRAINT ck_t_stock_move_stock_move_id CHECK (REGEXP_LIKE(i_stock_move_id, '^smv[0-9]{14}$', 'c')),
    CONSTRAINT ck_t_stock_move_qty CHECK (i_qty > 0),
    CONSTRAINT ck_t_stock_move_warehouse CHECK (
        (i_from_warehouse_id IS NOT NULL OR i_to_warehouse_id IS NOT NULL)
        AND (i_from_warehouse_id IS NULL OR i_to_warehouse_id IS NULL OR i_from_warehouse_id <> i_to_warehouse_id)),
    CONSTRAINT ck_t_stock_move_active CHECK (i_active IN (0, 1))
);
CREATE INDEX ix_t_stock_move_product_id_move_at ON t_stock_move (i_product_id, i_move_at);
CREATE INDEX ix_t_stock_move_from_warehouse_id ON t_stock_move (i_from_warehouse_id);
CREATE INDEX ix_t_stock_move_to_warehouse_id ON t_stock_move (i_to_warehouse_id);

-- Default read path: active rows only (D2)
CREATE VIEW v_m_product AS SELECT * FROM m_product WHERE i_active = 1;
```

## 6. ID generation and documents

### ID from the counter table
The procedure runs inside the caller's transaction, so the counter rolls back with the row that uses the ID. When two sessions create the first counter row for a period at the same moment, the second waits on the first one's uncommitted key, then gets `DUP_VAL_ON_INDEX` and increments the row instead.

```sql
CREATE OR REPLACE PROCEDURE p_next_id (
    p_abbr    IN  VARCHAR2,
    p_date    IN  VARCHAR2,   -- 'YYYYMMDD', 'YYYYMM' or NULL (Oracle treats '' as NULL)
    p_digits  IN  PLS_INTEGER,
    p_func_nm IN  VARCHAR2,
    p_by      IN  VARCHAR2,
    p_id      OUT VARCHAR2
) AS
    v_seq_id VARCHAR2(20) := p_abbr || p_date;
    v_now    NUMBER(19)   := ROUND((TRUNC(SYS_EXTRACT_UTC(SYSTIMESTAMP), 'MI') - DATE '1970-01-01') * 86400)
                             + FLOOR(EXTRACT(SECOND FROM SYS_EXTRACT_UTC(SYSTIMESTAMP)));
    v_n      NUMBER(19);
BEGIN
    UPDATE m_id_seq
       SET i_last_val = i_last_val + 1,
           i_updated_at = v_now, i_updated_func_nm = p_func_nm, i_updated_by = p_by
     WHERE i_id_seq_id = v_seq_id
    RETURNING i_last_val INTO v_n;

    IF SQL%ROWCOUNT = 0 THEN
        BEGIN
            INSERT INTO m_id_seq (i_id_seq_id, i_last_val,
                                  i_created_at, i_created_func_nm, i_created_by,
                                  i_updated_at, i_updated_func_nm, i_updated_by)
            VALUES (v_seq_id, 1, v_now, p_func_nm, p_by, v_now, p_func_nm, p_by);
            v_n := 1;
        EXCEPTION
            WHEN DUP_VAL_ON_INDEX THEN
                UPDATE m_id_seq
                   SET i_last_val = i_last_val + 1,
                       i_updated_at = v_now, i_updated_func_nm = p_func_nm, i_updated_by = p_by
                 WHERE i_id_seq_id = v_seq_id
                RETURNING i_last_val INTO v_n;
        END;
    END IF;

    IF v_n >= POWER(10, p_digits) THEN
        RAISE_APPLICATION_ERROR(-20001, 'ID counter exhausted for ' || v_seq_id);
    END IF;

    p_id := v_seq_id || LPAD(TO_CHAR(v_n), p_digits, '0');
END p_next_id;
/
```

In SQL*Plus and SQLcl, a PL/SQL unit ends with `/` on its own line; drivers send the block without it.

### Master rows
```sql
DECLARE
    v_id VARCHAR2(20);
BEGIN
    p_next_id('cus', NULL, 6, 'customer.create', 'usr000001', v_id);
    INSERT INTO m_customer (i_customer_id, i_customer_nm,
                            i_created_func_nm, i_created_by, i_updated_func_nm, i_updated_by)
    VALUES (v_id, 'บริษัท ตัวอย่าง จำกัด', 'customer.create', 'usr000001', 'customer.create', 'usr000001');

    p_next_id('prd', TO_CHAR(SYSTIMESTAMP AT TIME ZONE 'Asia/Bangkok', 'YYYYMM'), 4,
              'product.create', 'usr000001', v_id);
    INSERT INTO m_product (i_product_id, i_sku, i_product_nm, i_unit_price,
                           i_created_func_nm, i_created_by, i_updated_func_nm, i_updated_by)
    VALUES (v_id, 'SKU-001', '商品 A', 100, 'product.create', 'usr000001', 'product.create', 'usr000001');

    p_next_id('wh', NULL, 4, 'warehouse.create', 'usr000001', v_id);
    INSERT INTO m_warehouse (i_warehouse_id, i_warehouse_nm, i_open_time, i_close_time,
                             i_created_func_nm, i_created_by, i_updated_func_nm, i_updated_by)
    VALUES (v_id, '東京倉庫', 30600, 64800, 'warehouse.create', 'usr000001', 'warehouse.create', 'usr000001');
    COMMIT;
END;
/
```

**Optional sequence** for an ID without a date part (gaps after a rollback are acceptable):

```sql
CREATE SEQUENCE sq_m_customer START WITH 1 MINVALUE 1 MAXVALUE 999999 NOCYCLE;
-- 'cus' || LPAD(TO_CHAR(sq_m_customer.NEXTVAL), 6, '0')
```

### Creating a Header/Detail document
```sql
DECLARE
    v_func VARCHAR2(100) := 'invoice.create';
    v_by   VARCHAR2(50)  := 'usr000001';
    v_now  NUMBER(19)    := ROUND((TRUNC(SYS_EXTRACT_UTC(SYSTIMESTAMP), 'MI') - DATE '1970-01-01') * 86400)
                            + FLOOR(EXTRACT(SECOND FROM SYS_EXTRACT_UTC(SYSTIMESTAMP)));
    v_id   VARCHAR2(20);
BEGIN
    p_next_id('inv', TO_CHAR(SYSTIMESTAMP AT TIME ZONE 'Asia/Bangkok', 'YYYYMMDD'), 6, v_func, v_by, v_id);

    INSERT INTO t_invoice_h (i_invoice_id, i_invoice_at, i_customer_id, i_ccy,
                             i_created_at, i_created_func_nm, i_created_by,
                             i_updated_at, i_updated_func_nm, i_updated_by)
    SELECT v_id, v_now, c.i_customer_id, 'THB', v_now, v_func, v_by, v_now, v_func, v_by
      FROM m_customer c
     WHERE c.i_customer_id = 'cus000001' AND c.i_active = 1;   -- never reference an inactive parent

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
     FETCH FIRST 1 ROW ONLY;

    -- Recalculate the stored total from active lines, in the same transaction
    UPDATE t_invoice_h
       SET i_total_amt = (SELECT NVL(SUM(i_line_amt), 0) FROM t_invoice_d
                           WHERE i_invoice_id = v_id AND i_active = 1),
           i_updated_at = v_now, i_updated_func_nm = v_func, i_updated_by = v_by
     WHERE i_invoice_id = v_id;
    COMMIT;
END;
/
```

### Adding a line to an existing document
```sql
DECLARE
    v_id     VARCHAR2(20);
    v_locked VARCHAR2(20);
    v_next   NUMBER;
BEGIN
    SELECT MIN(i_invoice_id) INTO v_id FROM t_invoice_h;

    -- 1. Lock the header first (fixed lock order: header, then details)
    SELECT i_invoice_id INTO v_locked
      FROM t_invoice_h
     WHERE i_invoice_id = v_id AND i_status = 'draft' AND i_active = 1
       FOR UPDATE;   -- raises NO_DATA_FOUND when the invoice can't take new lines

    -- 2. Next line number = highest + 1, counting inactive lines too (numbers are never reused).
    --    LPAD cuts longer numbers, so check the limit explicitly.
    SELECT NVL(MAX(TO_NUMBER(i_internal_no)), 0) + 1 INTO v_next
      FROM t_invoice_d WHERE i_invoice_id = v_id;
    IF v_next > 999999 THEN
        RAISE_APPLICATION_ERROR(-20003, 'Line number limit reached');
    END IF;

    INSERT INTO t_invoice_d (i_invoice_id, i_internal_no, i_product_id, i_desc,
                             i_qty, i_unit_price, i_tax_rate, i_line_amt,
                             i_created_func_nm, i_created_by, i_updated_func_nm, i_updated_by)
    VALUES (v_id, LPAD(TO_CHAR(v_next), 6, '0'),
            (SELECT MIN(i_product_id) FROM v_m_product), 'Product B', 1, 150, 0.07, 150,
            'invoice.add_line', 'usr000001', 'invoice.add_line', 'usr000001');
    COMMIT;
END;
/
```

### Removing a line (soft delete)
```sql
UPDATE t_invoice_d
   SET i_active = 0,
       i_updated_at = ROUND((TRUNC(SYS_EXTRACT_UTC(SYSTIMESTAMP), 'MI') - DATE '1970-01-01') * 86400)
                      + FLOOR(EXTRACT(SECOND FROM SYS_EXTRACT_UTC(SYSTIMESTAMP))),
       i_updated_func_nm = 'invoice.remove_line', i_updated_by = 'usr000001'
 WHERE i_invoice_id = (SELECT MIN(i_invoice_id) FROM t_invoice_h) AND i_internal_no = '000002';
COMMIT;
```
Recalculate the header total in the same transaction, as above.

## 7. Unique codes among active rows only
Oracle has no filtered indexes, but a B-tree index skips rows whose key is entirely NULL. A function-based unique index that returns NULL for inactive rows therefore enforces uniqueness among active rows only:

```sql
ALTER TABLE m_product DROP CONSTRAINT uq_m_product_sku DROP INDEX;
CREATE UNIQUE INDEX uq_m_product_sku ON m_product (CASE WHEN i_active = 1 THEN i_sku END);
```

## 8. Performance notes
- **Indexes are not created for FKs automatically.** Create the `ix_` indexes yourself (K6). Unindexed FKs also make Oracle lock the whole child table when a parent row is updated or deleted.
- **Bind ID parameters as `VARCHAR2`**, not `NVARCHAR2`, so comparisons with `BYTE` columns use the index.
- **Range queries on `_at`** stay number comparisons. Never wrap the column in a conversion function inside a `WHERE`.
- **Sequential IDs** (abbreviation + date + counter) add to the right edge of the index. Under very high insert concurrency that edge becomes a hot block; reverse-key indexes are not an option because they break range scans, so prefer hash-partitioning the index in that case.

## 9. Catalog audit query

This query lists violations of the rules that can be checked from the catalog: N1, N2, N4, N5, N6, K1, K2, K3, K5, K6, U1, D1, D2, D3 and D4. Run it as the schema owner. An empty result means every automated check passed; the remaining rules need the manual review in `../schema-validation.md`. Oracle shows unquoted names in uppercase, so the pattern is uppercase in either project case.

```sql
WITH t AS (
    SELECT table_name AS tbl FROM user_tables WHERE table_name NOT LIKE 'BIN$%'
), col AS (
    SELECT c.table_name AS tbl, c.column_name AS col, c.data_type AS typ, c.data_precision AS prec,
           c.data_scale AS scale, c.char_used, c.data_length, c.nullable,
           CASE WHEN EXISTS (SELECT 1 FROM user_cons_columns cc JOIN user_constraints k
                                ON k.constraint_name = cc.constraint_name
                             WHERE cc.table_name = c.table_name AND cc.column_name = c.column_name
                               AND k.constraint_type IN ('P', 'R'))
                THEN 1 ELSE 0 END AS is_key
      FROM user_tab_columns c JOIN t ON t.tbl = c.table_name
), fk AS (
    SELECT k.constraint_name AS name, k.table_name AS tbl, k.r_constraint_name AS rname, k.delete_rule
      FROM user_constraints k JOIN t ON t.tbl = k.table_name WHERE k.constraint_type = 'R'
), audit AS (
    SELECT 'I_CREATED_AT' AS c, 'T' AS kind FROM dual UNION ALL SELECT 'I_CREATED_FUNC_NM', 'S' FROM dual
    UNION ALL SELECT 'I_CREATED_BY', 'S' FROM dual UNION ALL SELECT 'I_UPDATED_AT', 'T' FROM dual
    UNION ALL SELECT 'I_UPDATED_FUNC_NM', 'S' FROM dual UNION ALL SELECT 'I_UPDATED_BY', 'S' FROM dual
    UNION ALL SELECT 'I_ACTIVE', 'B' FROM dual
)
SELECT 'N1' AS rule_id, tbl AS object_name, 'name must be unquoted A-Z0-9_, start with a letter, no __ or trailing _' AS problem
  FROM t WHERE NOT REGEXP_LIKE(tbl, '^[A-Z][A-Z0-9]*(_[A-Z0-9]+)*$', 'c')
UNION ALL SELECT 'N2', tbl, 'table must start with m_ or t_' FROM t WHERE SUBSTR(tbl, 1, 2) NOT IN ('M_', 'T_')
UNION ALL SELECT 'N5', tbl, 'table name longer than 30' FROM t WHERE LENGTH(tbl) > 30
UNION ALL SELECT 'N1', tbl || '.' || col, 'name must be unquoted A-Z0-9_, start with a letter, no __ or trailing _'
  FROM col WHERE NOT REGEXP_LIKE(col, '^[A-Z][A-Z0-9]*(_[A-Z0-9]+)*$', 'c')
UNION ALL SELECT 'N4', tbl || '.' || col, 'column must start with i_' FROM col WHERE SUBSTR(col, 1, 2) <> 'I_'
UNION ALL SELECT 'N5', tbl || '.' || col, 'column name longer than 30' FROM col WHERE LENGTH(col) > 30
UNION ALL SELECT 'K1', tbl || '.' || col, 'key column must be VARCHAR2, is ' || typ
  FROM col WHERE is_key = 1 AND typ <> 'VARCHAR2'
UNION ALL SELECT 'K2', tbl || '.' || col, 'key column must use BYTE length semantics (ASCII)'
  FROM col WHERE is_key = 1 AND typ = 'VARCHAR2' AND char_used <> 'B'
UNION ALL SELECT 'K3', fk.name, 'FK column ' || cc.column_name || ' differs from ' || pc.column_name || ' in type or length'
  FROM fk
  JOIN user_cons_columns cc ON cc.constraint_name = fk.name
  JOIN user_cons_columns pc ON pc.constraint_name = fk.rname AND pc.position = cc.position
  JOIN user_tab_columns ccol ON ccol.table_name = cc.table_name AND ccol.column_name = cc.column_name
  JOIN user_tab_columns pcol ON pcol.table_name = pc.table_name AND pcol.column_name = pc.column_name
 WHERE ccol.data_type <> pcol.data_type OR ccol.data_length <> pcol.data_length OR ccol.char_used <> pcol.char_used
UNION ALL SELECT 'K5', name, 'FK must have no ON DELETE action' FROM fk WHERE delete_rule <> 'NO ACTION'
UNION ALL SELECT 'K6', fk.name, 'FK columns are not the leading columns of any index'
  FROM fk WHERE NOT EXISTS (
        SELECT 1 FROM user_indexes i
         WHERE i.table_name = fk.tbl
           AND (SELECT COUNT(*) FROM user_cons_columns cc
                  JOIN user_ind_columns ic ON ic.index_name = i.index_name
                                          AND ic.column_name = cc.column_name AND ic.column_position = cc.position
                 WHERE cc.constraint_name = fk.name)
             = (SELECT COUNT(*) FROM user_cons_columns cc WHERE cc.constraint_name = fk.name))
UNION ALL SELECT 'U1', tbl || '.' || col, 'date/time type not allowed: ' || typ
  FROM col WHERE typ = 'DATE' OR typ LIKE 'TIMESTAMP%' OR typ LIKE 'INTERVAL%'
UNION ALL SELECT 'U1', tbl || '.' || col, '_at/_date column must be NUMBER(19), is ' || typ
  FROM col WHERE (col LIKE '%\_AT' ESCAPE '\' OR col LIKE '%\_DATE' ESCAPE '\')
   AND NOT (typ = 'NUMBER' AND prec = 19 AND scale = 0)
UNION ALL SELECT CASE WHEN a.c = 'I_ACTIVE' THEN 'D2' ELSE 'D1' END, t.tbl, 'missing ' || LOWER(a.c)
  FROM t CROSS JOIN audit a
 WHERE NOT EXISTS (SELECT 1 FROM col WHERE col.tbl = t.tbl AND col.col = a.c)
UNION ALL SELECT CASE WHEN a.kind = 'B' THEN 'D2' ELSE 'D1' END, col.tbl || '.' || col.col,
       'must be NOT NULL ' || CASE a.kind WHEN 'T' THEN 'NUMBER(19)' WHEN 'B' THEN 'NUMBER(1)' ELSE 'VARCHAR2 BYTE' END
       || ', is ' || col.typ || CASE WHEN col.nullable = 'Y' THEN ' NULL' END
  FROM col JOIN audit a ON a.c = col.col
 WHERE col.nullable = 'Y'
    OR (a.kind = 'T' AND NOT (col.typ = 'NUMBER' AND col.prec = 19 AND col.scale = 0))
    OR (a.kind = 'B' AND NOT ((col.typ = 'NUMBER' AND col.prec = 1 AND col.scale = 0) OR col.typ = 'BOOLEAN'))
    OR (a.kind = 'S' AND NOT (col.typ = 'VARCHAR2' AND col.char_used = 'B'))
UNION ALL SELECT 'D3', tbl || '.' || col, 'flag must be NOT NULL NUMBER(1) or BOOLEAN, is ' || typ
  FROM col WHERE (col LIKE 'I\_IS\_%' ESCAPE '\' OR col LIKE 'I\_HAS\_%' ESCAPE '\')
   AND (nullable = 'Y' OR NOT ((typ = 'NUMBER' AND prec = 1 AND scale = 0) OR typ = 'BOOLEAN'))
UNION ALL SELECT 'D4', tbl || '.' || col, 'floating-point or unconstrained NUMBER: ' || typ
  FROM col WHERE typ IN ('FLOAT', 'BINARY_FLOAT', 'BINARY_DOUBLE') OR (typ = 'NUMBER' AND prec IS NULL)
UNION ALL SELECT 'N6', k.constraint_name, 'constraint name must start with '
       || CASE k.constraint_type WHEN 'P' THEN 'pk_' WHEN 'R' THEN 'fk_' WHEN 'U' THEN 'uq_' ELSE 'ck_' END || LOWER(k.table_name)
  FROM user_constraints k JOIN t ON t.tbl = k.table_name
 WHERE k.constraint_type IN ('P', 'R', 'U', 'C')
   AND NOT (k.constraint_type = 'C' AND k.generated = 'GENERATED NAME'
            AND REGEXP_LIKE(k.search_condition_vc, '^"[A-Z0-9_]+" IS NOT NULL$'))   -- column NOT NULLs
   AND NOT (CASE WHEN k.constraint_type = 'P' AND k.constraint_name = 'PK_' || k.table_name THEN 1
                 WHEN k.constraint_type <> 'P'
                      AND SUBSTR(k.constraint_name, 1, LENGTH(k.table_name) + 4)
                          = CASE k.constraint_type WHEN 'R' THEN 'FK_' WHEN 'U' THEN 'UQ_' ELSE 'CK_' END || k.table_name || '_'
                 THEN 1 ELSE 0 END = 1)
UNION ALL SELECT 'N6', i.index_name, 'index name must start with ix_' || LOWER(i.table_name) || '_ (or uq_ when unique)'
  FROM user_indexes i JOIN t ON t.tbl = i.table_name
 WHERE i.index_type <> 'LOB'
   AND NOT EXISTS (SELECT 1 FROM user_constraints k WHERE k.index_name = i.index_name AND k.constraint_type IN ('P', 'U'))
   AND SUBSTR(i.index_name, 1, LENGTH(i.table_name) + 4)
       <> CASE WHEN i.uniqueness = 'UNIQUE' THEN 'UQ_' ELSE 'IX_' END || i.table_name || '_'
UNION ALL SELECT 'N6', constraint_name, 'object name longer than 63'
  FROM user_constraints WHERE LENGTH(constraint_name) > 63 AND table_name IN (SELECT tbl FROM t)
ORDER BY 1, 2;
```
