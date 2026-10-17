# MySQL and MariaDB Implementation

Requires **MySQL 8.0.16 or later** (enforced CHECK constraints, expression defaults) or **MariaDB 10.5 or later**, the InnoDB engine, and the `utf8mb4` character set. One file covers both; differences are marked. This file maps the logical standard to MySQL/MariaDB. It never changes a logical rule.

## 1. Type mapping

| Logical | MySQL / MariaDB | Notes |
|---|---|---|
| `string(n)` | `VARCHAR(n)` with the table's `utf8mb4` character set | `n` counts characters; up to 4 bytes each. Never `utf8`/`utf8mb3`, which can't store all Unicode |
| `string(n) ascii` | `VARCHAR(n) CHARACTER SET ascii COLLATE ascii_bin` | One byte per character; binary, case-sensitive comparison (K2) |
| `int` | `INT` | |
| `long` | `BIGINT` | |
| `boolean` | `BOOLEAN` + CHECK `IN (0,1)` | Stored as `TINYINT(1)`, which accepts any small integer without the CHECK |
| `decimal(p,s)` | `DECIMAL(p,s)` | Never `FLOAT` or `DOUBLE` |
| `timestamp` | `BIGINT` | Unix seconds. Never `DATETIME`, `TIMESTAMP` or `DATE` |

- End every `CREATE TABLE` with `ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci`. `utf8mb4_unicode_ci` exists in both MySQL and MariaDB; a newer collation (MySQL `utf8mb4_0900_ai_ci`, MariaDB `utf8mb4_uca1400_ai_ci`) is fine when only one product is targeted.
- Connect with `utf8mb4` as the connection character set (`charset=utf8mb4` in the driver). Otherwise Thai or Japanese text arrives as `?`.
- Never use `AUTO_INCREMENT` (K1).

## 2. Names and quoting
- **Lowercase project:** write names unquoted. Lowercase names work on every `lower_case_table_names` setting (0 on Linux, 1 on Windows, 2 on macOS), so the same scripts run everywhere.
- **Uppercase project:** table names are file names on disk. Every server must run with `lower_case_table_names = 0`, which is only supported on case-sensitive file systems; otherwise the tables are stored in lowercase. Column names are not affected.
- Backticks are never needed: names built with the `m_`/`t_`/`i_` prefixes are never keywords.
- Name limit: 64 characters, which covers N6.
- **The PK is always named `PRIMARY`.** MySQL and MariaDB ignore the name in `CONSTRAINT pk_<table> PRIMARY KEY`. Write it anyway for documentation; the catalog audit skips the PK name. This is a documented vendor limitation of N6.
- CHECK constraint names must be unique in the schema; the `ck_<table>_…` pattern ensures it.

## 3. Unix time (seconds)

| Task | Expression |
|---|---|
| Now | `TIMESTAMPDIFF(SECOND, '1970-01-01', UTC_TIMESTAMP())` |
| Unix → UTC `DATETIME` | `DATE_ADD('1970-01-01', INTERVAL u SECOND)` |
| UTC `DATETIME` → Unix | `TIMESTAMPDIFF(SECOND, '1970-01-01', dt)` |
| `DATE` → `_date` value | `TIMESTAMPDIFF(SECOND, '1970-01-01', d)` |
| `_date` value → `DATE` | `DATE(DATE_ADD('1970-01-01', INTERVAL u SECOND))` |
| Start of a local business day (Bangkok, fixed +07:00) | `TIMESTAMPDIFF(SECOND, '1970-01-01', CONVERT_TZ('2026-10-17 00:00:00', '+07:00', '+00:00'))` |

- These expressions don't depend on the session `time_zone` and work after 2038. Avoid `UNIX_TIMESTAMP(dt)` and `FROM_UNIXTIME(u)`: they convert through the session time zone, and older versions stop at 2038.
- `UTC_TIMESTAMP()` has no fractional seconds, so the result is always whole seconds.
- Named zones (`'Asia/Bangkok'`) in `CONVERT_TZ` need the time-zone tables loaded (`mysql_tzinfo_to_sql`). A fixed offset works for zones without daylight saving.

## 4. CHECK patterns
`REGEXP` on an `ascii_bin` column is case-sensitive, and the `^…$` anchors reject spaces:

```sql
CHECK (i_customer_id REGEXP '^cus[0-9]{6}$')       -- fixed-format ID
CHECK (i_sku REGEXP '^[A-Za-z0-9_-]+$')            -- business code
CHECK (i_active IN (0, 1))                          -- every boolean
CHECK (i_due_date % 86400 = 0)                      -- calendar date
```

## 5. Example schema (compliant)

FK indexes are declared inside `CREATE TABLE`, before the FKs. Otherwise InnoDB creates its own index for each FK, named after the constraint, and the `ix_` name is lost. The data-access layer supplies the `func_nm` and `by` audit values on every insert.

```sql
-- Counter for IDs (configuration -> master)
CREATE TABLE m_id_seq (
    i_id_seq_id        VARCHAR(20)  CHARACTER SET ascii COLLATE ascii_bin NOT NULL,
    i_last_val         BIGINT       NOT NULL,
    i_created_at       BIGINT       NOT NULL DEFAULT (TIMESTAMPDIFF(SECOND, '1970-01-01', UTC_TIMESTAMP())),
    i_created_func_nm  VARCHAR(100) CHARACTER SET ascii COLLATE ascii_bin NOT NULL,
    i_created_by       VARCHAR(50)  CHARACTER SET ascii COLLATE ascii_bin NOT NULL,
    i_updated_at       BIGINT       NOT NULL DEFAULT (TIMESTAMPDIFF(SECOND, '1970-01-01', UTC_TIMESTAMP())),
    i_updated_func_nm  VARCHAR(100) CHARACTER SET ascii COLLATE ascii_bin NOT NULL,
    i_updated_by       VARCHAR(50)  CHARACTER SET ascii COLLATE ascii_bin NOT NULL,
    i_active           BOOLEAN      NOT NULL DEFAULT TRUE,
    CONSTRAINT pk_m_id_seq PRIMARY KEY (i_id_seq_id),
    CONSTRAINT ck_m_id_seq_id_seq_id CHECK (i_id_seq_id REGEXP '^[a-z]{2,4}([0-9]{6}|[0-9]{8})?$'),
    CONSTRAINT ck_m_id_seq_last_val CHECK (i_last_val >= 0),
    CONSTRAINT ck_m_id_seq_active CHECK (i_active IN (0, 1))
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- Master, ID without a date part: cus + 6 digits
CREATE TABLE m_customer (
    i_customer_id      VARCHAR(20)  CHARACTER SET ascii COLLATE ascii_bin NOT NULL,
    i_customer_nm      VARCHAR(200) NOT NULL,
    i_tax_no           VARCHAR(30)  CHARACTER SET ascii COLLATE ascii_bin NULL,  -- NULL = not registered for tax
    i_created_at       BIGINT       NOT NULL DEFAULT (TIMESTAMPDIFF(SECOND, '1970-01-01', UTC_TIMESTAMP())),
    i_created_func_nm  VARCHAR(100) CHARACTER SET ascii COLLATE ascii_bin NOT NULL,
    i_created_by       VARCHAR(50)  CHARACTER SET ascii COLLATE ascii_bin NOT NULL,
    i_updated_at       BIGINT       NOT NULL DEFAULT (TIMESTAMPDIFF(SECOND, '1970-01-01', UTC_TIMESTAMP())),
    i_updated_func_nm  VARCHAR(100) CHARACTER SET ascii COLLATE ascii_bin NOT NULL,
    i_updated_by       VARCHAR(50)  CHARACTER SET ascii COLLATE ascii_bin NOT NULL,
    i_active           BOOLEAN      NOT NULL DEFAULT TRUE,
    CONSTRAINT pk_m_customer PRIMARY KEY (i_customer_id),
    CONSTRAINT ck_m_customer_customer_id CHECK (i_customer_id REGEXP '^cus[0-9]{6}$'),
    CONSTRAINT ck_m_customer_tax_no CHECK (i_tax_no REGEXP '^[A-Za-z0-9_-]+$'),
    CONSTRAINT ck_m_customer_active CHECK (i_active IN (0, 1))
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- Master, monthly ID: prd + yyyymm + 4 digits; changeable business code in its own column
CREATE TABLE m_product (
    i_product_id       VARCHAR(20)  CHARACTER SET ascii COLLATE ascii_bin NOT NULL,
    i_sku              VARCHAR(30)  CHARACTER SET ascii COLLATE ascii_bin NOT NULL,
    i_product_nm       VARCHAR(200) NOT NULL,
    i_unit_price       DECIMAL(19,4) NOT NULL,
    i_is_taxable       BOOLEAN      NOT NULL DEFAULT TRUE,
    i_created_at       BIGINT       NOT NULL DEFAULT (TIMESTAMPDIFF(SECOND, '1970-01-01', UTC_TIMESTAMP())),
    i_created_func_nm  VARCHAR(100) CHARACTER SET ascii COLLATE ascii_bin NOT NULL,
    i_created_by       VARCHAR(50)  CHARACTER SET ascii COLLATE ascii_bin NOT NULL,
    i_updated_at       BIGINT       NOT NULL DEFAULT (TIMESTAMPDIFF(SECOND, '1970-01-01', UTC_TIMESTAMP())),
    i_updated_func_nm  VARCHAR(100) CHARACTER SET ascii COLLATE ascii_bin NOT NULL,
    i_updated_by       VARCHAR(50)  CHARACTER SET ascii COLLATE ascii_bin NOT NULL,
    i_active           BOOLEAN      NOT NULL DEFAULT TRUE,
    CONSTRAINT pk_m_product PRIMARY KEY (i_product_id),
    CONSTRAINT uq_m_product_sku UNIQUE (i_sku),   -- unique for all time (default)
    CONSTRAINT ck_m_product_product_id CHECK (i_product_id REGEXP '^prd[0-9]{10}$'),
    CONSTRAINT ck_m_product_sku CHECK (i_sku REGEXP '^[A-Za-z0-9_-]+$'),
    CONSTRAINT ck_m_product_unit_price CHECK (i_unit_price >= 0),
    CONSTRAINT ck_m_product_is_taxable CHECK (i_is_taxable IN (0, 1)),
    CONSTRAINT ck_m_product_active CHECK (i_active IN (0, 1))
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- Master with times of day: wh + 4 digits
CREATE TABLE m_warehouse (
    i_warehouse_id     VARCHAR(20)  CHARACTER SET ascii COLLATE ascii_bin NOT NULL,
    i_warehouse_nm     VARCHAR(200) NOT NULL,
    i_open_time        INT          NOT NULL,  -- seconds after local midnight
    i_close_time       INT          NOT NULL,
    i_created_at       BIGINT       NOT NULL DEFAULT (TIMESTAMPDIFF(SECOND, '1970-01-01', UTC_TIMESTAMP())),
    i_created_func_nm  VARCHAR(100) CHARACTER SET ascii COLLATE ascii_bin NOT NULL,
    i_created_by       VARCHAR(50)  CHARACTER SET ascii COLLATE ascii_bin NOT NULL,
    i_updated_at       BIGINT       NOT NULL DEFAULT (TIMESTAMPDIFF(SECOND, '1970-01-01', UTC_TIMESTAMP())),
    i_updated_func_nm  VARCHAR(100) CHARACTER SET ascii COLLATE ascii_bin NOT NULL,
    i_updated_by       VARCHAR(50)  CHARACTER SET ascii COLLATE ascii_bin NOT NULL,
    i_active           BOOLEAN      NOT NULL DEFAULT TRUE,
    CONSTRAINT pk_m_warehouse PRIMARY KEY (i_warehouse_id),
    CONSTRAINT ck_m_warehouse_warehouse_id CHECK (i_warehouse_id REGEXP '^wh[0-9]{4}$'),
    CONSTRAINT ck_m_warehouse_open_time CHECK (i_open_time BETWEEN 0 AND 86399),
    CONSTRAINT ck_m_warehouse_close_time CHECK (i_close_time BETWEEN 0 AND 86399),
    CONSTRAINT ck_m_warehouse_active CHECK (i_active IN (0, 1))
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- Header, daily ID: inv + yyyymmdd + 6 digits
CREATE TABLE t_invoice_h (
    i_invoice_id       VARCHAR(20)  CHARACTER SET ascii COLLATE ascii_bin NOT NULL,
    i_invoice_at       BIGINT       NOT NULL,
    i_due_date         BIGINT       NULL,  -- NULL = payable immediately
    i_customer_id      VARCHAR(20)  CHARACTER SET ascii COLLATE ascii_bin NOT NULL,
    i_status           VARCHAR(20)  CHARACTER SET ascii COLLATE ascii_bin NOT NULL DEFAULT 'draft',
    i_ccy              VARCHAR(3)   CHARACTER SET ascii COLLATE ascii_bin NOT NULL,
    i_total_amt        DECIMAL(19,4) NOT NULL DEFAULT 0,  -- stored total: sum of active lines
    i_note             VARCHAR(2000) NULL,               -- NULL = no note
    i_created_at       BIGINT       NOT NULL DEFAULT (TIMESTAMPDIFF(SECOND, '1970-01-01', UTC_TIMESTAMP())),
    i_created_func_nm  VARCHAR(100) CHARACTER SET ascii COLLATE ascii_bin NOT NULL,
    i_created_by       VARCHAR(50)  CHARACTER SET ascii COLLATE ascii_bin NOT NULL,
    i_updated_at       BIGINT       NOT NULL DEFAULT (TIMESTAMPDIFF(SECOND, '1970-01-01', UTC_TIMESTAMP())),
    i_updated_func_nm  VARCHAR(100) CHARACTER SET ascii COLLATE ascii_bin NOT NULL,
    i_updated_by       VARCHAR(50)  CHARACTER SET ascii COLLATE ascii_bin NOT NULL,
    i_active           BOOLEAN      NOT NULL DEFAULT TRUE,
    CONSTRAINT pk_t_invoice_h PRIMARY KEY (i_invoice_id),
    -- FK index; also serves "a customer's invoices by date"
    INDEX ix_t_invoice_h_customer_id_invoice_at (i_customer_id, i_invoice_at),
    CONSTRAINT fk_t_invoice_h_customer FOREIGN KEY (i_customer_id)
        REFERENCES m_customer (i_customer_id) ON DELETE NO ACTION ON UPDATE NO ACTION,
    CONSTRAINT ck_t_invoice_h_invoice_id CHECK (i_invoice_id REGEXP '^inv[0-9]{14}$'),
    CONSTRAINT ck_t_invoice_h_due_date CHECK (i_due_date % 86400 = 0),
    CONSTRAINT ck_t_invoice_h_status CHECK (i_status IN ('draft', 'completed', 'cancelled')),
    CONSTRAINT ck_t_invoice_h_ccy CHECK (i_ccy REGEXP '^[A-Z]{3}$'),
    CONSTRAINT ck_t_invoice_h_total_amt CHECK (i_total_amt >= 0),
    CONSTRAINT ck_t_invoice_h_active CHECK (i_active IN (0, 1))
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- Detail: composite PK (header ID, i_internal_no)
CREATE TABLE t_invoice_d (
    i_invoice_id       VARCHAR(20)  CHARACTER SET ascii COLLATE ascii_bin NOT NULL,
    i_internal_no      VARCHAR(6)   CHARACTER SET ascii COLLATE ascii_bin NOT NULL,
    i_product_id       VARCHAR(20)  CHARACTER SET ascii COLLATE ascii_bin NOT NULL,
    i_desc             VARCHAR(500) NOT NULL,   -- snapshot of the product name
    i_qty              DECIMAL(19,4) NOT NULL,
    i_unit_price       DECIMAL(19,4) NOT NULL,  -- snapshot at sale time
    i_tax_rate         DECIMAL(9,6)  NOT NULL,  -- snapshot at sale time
    i_line_amt         DECIMAL(19,4) NOT NULL,
    i_created_at       BIGINT       NOT NULL DEFAULT (TIMESTAMPDIFF(SECOND, '1970-01-01', UTC_TIMESTAMP())),
    i_created_func_nm  VARCHAR(100) CHARACTER SET ascii COLLATE ascii_bin NOT NULL,
    i_created_by       VARCHAR(50)  CHARACTER SET ascii COLLATE ascii_bin NOT NULL,
    i_updated_at       BIGINT       NOT NULL DEFAULT (TIMESTAMPDIFF(SECOND, '1970-01-01', UTC_TIMESTAMP())),
    i_updated_func_nm  VARCHAR(100) CHARACTER SET ascii COLLATE ascii_bin NOT NULL,
    i_updated_by       VARCHAR(50)  CHARACTER SET ascii COLLATE ascii_bin NOT NULL,
    i_active           BOOLEAN      NOT NULL DEFAULT TRUE,
    CONSTRAINT pk_t_invoice_d PRIMARY KEY (i_invoice_id, i_internal_no),
    -- FK to the header is covered by the PK's leading column; FK to the product needs its own index
    INDEX ix_t_invoice_d_product_id (i_product_id),
    CONSTRAINT fk_t_invoice_d_invoice_h FOREIGN KEY (i_invoice_id)
        REFERENCES t_invoice_h (i_invoice_id) ON DELETE NO ACTION ON UPDATE NO ACTION,
    CONSTRAINT fk_t_invoice_d_product FOREIGN KEY (i_product_id)
        REFERENCES m_product (i_product_id) ON DELETE NO ACTION ON UPDATE NO ACTION,
    CONSTRAINT ck_t_invoice_d_internal_no CHECK (i_internal_no REGEXP '^[0-9]{6}$' AND i_internal_no <> '000000'),
    CONSTRAINT ck_t_invoice_d_qty CHECK (i_qty > 0),
    CONSTRAINT ck_t_invoice_d_unit_price CHECK (i_unit_price >= 0),
    CONSTRAINT ck_t_invoice_d_tax_rate CHECK (i_tax_rate BETWEEN 0 AND 1),
    CONSTRAINT ck_t_invoice_d_active CHECK (i_active IN (0, 1))
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- Standalone transaction: one record per event, role FKs to the same master
CREATE TABLE t_stock_move (
    i_stock_move_id      VARCHAR(20)  CHARACTER SET ascii COLLATE ascii_bin NOT NULL,
    i_move_at            BIGINT       NOT NULL,
    i_product_id         VARCHAR(20)  CHARACTER SET ascii COLLATE ascii_bin NOT NULL,
    i_from_warehouse_id  VARCHAR(20)  CHARACTER SET ascii COLLATE ascii_bin NULL,  -- NULL = received from outside
    i_to_warehouse_id    VARCHAR(20)  CHARACTER SET ascii COLLATE ascii_bin NULL,  -- NULL = shipped out
    i_qty                DECIMAL(19,4) NOT NULL,
    i_created_at         BIGINT       NOT NULL DEFAULT (TIMESTAMPDIFF(SECOND, '1970-01-01', UTC_TIMESTAMP())),
    i_created_func_nm    VARCHAR(100) CHARACTER SET ascii COLLATE ascii_bin NOT NULL,
    i_created_by         VARCHAR(50)  CHARACTER SET ascii COLLATE ascii_bin NOT NULL,
    i_updated_at         BIGINT       NOT NULL DEFAULT (TIMESTAMPDIFF(SECOND, '1970-01-01', UTC_TIMESTAMP())),
    i_updated_func_nm    VARCHAR(100) CHARACTER SET ascii COLLATE ascii_bin NOT NULL,
    i_updated_by         VARCHAR(50)  CHARACTER SET ascii COLLATE ascii_bin NOT NULL,
    i_active             BOOLEAN      NOT NULL DEFAULT TRUE,
    CONSTRAINT pk_t_stock_move PRIMARY KEY (i_stock_move_id),
    INDEX ix_t_stock_move_product_id_move_at (i_product_id, i_move_at),
    INDEX ix_t_stock_move_from_warehouse_id (i_from_warehouse_id),
    INDEX ix_t_stock_move_to_warehouse_id (i_to_warehouse_id),
    CONSTRAINT fk_t_stock_move_product FOREIGN KEY (i_product_id)
        REFERENCES m_product (i_product_id) ON DELETE NO ACTION ON UPDATE NO ACTION,
    CONSTRAINT fk_t_stock_move_from_warehouse FOREIGN KEY (i_from_warehouse_id)
        REFERENCES m_warehouse (i_warehouse_id) ON DELETE NO ACTION ON UPDATE NO ACTION,
    CONSTRAINT fk_t_stock_move_to_warehouse FOREIGN KEY (i_to_warehouse_id)
        REFERENCES m_warehouse (i_warehouse_id) ON DELETE NO ACTION ON UPDATE NO ACTION,
    CONSTRAINT ck_t_stock_move_stock_move_id CHECK (i_stock_move_id REGEXP '^smv[0-9]{14}$'),
    CONSTRAINT ck_t_stock_move_qty CHECK (i_qty > 0),
    CONSTRAINT ck_t_stock_move_warehouse CHECK (
        (i_from_warehouse_id IS NOT NULL OR i_to_warehouse_id IS NOT NULL)
        AND (i_from_warehouse_id IS NULL OR i_to_warehouse_id IS NULL OR i_from_warehouse_id <> i_to_warehouse_id)),
    CONSTRAINT ck_t_stock_move_active CHECK (i_active IN (0, 1))
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- Default read path: active rows only (D2)
CREATE VIEW v_m_product AS SELECT * FROM m_product WHERE i_active = 1;
```

## 6. ID generation and documents

### ID from the counter table
MySQL has no sequences, so every ID comes from `m_id_seq`. MariaDB 10.3+ has sequences, but the counter table keeps one method for both products. `LAST_INSERT_ID(expr)` returns the new value to the session in the same statement that increments it. Call the procedure inside the caller's transaction, so the counter rolls back with the row that uses the ID. Scripts run in the `mysql`/`mariadb` client need the `DELIMITER` lines; drivers send the `CREATE PROCEDURE` statement without them.

```sql
DELIMITER //
CREATE PROCEDURE p_next_id(IN p_abbr VARCHAR(4), IN p_date VARCHAR(8), IN p_digits INT,
                           IN p_func_nm VARCHAR(100), IN p_by VARCHAR(50), OUT p_id VARCHAR(20))
BEGIN
    DECLARE v_seq_id VARCHAR(20) CHARACTER SET ascii COLLATE ascii_bin DEFAULT CONCAT(p_abbr, p_date);
    DECLARE v_now    BIGINT DEFAULT TIMESTAMPDIFF(SECOND, '1970-01-01', UTC_TIMESTAMP());
    DECLARE v_n      BIGINT;

    -- Atomic insert-or-increment; the row lock is held until the caller commits
    INSERT INTO m_id_seq (i_id_seq_id, i_last_val,
                          i_created_at, i_created_func_nm, i_created_by,
                          i_updated_at, i_updated_func_nm, i_updated_by)
    VALUES (v_seq_id, LAST_INSERT_ID(1), v_now, p_func_nm, p_by, v_now, p_func_nm, p_by)
    ON DUPLICATE KEY UPDATE
        i_last_val        = LAST_INSERT_ID(i_last_val + 1),
        i_updated_at      = v_now,
        i_updated_func_nm = p_func_nm,
        i_updated_by      = p_by;
    SET v_n = LAST_INSERT_ID();

    IF v_n >= POWER(10, p_digits) THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'ID counter exhausted';
    END IF;

    SET p_id = CONCAT(v_seq_id, LPAD(v_n, p_digits, '0'));
END //
DELIMITER ;
```

### Master rows
```sql
START TRANSACTION;
CALL p_next_id('cus', '', 6, 'customer.create', 'usr000001', @id);
INSERT INTO m_customer (i_customer_id, i_customer_nm,
                        i_created_func_nm, i_created_by, i_updated_func_nm, i_updated_by)
VALUES (@id, 'บริษัท ตัวอย่าง จำกัด', 'customer.create', 'usr000001', 'customer.create', 'usr000001');

CALL p_next_id('prd', DATE_FORMAT(CONVERT_TZ(UTC_TIMESTAMP(), '+00:00', '+07:00'), '%Y%m'), 4,
               'product.create', 'usr000001', @id);
INSERT INTO m_product (i_product_id, i_sku, i_product_nm, i_unit_price,
                       i_created_func_nm, i_created_by, i_updated_func_nm, i_updated_by)
VALUES (@id, 'SKU-001', '商品 A', 100, 'product.create', 'usr000001', 'product.create', 'usr000001');

CALL p_next_id('wh', '', 4, 'warehouse.create', 'usr000001', @id);
INSERT INTO m_warehouse (i_warehouse_id, i_warehouse_nm, i_open_time, i_close_time,
                         i_created_func_nm, i_created_by, i_updated_func_nm, i_updated_by)
VALUES (@id, '東京倉庫', 30600, 64800, 'warehouse.create', 'usr000001', 'warehouse.create', 'usr000001');
COMMIT;
```

### Creating a Header/Detail document
```sql
START TRANSACTION;
SET @func = 'invoice.create', @by = 'usr000001';
SET @now  = TIMESTAMPDIFF(SECOND, '1970-01-01', UTC_TIMESTAMP());
CALL p_next_id('inv', DATE_FORMAT(CONVERT_TZ(UTC_TIMESTAMP(), '+00:00', '+07:00'), '%Y%m%d'), 6,
               @func, @by, @invoice_id);

INSERT INTO t_invoice_h (i_invoice_id, i_invoice_at, i_customer_id, i_ccy,
                         i_created_at, i_created_func_nm, i_created_by,
                         i_updated_at, i_updated_func_nm, i_updated_by)
SELECT @invoice_id, @now, c.i_customer_id, 'THB', @now, @func, @by, @now, @func, @by
  FROM m_customer c
 WHERE c.i_customer_id = 'cus000001' AND c.i_active = 1;   -- never reference an inactive parent

-- Lines are numbered 000001, 000002, ...; price, tax and name are copied (H3)
INSERT INTO t_invoice_d (i_invoice_id, i_internal_no, i_product_id, i_desc,
                         i_qty, i_unit_price, i_tax_rate, i_line_amt,
                         i_created_at, i_created_func_nm, i_created_by,
                         i_updated_at, i_updated_func_nm, i_updated_by)
SELECT @invoice_id, '000001', p.i_product_id, p.i_product_nm,
       2, p.i_unit_price, 0.07, 2 * p.i_unit_price,
       @now, @func, @by, @now, @func, @by
  FROM v_m_product p
 ORDER BY p.i_product_id
 LIMIT 1;

-- Recalculate the stored total from active lines, in the same transaction
UPDATE t_invoice_h
   SET i_total_amt = COALESCE((SELECT SUM(i_line_amt) FROM t_invoice_d
                                WHERE i_invoice_id = @invoice_id AND i_active = 1), 0),
       i_updated_at = @now, i_updated_func_nm = @func, i_updated_by = @by
 WHERE i_invoice_id = @invoice_id;
COMMIT;
```

### Adding a line to an existing document
This continues in the same session, so `@invoice_id` still holds the invoice created above.

```sql
START TRANSACTION;

-- 1. Lock the header first (fixed lock order: header, then details)
SELECT i_invoice_id INTO @locked
  FROM t_invoice_h
 WHERE i_invoice_id = @invoice_id AND i_status = 'draft' AND i_active = 1
   FOR UPDATE;
-- The application stops if @locked is NULL.

-- 2. Next line number = highest + 1, counting inactive lines too (numbers are never reused).
--    LPAD cuts a longer number down to 6 digits, so the CASE returns NULL at the limit
--    and NOT NULL stops the insert instead of reusing a number.
INSERT INTO t_invoice_d (i_invoice_id, i_internal_no, i_product_id, i_desc,
                         i_qty, i_unit_price, i_tax_rate, i_line_amt,
                         i_created_func_nm, i_created_by, i_updated_func_nm, i_updated_by)
SELECT @locked,
       CASE WHEN COALESCE(MAX(CAST(d.i_internal_no AS UNSIGNED)), 0) < 999999
            THEN LPAD(COALESCE(MAX(CAST(d.i_internal_no AS UNSIGNED)), 0) + 1, 6, '0') END,
       (SELECT MIN(i_product_id) FROM v_m_product), 'Product B', 1, 150, 0.07, 150,
       'invoice.add_line', 'usr000001', 'invoice.add_line', 'usr000001'
  FROM t_invoice_d d
 WHERE d.i_invoice_id = @locked;

COMMIT;
```

### Removing a line (soft delete)
```sql
UPDATE t_invoice_d
   SET i_active = 0,
       i_updated_at = TIMESTAMPDIFF(SECOND, '1970-01-01', UTC_TIMESTAMP()),
       i_updated_func_nm = 'invoice.remove_line', i_updated_by = 'usr000001'
 WHERE i_invoice_id = @invoice_id AND i_internal_no = '000002';
```
Recalculate the header total in the same transaction, as above.

## 7. Unique codes among active rows only
Neither product has filtered indexes, and MariaDB has no functional indexes. A virtual generated column that is NULL for inactive rows, with a unique index on it, works on both, because a unique index allows many NULLs:

```sql
ALTER TABLE m_product
    DROP INDEX uq_m_product_sku,
    ADD COLUMN i_sku_active VARCHAR(30) CHARACTER SET ascii COLLATE ascii_bin
        AS (CASE WHEN i_active = 1 THEN i_sku END) VIRTUAL,
    ADD UNIQUE INDEX uq_m_product_sku (i_sku_active);
```
`i_sku_active` is a technical column; the application never writes it. It is exempt from D1/D2 checks because it is derived. Recreate `v_m_product` after this change if it uses `SELECT *`.

## 8. Performance notes
- **The PK is the clustered index** in InnoDB. Abbreviation + date + counter IDs increase in insert order, so new rows go at the end. Every secondary index also stores the PK, which is another reason to keep IDs short ASCII strings.
- **Declare FK indexes in `CREATE TABLE`** (section 5), so InnoDB doesn't add its own.
- **Compare IDs with values of the same character set.** A `utf8mb4` parameter compared with an `ascii` column is converted, which can stop the index being used in some versions. Most drivers send parameters correctly; for literals in ad-hoc SQL, `_ascii'cus000001'` makes the type explicit.
- **Range queries on `_at`** stay integer comparisons. Never wrap the column in `FROM_UNIXTIME()` inside a `WHERE`.
- **Row size.** All `VARCHAR` columns of a table share a 65,535-byte limit, counted at 4 bytes per `utf8mb4` character. The standard sizes stay well within it.

## 9. Catalog audit query

This query lists violations of the rules that can be checked from `information_schema`: N1, N2, N4, N5, N6, K1, K2, K3, K5, K6, U1, D1, D2, D3 and D4. Run it while connected to the database to check. An empty result means every automated check passed; the remaining rules need the manual review in `../schema-validation.md`. The name pattern is for a lowercase project; in an uppercase project replace `a-z` with `A-Z` and the prefixes with their uppercase form. Generated columns such as `i_sku_active` are skipped.

```sql
WITH t AS (
    SELECT TABLE_NAME AS tbl FROM information_schema.TABLES
     WHERE TABLE_SCHEMA = DATABASE() AND TABLE_TYPE = 'BASE TABLE'
), col AS (
    SELECT c.TABLE_NAME AS tbl, c.COLUMN_NAME AS col, c.DATA_TYPE AS typ, c.COLUMN_TYPE AS ctype,
           c.CHARACTER_MAXIMUM_LENGTH AS len, c.CHARACTER_SET_NAME AS cs, c.COLLATION_NAME AS coll,
           c.IS_NULLABLE AS nullable,
           EXISTS (SELECT 1 FROM information_schema.KEY_COLUMN_USAGE k
                    JOIN information_schema.TABLE_CONSTRAINTS tc
                      ON tc.CONSTRAINT_SCHEMA = k.CONSTRAINT_SCHEMA AND tc.TABLE_NAME = k.TABLE_NAME
                     AND tc.CONSTRAINT_NAME = k.CONSTRAINT_NAME
                   WHERE k.TABLE_SCHEMA = c.TABLE_SCHEMA AND k.TABLE_NAME = c.TABLE_NAME
                     AND k.COLUMN_NAME = c.COLUMN_NAME
                     AND tc.CONSTRAINT_TYPE IN ('PRIMARY KEY', 'FOREIGN KEY')) AS is_key
      FROM information_schema.COLUMNS c JOIN t ON t.tbl = c.TABLE_NAME
     WHERE c.TABLE_SCHEMA = DATABASE() AND (c.GENERATION_EXPRESSION IS NULL OR c.GENERATION_EXPRESSION = '')
), fk AS (
    SELECT r.CONSTRAINT_NAME AS name, r.TABLE_NAME AS tbl, r.DELETE_RULE, r.UPDATE_RULE
      FROM information_schema.REFERENTIAL_CONSTRAINTS r WHERE r.CONSTRAINT_SCHEMA = DATABASE()
), audit (c, kind) AS (
    SELECT 'i_created_at', 't' UNION ALL SELECT 'i_created_func_nm', 's' UNION ALL SELECT 'i_created_by', 's'
    UNION ALL SELECT 'i_updated_at', 't' UNION ALL SELECT 'i_updated_func_nm', 's' UNION ALL SELECT 'i_updated_by', 's'
    UNION ALL SELECT 'i_active', 'b'
)
SELECT 'N1' AS rule_id, tbl AS object_name, 'name must be a-z0-9_, start with a letter, no __ or trailing _' AS problem
  FROM t WHERE NOT tbl REGEXP BINARY '^[a-z][a-z0-9]*(_[a-z0-9]+)*$'
UNION ALL SELECT 'N2', tbl, 'table must start with m_ or t_' FROM t WHERE LEFT(tbl, 2) NOT IN ('m_', 't_')
UNION ALL SELECT 'N5', tbl, 'table name longer than 30' FROM t WHERE CHAR_LENGTH(tbl) > 30
UNION ALL SELECT 'N1', CONCAT(tbl, '.', col), 'name must be a-z0-9_, start with a letter, no __ or trailing _'
  FROM col WHERE NOT col REGEXP BINARY '^[a-z][a-z0-9]*(_[a-z0-9]+)*$'
UNION ALL SELECT 'N4', CONCAT(tbl, '.', col), 'column must start with i_' FROM col WHERE LEFT(col, 2) <> 'i_'
UNION ALL SELECT 'N5', CONCAT(tbl, '.', col), 'column name longer than 30' FROM col WHERE CHAR_LENGTH(col) > 30
UNION ALL SELECT 'K1', CONCAT(tbl, '.', col), CONCAT('key column must be VARCHAR, is ', ctype)
  FROM col WHERE is_key AND typ <> 'varchar'
UNION ALL SELECT 'K2', CONCAT(tbl, '.', col), 'key column must be CHARACTER SET ascii COLLATE ascii_bin'
  FROM col WHERE is_key AND typ = 'varchar' AND IFNULL(coll, '') <> 'ascii_bin'
UNION ALL SELECT 'K3', k.CONSTRAINT_NAME, CONCAT('FK column ', k.COLUMN_NAME, ' differs from ', k.REFERENCED_COLUMN_NAME,
                                          ' in type, length or collation')
  FROM information_schema.KEY_COLUMN_USAGE k
  JOIN information_schema.COLUMNS cc ON cc.TABLE_SCHEMA = k.TABLE_SCHEMA AND cc.TABLE_NAME = k.TABLE_NAME
                                     AND cc.COLUMN_NAME = k.COLUMN_NAME
  JOIN information_schema.COLUMNS pc ON pc.TABLE_SCHEMA = k.REFERENCED_TABLE_SCHEMA AND pc.TABLE_NAME = k.REFERENCED_TABLE_NAME
                                     AND pc.COLUMN_NAME = k.REFERENCED_COLUMN_NAME
 WHERE k.TABLE_SCHEMA = DATABASE() AND k.REFERENCED_TABLE_NAME IS NOT NULL
   AND (cc.COLUMN_TYPE <> pc.COLUMN_TYPE OR IFNULL(cc.COLLATION_NAME, '') <> IFNULL(pc.COLLATION_NAME, ''))
UNION ALL SELECT 'K5', name, 'FK actions must be NO ACTION'
  FROM fk WHERE DELETE_RULE NOT IN ('NO ACTION', 'RESTRICT') OR UPDATE_RULE NOT IN ('NO ACTION', 'RESTRICT')
UNION ALL SELECT 'K6', fk.name, 'FK columns are not the leading columns of an ix_/uq_/PK index'
  FROM fk WHERE NOT EXISTS (
        SELECT 1 FROM information_schema.STATISTICS s
         WHERE s.TABLE_SCHEMA = DATABASE() AND s.TABLE_NAME = fk.tbl AND s.INDEX_NAME <> fk.name
         GROUP BY s.INDEX_NAME
        HAVING SUM(EXISTS (SELECT 1 FROM information_schema.KEY_COLUMN_USAGE k
                            WHERE k.CONSTRAINT_SCHEMA = DATABASE() AND k.TABLE_NAME = fk.tbl
                              AND k.CONSTRAINT_NAME = fk.name AND k.COLUMN_NAME = s.COLUMN_NAME
                              AND k.ORDINAL_POSITION = s.SEQ_IN_INDEX))
             = (SELECT COUNT(*) FROM information_schema.KEY_COLUMN_USAGE k
                 WHERE k.CONSTRAINT_SCHEMA = DATABASE() AND k.TABLE_NAME = fk.tbl AND k.CONSTRAINT_NAME = fk.name))
UNION ALL SELECT 'U1', CONCAT(tbl, '.', col), CONCAT('date/time type not allowed: ', ctype)
  FROM col WHERE typ IN ('date', 'datetime', 'timestamp', 'time', 'year')
UNION ALL SELECT 'U1', CONCAT(tbl, '.', col), CONCAT('_at/_date column must be BIGINT, is ', ctype)
  FROM col WHERE (col LIKE '%\_at' OR col LIKE '%\_date') AND typ <> 'bigint'
UNION ALL SELECT IF(audit.c = 'i_active', 'D2', 'D1'), t.tbl, CONCAT('missing ', audit.c)
  FROM t CROSS JOIN audit
 WHERE NOT EXISTS (SELECT 1 FROM col WHERE col.tbl = t.tbl AND col.col = audit.c)
UNION ALL SELECT IF(audit.kind = 'b', 'D2', 'D1'), CONCAT(col.tbl, '.', col.col),
       CONCAT('must be NOT NULL ', CASE audit.kind WHEN 't' THEN 'BIGINT' WHEN 'b' THEN 'BOOLEAN' ELSE 'VARCHAR' END,
              ', is ', col.ctype, IF(col.nullable = 'YES', ' NULL', ''))
  FROM col JOIN audit ON audit.c = col.col
 WHERE col.nullable = 'YES'
    OR col.typ <> CASE audit.kind WHEN 't' THEN 'bigint' WHEN 'b' THEN 'tinyint' ELSE 'varchar' END
    OR (audit.kind = 'b' AND col.ctype <> 'tinyint(1)')
UNION ALL SELECT 'D3', CONCAT(tbl, '.', col), CONCAT('flag must be NOT NULL BOOLEAN, is ', ctype, IF(nullable = 'YES', ' NULL', ''))
  FROM col WHERE (col LIKE 'i\_is\_%' OR col LIKE 'i\_has\_%') AND (ctype <> 'tinyint(1)' OR nullable = 'YES')
UNION ALL SELECT 'D4', CONCAT(tbl, '.', col), CONCAT('floating-point type: ', ctype)
  FROM col WHERE typ IN ('float', 'double', 'real')
UNION ALL SELECT 'N6', tc.CONSTRAINT_NAME, CONCAT('constraint name must start with ',
       CASE tc.CONSTRAINT_TYPE WHEN 'FOREIGN KEY' THEN 'fk_' WHEN 'UNIQUE' THEN 'uq_' ELSE 'ck_' END, tc.TABLE_NAME, '_')
  FROM information_schema.TABLE_CONSTRAINTS tc
 WHERE tc.CONSTRAINT_SCHEMA = DATABASE() AND tc.CONSTRAINT_TYPE IN ('FOREIGN KEY', 'UNIQUE', 'CHECK')
   AND tc.TABLE_NAME IN (SELECT tbl FROM t)
   AND LEFT(tc.CONSTRAINT_NAME, CHAR_LENGTH(tc.TABLE_NAME) + 4) <> CONCAT(
       CASE tc.CONSTRAINT_TYPE WHEN 'FOREIGN KEY' THEN 'fk_' WHEN 'UNIQUE' THEN 'uq_' ELSE 'ck_' END, tc.TABLE_NAME, '_')
UNION ALL SELECT 'N6', CONCAT(s.TABLE_NAME, '.', s.INDEX_NAME), CONCAT('index name must start with ix_', s.TABLE_NAME, '_ (or uq_ when unique)')
  FROM (SELECT DISTINCT TABLE_NAME, INDEX_NAME, NON_UNIQUE FROM information_schema.STATISTICS
         WHERE TABLE_SCHEMA = DATABASE() AND INDEX_NAME <> 'PRIMARY') s
 WHERE s.TABLE_NAME IN (SELECT tbl FROM t)
   AND s.INDEX_NAME NOT IN (SELECT CONSTRAINT_NAME FROM information_schema.TABLE_CONSTRAINTS
                             WHERE CONSTRAINT_SCHEMA = DATABASE() AND TABLE_NAME = s.TABLE_NAME AND CONSTRAINT_TYPE = 'UNIQUE')
   AND LEFT(s.INDEX_NAME, CHAR_LENGTH(s.TABLE_NAME) + 4) <> CONCAT(IF(s.NON_UNIQUE = 0, 'uq_', 'ix_'), s.TABLE_NAME, '_')
UNION ALL SELECT 'N6', CONSTRAINT_NAME, 'object name longer than 63'
  FROM information_schema.TABLE_CONSTRAINTS WHERE CONSTRAINT_SCHEMA = DATABASE() AND CHAR_LENGTH(CONSTRAINT_NAME) > 63
ORDER BY 1, 2;
```
