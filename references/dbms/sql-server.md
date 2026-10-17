# Microsoft SQL Server Implementation

Requires SQL Server **2016 or later** (`DATEDIFF_BIG`, `AT TIME ZONE`) or Azure SQL. This file maps the logical standard to SQL Server. It never changes a logical rule.

## 1. Type mapping

| Logical | SQL Server | Notes |
|---|---|---|
| `string(n)` | `NVARCHAR(n)`; `NVARCHAR(MAX)` above 4000 | Unicode (UTF-16); Thai, Japanese and Chinese are 1 unit per character. Database default collation. Literals need the `N` prefix: `N'東京倉庫'` |
| `string(n) ascii` | `VARCHAR(n) COLLATE Latin1_General_100_BIN2` | Binary collation: exact, case-sensitive comparison (K2). Half the size of `NVARCHAR` in keys and indexes |
| `int` | `INT` | |
| `long` | `BIGINT` | |
| `boolean` | `BIT` | `1` = true, `0` = false. Accepts no other values, so no CHECK is needed |
| `decimal(p,s)` | `DECIMAL(p,s)` | Never `MONEY`, `FLOAT` or `REAL` |
| `timestamp` | `BIGINT` | Unix seconds. Never `DATETIME`, `DATETIME2`, `DATE` or the `TIMESTAMP`/`ROWVERSION` type |

Every column in a PK/FK pair uses exactly the same type, length and collation (K3); SQL Server rejects the FK otherwise. Never use `IDENTITY` (K1).

## 2. Names and quoting
- SQL Server stores names in the case they are written. Write them unquoted, in the project case; brackets are never needed because the `m_`/`t_`/`i_` prefixes avoid reserved words.
- With the usual case-insensitive database collation, `m_customer` and `M_CUSTOMER` refer to the same object. A case-sensitive database collation makes the case significant, so write names exactly in the project case everywhere.
- Constraint names must be unique per schema; the `pk_<table>` / `fk_<table>_…` patterns ensure this. Name limit: 128 characters.
- Use `p_` for procedures, never `sp_`.

## 3. Unix time (seconds)

| Task | Expression |
|---|---|
| Now | `DATEDIFF_BIG(SECOND, '19700101', SYSUTCDATETIME())` (rounds down for dates after 1970) |
| UTC `DATETIME2` → Unix | `DATEDIFF_BIG(SECOND, '19700101', @utc_dt)` |
| Unix → UTC `DATETIME2` (safe after 2038) | `DATEADD(SECOND, CAST(@u % 86400 AS INT), DATEADD(DAY, CAST(@u / 86400 AS INT), CAST('19700101' AS DATETIME2(0))))` |
| `DATE` → `_date` value | `DATEDIFF_BIG(SECOND, '19700101', CAST(@d AS DATETIME2(0)))` |
| `_date` value → `DATE` | `DATEADD(DAY, CAST(@u / 86400 AS INT), CAST('19700101' AS DATE))` |
| Start of a local business day | `DATEDIFF_BIG(SECOND, CAST('1970-01-01T00:00:00+00:00' AS DATETIMEOFFSET), CAST(@d AS DATETIME2(0)) AT TIME ZONE 'SE Asia Standard Time')` |
| Business date for an ID | `CONVERT(CHAR(8), CAST(SYSDATETIMEOFFSET() AT TIME ZONE 'SE Asia Standard Time' AS DATE), 112)` |

- `DATEADD` accepts only `INT` seconds. A plain `DATEADD(SECOND, @u, '19700101')` fails after 2038-01-19, so always split the value into days plus seconds as shown.
- `AT TIME ZONE` uses Windows time-zone names. Bangkok is `'SE Asia Standard Time'`; the list is in `sys.time_zone_info`.
- For a local business day, query with `i_x_at >= @start AND i_x_at < @start + 86400`.

## 4. CHECK patterns

SQL Server ignores trailing spaces in `=` comparisons and in `LEN()`. `'cus000001 '` would then collide with `'cus000001'` instead of being rejected, so every ASCII check also compares `DATALENGTH`, which counts trailing spaces:

```sql
-- Fixed-format ID: prefix + digits
CHECK (DATALENGTH(i_customer_id) = 9 AND i_customer_id LIKE 'cus[0-9][0-9][0-9][0-9][0-9][0-9]')
-- Business code: A-Z, a-z, 0-9, _ and -; no spaces, not empty
CHECK (LEN(i_sku) > 0 AND DATALENGTH(i_sku) = LEN(i_sku) AND i_sku NOT LIKE '%[^A-Za-z0-9_-]%')
-- Calendar date
CHECK (i_due_date % 86400 = 0)
```

The ranges `[a-z]` and `[0-9]` follow code-point order only because the column uses a `_BIN2` collation; under a case-insensitive collation `[a-z]` also matches uppercase. Keep the binary collation on every ASCII column.

## 5. Example schema (compliant)

The data-access layer supplies the `func_nm` and `by` audit values on every insert; the timestamps also have database defaults as a safety net.

```sql
-- Counter for IDs (configuration -> master)
CREATE TABLE m_id_seq (
    i_id_seq_id        VARCHAR(20)  COLLATE Latin1_General_100_BIN2 NOT NULL,
    i_last_val         BIGINT       NOT NULL,
    i_created_at       BIGINT       NOT NULL CONSTRAINT df_m_id_seq_created_at DEFAULT (DATEDIFF_BIG(SECOND, '19700101', SYSUTCDATETIME())),
    i_created_func_nm  VARCHAR(100) COLLATE Latin1_General_100_BIN2 NOT NULL,
    i_created_by       VARCHAR(50)  COLLATE Latin1_General_100_BIN2 NOT NULL,
    i_updated_at       BIGINT       NOT NULL CONSTRAINT df_m_id_seq_updated_at DEFAULT (DATEDIFF_BIG(SECOND, '19700101', SYSUTCDATETIME())),
    i_updated_func_nm  VARCHAR(100) COLLATE Latin1_General_100_BIN2 NOT NULL,
    i_updated_by       VARCHAR(50)  COLLATE Latin1_General_100_BIN2 NOT NULL,
    i_active           BIT          NOT NULL CONSTRAINT df_m_id_seq_active DEFAULT (1),
    CONSTRAINT pk_m_id_seq PRIMARY KEY (i_id_seq_id),
    CONSTRAINT ck_m_id_seq_id_seq_id CHECK (LEN(i_id_seq_id) >= 2 AND DATALENGTH(i_id_seq_id) = LEN(i_id_seq_id)
        AND i_id_seq_id LIKE '[a-z][a-z]%' AND i_id_seq_id NOT LIKE '%[^a-z0-9]%'),
    CONSTRAINT ck_m_id_seq_last_val CHECK (i_last_val >= 0)
);

-- Master, ID without a date part: cus + 6 digits
CREATE TABLE m_customer (
    i_customer_id      VARCHAR(20)   COLLATE Latin1_General_100_BIN2 NOT NULL,
    i_customer_nm      NVARCHAR(200) NOT NULL,
    i_tax_no           VARCHAR(30)   COLLATE Latin1_General_100_BIN2 NULL,  -- NULL = not registered for tax
    i_created_at       BIGINT        NOT NULL CONSTRAINT df_m_customer_created_at DEFAULT (DATEDIFF_BIG(SECOND, '19700101', SYSUTCDATETIME())),
    i_created_func_nm  VARCHAR(100)  COLLATE Latin1_General_100_BIN2 NOT NULL,
    i_created_by       VARCHAR(50)   COLLATE Latin1_General_100_BIN2 NOT NULL,
    i_updated_at       BIGINT        NOT NULL CONSTRAINT df_m_customer_updated_at DEFAULT (DATEDIFF_BIG(SECOND, '19700101', SYSUTCDATETIME())),
    i_updated_func_nm  VARCHAR(100)  COLLATE Latin1_General_100_BIN2 NOT NULL,
    i_updated_by       VARCHAR(50)   COLLATE Latin1_General_100_BIN2 NOT NULL,
    i_active           BIT           NOT NULL CONSTRAINT df_m_customer_active DEFAULT (1),
    CONSTRAINT pk_m_customer PRIMARY KEY (i_customer_id),
    CONSTRAINT ck_m_customer_customer_id CHECK (DATALENGTH(i_customer_id) = 9
        AND i_customer_id LIKE 'cus[0-9][0-9][0-9][0-9][0-9][0-9]'),
    CONSTRAINT ck_m_customer_tax_no CHECK (LEN(i_tax_no) > 0 AND DATALENGTH(i_tax_no) = LEN(i_tax_no)
        AND i_tax_no NOT LIKE '%[^A-Za-z0-9_-]%')
);

-- Master, monthly ID: prd + yyyymm + 4 digits; changeable business code in its own column
CREATE TABLE m_product (
    i_product_id       VARCHAR(20)   COLLATE Latin1_General_100_BIN2 NOT NULL,
    i_sku              VARCHAR(30)   COLLATE Latin1_General_100_BIN2 NOT NULL,
    i_product_nm       NVARCHAR(200) NOT NULL,
    i_unit_price       DECIMAL(19,4) NOT NULL,
    i_is_taxable       BIT           NOT NULL CONSTRAINT df_m_product_is_taxable DEFAULT (1),
    i_created_at       BIGINT        NOT NULL CONSTRAINT df_m_product_created_at DEFAULT (DATEDIFF_BIG(SECOND, '19700101', SYSUTCDATETIME())),
    i_created_func_nm  VARCHAR(100)  COLLATE Latin1_General_100_BIN2 NOT NULL,
    i_created_by       VARCHAR(50)   COLLATE Latin1_General_100_BIN2 NOT NULL,
    i_updated_at       BIGINT        NOT NULL CONSTRAINT df_m_product_updated_at DEFAULT (DATEDIFF_BIG(SECOND, '19700101', SYSUTCDATETIME())),
    i_updated_func_nm  VARCHAR(100)  COLLATE Latin1_General_100_BIN2 NOT NULL,
    i_updated_by       VARCHAR(50)   COLLATE Latin1_General_100_BIN2 NOT NULL,
    i_active           BIT           NOT NULL CONSTRAINT df_m_product_active DEFAULT (1),
    CONSTRAINT pk_m_product PRIMARY KEY (i_product_id),
    CONSTRAINT uq_m_product_sku UNIQUE (i_sku),   -- unique for all time (default)
    CONSTRAINT ck_m_product_product_id CHECK (DATALENGTH(i_product_id) = 13
        AND i_product_id LIKE 'prd[0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9]'),
    CONSTRAINT ck_m_product_sku CHECK (LEN(i_sku) > 0 AND DATALENGTH(i_sku) = LEN(i_sku)
        AND i_sku NOT LIKE '%[^A-Za-z0-9_-]%'),
    CONSTRAINT ck_m_product_unit_price CHECK (i_unit_price >= 0)
);

-- Master with times of day: wh + 4 digits
CREATE TABLE m_warehouse (
    i_warehouse_id     VARCHAR(20)   COLLATE Latin1_General_100_BIN2 NOT NULL,
    i_warehouse_nm     NVARCHAR(200) NOT NULL,
    i_open_time        INT           NOT NULL,  -- seconds after local midnight
    i_close_time       INT           NOT NULL,
    i_created_at       BIGINT        NOT NULL CONSTRAINT df_m_warehouse_created_at DEFAULT (DATEDIFF_BIG(SECOND, '19700101', SYSUTCDATETIME())),
    i_created_func_nm  VARCHAR(100)  COLLATE Latin1_General_100_BIN2 NOT NULL,
    i_created_by       VARCHAR(50)   COLLATE Latin1_General_100_BIN2 NOT NULL,
    i_updated_at       BIGINT        NOT NULL CONSTRAINT df_m_warehouse_updated_at DEFAULT (DATEDIFF_BIG(SECOND, '19700101', SYSUTCDATETIME())),
    i_updated_func_nm  VARCHAR(100)  COLLATE Latin1_General_100_BIN2 NOT NULL,
    i_updated_by       VARCHAR(50)   COLLATE Latin1_General_100_BIN2 NOT NULL,
    i_active           BIT           NOT NULL CONSTRAINT df_m_warehouse_active DEFAULT (1),
    CONSTRAINT pk_m_warehouse PRIMARY KEY (i_warehouse_id),
    CONSTRAINT ck_m_warehouse_warehouse_id CHECK (DATALENGTH(i_warehouse_id) = 6
        AND i_warehouse_id LIKE 'wh[0-9][0-9][0-9][0-9]'),
    CONSTRAINT ck_m_warehouse_open_time CHECK (i_open_time BETWEEN 0 AND 86399),
    CONSTRAINT ck_m_warehouse_close_time CHECK (i_close_time BETWEEN 0 AND 86399)
);

-- Header, daily ID: inv + yyyymmdd + 6 digits
CREATE TABLE t_invoice_h (
    i_invoice_id       VARCHAR(20)    COLLATE Latin1_General_100_BIN2 NOT NULL,
    i_invoice_at       BIGINT         NOT NULL,
    i_due_date         BIGINT         NULL,  -- NULL = payable immediately
    i_customer_id      VARCHAR(20)    COLLATE Latin1_General_100_BIN2 NOT NULL,
    i_status           VARCHAR(20)    COLLATE Latin1_General_100_BIN2 NOT NULL CONSTRAINT df_t_invoice_h_status DEFAULT ('draft'),
    i_ccy              VARCHAR(3)     COLLATE Latin1_General_100_BIN2 NOT NULL,
    i_total_amt        DECIMAL(19,4)  NOT NULL CONSTRAINT df_t_invoice_h_total_amt DEFAULT (0),  -- stored total: sum of active lines
    i_note             NVARCHAR(2000) NULL,  -- NULL = no note
    i_created_at       BIGINT         NOT NULL CONSTRAINT df_t_invoice_h_created_at DEFAULT (DATEDIFF_BIG(SECOND, '19700101', SYSUTCDATETIME())),
    i_created_func_nm  VARCHAR(100)   COLLATE Latin1_General_100_BIN2 NOT NULL,
    i_created_by       VARCHAR(50)    COLLATE Latin1_General_100_BIN2 NOT NULL,
    i_updated_at       BIGINT         NOT NULL CONSTRAINT df_t_invoice_h_updated_at DEFAULT (DATEDIFF_BIG(SECOND, '19700101', SYSUTCDATETIME())),
    i_updated_func_nm  VARCHAR(100)   COLLATE Latin1_General_100_BIN2 NOT NULL,
    i_updated_by       VARCHAR(50)    COLLATE Latin1_General_100_BIN2 NOT NULL,
    i_active           BIT            NOT NULL CONSTRAINT df_t_invoice_h_active DEFAULT (1),
    CONSTRAINT pk_t_invoice_h PRIMARY KEY (i_invoice_id),
    CONSTRAINT fk_t_invoice_h_customer FOREIGN KEY (i_customer_id)
        REFERENCES m_customer (i_customer_id) ON DELETE NO ACTION ON UPDATE NO ACTION,
    CONSTRAINT ck_t_invoice_h_invoice_id CHECK (DATALENGTH(i_invoice_id) = 17
        AND i_invoice_id LIKE 'inv[0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9]'),
    CONSTRAINT ck_t_invoice_h_due_date CHECK (i_due_date % 86400 = 0),
    CONSTRAINT ck_t_invoice_h_status CHECK (i_status IN ('draft', 'completed', 'cancelled')),
    CONSTRAINT ck_t_invoice_h_ccy CHECK (DATALENGTH(i_ccy) = 3 AND i_ccy LIKE '[A-Z][A-Z][A-Z]'),
    CONSTRAINT ck_t_invoice_h_total_amt CHECK (i_total_amt >= 0)
);
-- FK index; also serves "a customer's invoices by date"
CREATE INDEX ix_t_invoice_h_customer_id_invoice_at ON t_invoice_h (i_customer_id, i_invoice_at);

-- Detail: composite PK (header ID, i_internal_no)
CREATE TABLE t_invoice_d (
    i_invoice_id       VARCHAR(20)   COLLATE Latin1_General_100_BIN2 NOT NULL,
    i_internal_no      VARCHAR(6)    COLLATE Latin1_General_100_BIN2 NOT NULL,
    i_product_id       VARCHAR(20)   COLLATE Latin1_General_100_BIN2 NOT NULL,
    i_desc             NVARCHAR(500) NOT NULL,  -- snapshot of the product name
    i_qty              DECIMAL(19,4) NOT NULL,
    i_unit_price       DECIMAL(19,4) NOT NULL,  -- snapshot at sale time
    i_tax_rate         DECIMAL(9,6)  NOT NULL,  -- snapshot at sale time
    i_line_amt         DECIMAL(19,4) NOT NULL,
    i_created_at       BIGINT        NOT NULL CONSTRAINT df_t_invoice_d_created_at DEFAULT (DATEDIFF_BIG(SECOND, '19700101', SYSUTCDATETIME())),
    i_created_func_nm  VARCHAR(100)  COLLATE Latin1_General_100_BIN2 NOT NULL,
    i_created_by       VARCHAR(50)   COLLATE Latin1_General_100_BIN2 NOT NULL,
    i_updated_at       BIGINT        NOT NULL CONSTRAINT df_t_invoice_d_updated_at DEFAULT (DATEDIFF_BIG(SECOND, '19700101', SYSUTCDATETIME())),
    i_updated_func_nm  VARCHAR(100)  COLLATE Latin1_General_100_BIN2 NOT NULL,
    i_updated_by       VARCHAR(50)   COLLATE Latin1_General_100_BIN2 NOT NULL,
    i_active           BIT           NOT NULL CONSTRAINT df_t_invoice_d_active DEFAULT (1),
    CONSTRAINT pk_t_invoice_d PRIMARY KEY (i_invoice_id, i_internal_no),
    CONSTRAINT fk_t_invoice_d_invoice_h FOREIGN KEY (i_invoice_id)
        REFERENCES t_invoice_h (i_invoice_id) ON DELETE NO ACTION ON UPDATE NO ACTION,
    CONSTRAINT fk_t_invoice_d_product FOREIGN KEY (i_product_id)
        REFERENCES m_product (i_product_id) ON DELETE NO ACTION ON UPDATE NO ACTION,
    CONSTRAINT ck_t_invoice_d_internal_no CHECK (DATALENGTH(i_internal_no) = 6
        AND i_internal_no LIKE '[0-9][0-9][0-9][0-9][0-9][0-9]' AND i_internal_no <> '000000'),
    CONSTRAINT ck_t_invoice_d_qty CHECK (i_qty > 0),
    CONSTRAINT ck_t_invoice_d_unit_price CHECK (i_unit_price >= 0),
    CONSTRAINT ck_t_invoice_d_tax_rate CHECK (i_tax_rate BETWEEN 0 AND 1)
);
-- FK to the header is covered by the PK's leading column; FK to the product needs its own index
CREATE INDEX ix_t_invoice_d_product_id ON t_invoice_d (i_product_id);

-- Standalone transaction: one record per event, role FKs to the same master
CREATE TABLE t_stock_move (
    i_stock_move_id      VARCHAR(20)   COLLATE Latin1_General_100_BIN2 NOT NULL,
    i_move_at            BIGINT        NOT NULL,
    i_product_id         VARCHAR(20)   COLLATE Latin1_General_100_BIN2 NOT NULL,
    i_from_warehouse_id  VARCHAR(20)   COLLATE Latin1_General_100_BIN2 NULL,  -- NULL = received from outside
    i_to_warehouse_id    VARCHAR(20)   COLLATE Latin1_General_100_BIN2 NULL,  -- NULL = shipped out
    i_qty                DECIMAL(19,4) NOT NULL,
    i_created_at         BIGINT        NOT NULL CONSTRAINT df_t_stock_move_created_at DEFAULT (DATEDIFF_BIG(SECOND, '19700101', SYSUTCDATETIME())),
    i_created_func_nm    VARCHAR(100)  COLLATE Latin1_General_100_BIN2 NOT NULL,
    i_created_by         VARCHAR(50)   COLLATE Latin1_General_100_BIN2 NOT NULL,
    i_updated_at         BIGINT        NOT NULL CONSTRAINT df_t_stock_move_updated_at DEFAULT (DATEDIFF_BIG(SECOND, '19700101', SYSUTCDATETIME())),
    i_updated_func_nm    VARCHAR(100)  COLLATE Latin1_General_100_BIN2 NOT NULL,
    i_updated_by         VARCHAR(50)   COLLATE Latin1_General_100_BIN2 NOT NULL,
    i_active             BIT           NOT NULL CONSTRAINT df_t_stock_move_active DEFAULT (1),
    CONSTRAINT pk_t_stock_move PRIMARY KEY (i_stock_move_id),
    CONSTRAINT fk_t_stock_move_product FOREIGN KEY (i_product_id)
        REFERENCES m_product (i_product_id) ON DELETE NO ACTION ON UPDATE NO ACTION,
    CONSTRAINT fk_t_stock_move_from_warehouse FOREIGN KEY (i_from_warehouse_id)
        REFERENCES m_warehouse (i_warehouse_id) ON DELETE NO ACTION ON UPDATE NO ACTION,
    CONSTRAINT fk_t_stock_move_to_warehouse FOREIGN KEY (i_to_warehouse_id)
        REFERENCES m_warehouse (i_warehouse_id) ON DELETE NO ACTION ON UPDATE NO ACTION,
    CONSTRAINT ck_t_stock_move_stock_move_id CHECK (DATALENGTH(i_stock_move_id) = 17
        AND i_stock_move_id LIKE 'smv[0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9]'),
    CONSTRAINT ck_t_stock_move_qty CHECK (i_qty > 0),
    CONSTRAINT ck_t_stock_move_warehouse CHECK (
        (i_from_warehouse_id IS NOT NULL OR i_to_warehouse_id IS NOT NULL)
        AND (i_from_warehouse_id IS NULL OR i_to_warehouse_id IS NULL OR i_from_warehouse_id <> i_to_warehouse_id))
);
CREATE INDEX ix_t_stock_move_product_id_move_at ON t_stock_move (i_product_id, i_move_at);
CREATE INDEX ix_t_stock_move_from_warehouse_id ON t_stock_move (i_from_warehouse_id);
CREATE INDEX ix_t_stock_move_to_warehouse_id ON t_stock_move (i_to_warehouse_id);
GO

-- Default read path: active rows only (D2)
CREATE VIEW v_m_product AS SELECT * FROM m_product WHERE i_active = 1;
GO
```

## 6. ID generation and documents

### ID from the counter table
Run the procedure inside the caller's transaction, so the counter and the row that uses the ID commit or roll back together. `CREATE PROCEDURE` must be the only statement in its batch, so scripts put `GO` before and after it.

```sql
CREATE PROCEDURE p_next_id
    @abbr    VARCHAR(4),
    @date    VARCHAR(8),    -- 'yyyymmdd', 'yyyymm' or '' in the business time zone
    @digits  TINYINT,
    @func_nm VARCHAR(100),
    @by      VARCHAR(50),
    @id      VARCHAR(20) OUTPUT
AS
BEGIN
    SET NOCOUNT ON;
    SET XACT_ABORT ON;
    IF @@TRANCOUNT = 0
        THROW 50000, 'p_next_id must run inside the caller''s transaction.', 1;

    DECLARE @seq_id VARCHAR(20) = @abbr + @date;
    DECLARE @now    BIGINT      = DATEDIFF_BIG(SECOND, '19700101', SYSUTCDATETIME());
    DECLARE @n      BIGINT;

    -- UPDLOCK + HOLDLOCK also locks the key when the row does not exist yet,
    -- so two sessions cannot both insert the first row for a period.
    UPDATE m_id_seq WITH (UPDLOCK, HOLDLOCK)
       SET @n = i_last_val = i_last_val + 1,
           i_updated_at = @now, i_updated_func_nm = @func_nm, i_updated_by = @by
     WHERE i_id_seq_id = @seq_id;

    IF @@ROWCOUNT = 0
    BEGIN
        SET @n = 1;
        INSERT INTO m_id_seq (i_id_seq_id, i_last_val,
                              i_created_at, i_created_func_nm, i_created_by,
                              i_updated_at, i_updated_func_nm, i_updated_by)
        VALUES (@seq_id, @n, @now, @func_nm, @by, @now, @func_nm, @by);
    END;

    IF @n >= POWER(CAST(10 AS BIGINT), @digits)
        THROW 50001, 'ID counter exhausted for this abbreviation and period.', 1;

    SET @id = @seq_id + RIGHT(REPLICATE('0', @digits) + CAST(@n AS VARCHAR(19)), @digits);
END;
GO
```

### Master rows
```sql
SET XACT_ABORT ON;
BEGIN TRANSACTION;
DECLARE @id VARCHAR(20);
DECLARE @month VARCHAR(6) = LEFT(CONVERT(CHAR(8), CAST(SYSDATETIMEOFFSET() AT TIME ZONE 'SE Asia Standard Time' AS DATE), 112), 6);

EXEC p_next_id 'cus', '', 6, 'customer.create', 'usr000001', @id OUTPUT;
INSERT INTO m_customer (i_customer_id, i_customer_nm,
                        i_created_func_nm, i_created_by, i_updated_func_nm, i_updated_by)
VALUES (@id, N'บริษัท ตัวอย่าง จำกัด', 'customer.create', 'usr000001', 'customer.create', 'usr000001');

EXEC p_next_id 'prd', @month, 4, 'product.create', 'usr000001', @id OUTPUT;
INSERT INTO m_product (i_product_id, i_sku, i_product_nm, i_unit_price,
                       i_created_func_nm, i_created_by, i_updated_func_nm, i_updated_by)
VALUES (@id, 'SKU-001', N'商品 A', 100, 'product.create', 'usr000001', 'product.create', 'usr000001');

EXEC p_next_id 'wh', '', 4, 'warehouse.create', 'usr000001', @id OUTPUT;
INSERT INTO m_warehouse (i_warehouse_id, i_warehouse_nm, i_open_time, i_close_time,
                         i_created_func_nm, i_created_by, i_updated_func_nm, i_updated_by)
VALUES (@id, N'東京倉庫', 30600, 64800, 'warehouse.create', 'usr000001', 'warehouse.create', 'usr000001');
COMMIT TRANSACTION;
GO
```

**Optional sequence** for an ID without a date part (gaps after a rollback are acceptable):

```sql
CREATE SEQUENCE sq_m_customer AS INT START WITH 1 MINVALUE 1 MAXVALUE 999999 NO CYCLE;
-- 'cus' + RIGHT('000000' + CAST(NEXT VALUE FOR sq_m_customer AS VARCHAR(6)), 6)
```

### Creating a Header/Detail document
```sql
SET XACT_ABORT ON;
BEGIN TRANSACTION;

DECLARE @invoice_id VARCHAR(20);
DECLARE @func VARCHAR(100) = 'invoice.create', @by VARCHAR(50) = 'usr000001';
DECLARE @now  BIGINT = DATEDIFF_BIG(SECOND, '19700101', SYSUTCDATETIME());
DECLARE @day  VARCHAR(8) = CONVERT(CHAR(8), CAST(SYSDATETIMEOFFSET() AT TIME ZONE 'SE Asia Standard Time' AS DATE), 112);

EXEC p_next_id 'inv', @day, 6, @func, @by, @invoice_id OUTPUT;

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
SELECT TOP (1) @invoice_id, '000001', p.i_product_id, p.i_product_nm,
       2, p.i_unit_price, 0.07, 2 * p.i_unit_price,
       @now, @func, @by, @now, @func, @by
  FROM v_m_product p
 ORDER BY p.i_product_id;

-- Recalculate the stored total from active lines, in the same transaction
UPDATE t_invoice_h
   SET i_total_amt = (SELECT ISNULL(SUM(i_line_amt), 0) FROM t_invoice_d
                       WHERE i_invoice_id = @invoice_id AND i_active = 1),
       i_updated_at = @now, i_updated_func_nm = @func, i_updated_by = @by
 WHERE i_invoice_id = @invoice_id;

COMMIT TRANSACTION;
GO
```

### Adding a line to an existing document
```sql
SET XACT_ABORT ON;
BEGIN TRANSACTION;

DECLARE @invoice_id VARCHAR(20) = (SELECT MIN(i_invoice_id) FROM t_invoice_h);
DECLARE @locked VARCHAR(20), @next INT;

-- 1. Lock the header first (fixed lock order: header, then details)
SELECT @locked = i_invoice_id
  FROM t_invoice_h WITH (UPDLOCK, HOLDLOCK)
 WHERE i_invoice_id = @invoice_id AND i_status = 'draft' AND i_active = 1;
IF @locked IS NULL
    THROW 50002, 'Invoice not found, not active or not in draft status.', 1;

-- 2. Next line number = highest + 1, counting inactive lines too (numbers are never reused)
SELECT @next = ISNULL(MAX(CAST(i_internal_no AS INT)), 0) + 1
  FROM t_invoice_d
 WHERE i_invoice_id = @invoice_id;
IF @next > 999999
    THROW 50003, 'Line number limit reached.', 1;

INSERT INTO t_invoice_d (i_invoice_id, i_internal_no, i_product_id, i_desc,
                         i_qty, i_unit_price, i_tax_rate, i_line_amt,
                         i_created_func_nm, i_created_by, i_updated_func_nm, i_updated_by)
VALUES (@invoice_id, RIGHT('000000' + CAST(@next AS VARCHAR(7)), 6),
        (SELECT MIN(i_product_id) FROM v_m_product), N'Product B', 1, 150, 0.07, 150,
        'invoice.add_line', 'usr000001', 'invoice.add_line', 'usr000001');

COMMIT TRANSACTION;
GO
```

### Removing a line (soft delete)
```sql
UPDATE t_invoice_d
   SET i_active = 0,
       i_updated_at = DATEDIFF_BIG(SECOND, '19700101', SYSUTCDATETIME()),
       i_updated_func_nm = 'invoice.remove_line', i_updated_by = 'usr000001'
 WHERE i_invoice_id = (SELECT MIN(i_invoice_id) FROM t_invoice_h) AND i_internal_no = '000002';
GO
```
Recalculate the header total in the same transaction, as above.

## 7. Unique codes among active rows only
To let a code be reused after its row is deactivated, replace the plain unique constraint with a filtered unique index:

```sql
ALTER TABLE m_product DROP CONSTRAINT uq_m_product_sku;
CREATE UNIQUE INDEX uq_m_product_sku ON m_product (i_sku) WHERE i_active = 1;
GO
```
Every session that writes to the table needs `SET ANSI_NULLS ON` and `QUOTED_IDENTIFIER ON`, which are the defaults.

## 8. Performance notes
- **The PK is clustered by default.** Abbreviation + date + counter IDs increase in insert order, so new rows go at the end of the index. Random UUID strings would scatter inserts and fragment the index.
- **Indexes are not created for FKs automatically.** Create the `ix_` indexes yourself (K6).
- **Send ID parameters as `VARCHAR`, not `NVARCHAR`.** Many drivers send strings as `NVARCHAR` by default. Comparing an `NVARCHAR` parameter with a `VARCHAR` ID column forces a conversion that can turn an index seek into a full scan. Set the parameter type explicitly (.NET: `SqlDbType.VarChar`). JDBC's `sendStringParametersAsUnicode=false` applies to every string parameter, including Unicode text, so set the type per parameter instead.
- **Send text parameters as `NVARCHAR`.** A Thai or Japanese value sent as `VARCHAR` is converted to `?` before it reaches the column.
- **Consider `READ_COMMITTED_SNAPSHOT ON`** so readers don't block writers. Writers still lock as shown above.
- **Range queries on `_at` integers use the index.** Never wrap the column in `DATEADD` or other functions inside a `WHERE`.

## 9. Catalog audit query

This query lists violations of the rules that can be checked from the catalog: N1, N2, N4, N5, N6, K1, K2, K3, K5, K6, U1, D1, D2, D3 and D4. Run it in the database to check; it covers the caller's default schema. An empty result means every automated check passed; the remaining rules need the manual review in `../schema-validation.md`. The name pattern is for a lowercase project; in an uppercase project replace `a-z` with `A-Z` and the prefixes with their uppercase form.

```sql
WITH t AS (
    SELECT object_id, CAST(name AS NVARCHAR(128)) COLLATE Latin1_General_100_BIN2 AS tbl
      FROM sys.tables
     WHERE is_ms_shipped = 0 AND schema_id = SCHEMA_ID()
), cols AS (
    SELECT t.object_id, t.tbl, CAST(c.name AS NVARCHAR(128)) COLLATE Latin1_General_100_BIN2 AS col,
           c.max_length, c.is_nullable, c.collation_name,
           CAST(ty.name AS NVARCHAR(128)) COLLATE Latin1_General_100_BIN2 AS typ,
           CASE WHEN EXISTS (SELECT 1 FROM sys.index_columns ic
                               JOIN sys.indexes i ON i.object_id = ic.object_id AND i.index_id = ic.index_id
                              WHERE i.is_primary_key = 1 AND ic.object_id = c.object_id AND ic.column_id = c.column_id)
                  OR EXISTS (SELECT 1 FROM sys.foreign_key_columns fkc
                              WHERE fkc.parent_object_id = c.object_id AND fkc.parent_column_id = c.column_id)
                THEN 1 ELSE 0 END AS is_key
      FROM t
      JOIN sys.columns c ON c.object_id = t.object_id
      JOIN sys.types ty ON ty.user_type_id = c.user_type_id
), audit AS (
    SELECT c, kind FROM (VALUES ('i_created_at', 'bigint'), ('i_created_func_nm', 'varchar'), ('i_created_by', 'varchar'),
                                ('i_updated_at', 'bigint'), ('i_updated_func_nm', 'varchar'), ('i_updated_by', 'varchar'),
                                ('i_active', 'bit')) AS v(c, kind)
), obj AS (   -- constraints and indexes with their expected prefix
    SELECT t.tbl, CAST(k.name AS NVARCHAR(128)) COLLATE Latin1_General_100_BIN2 AS name,
           CASE k.type WHEN 'PK' THEN 'pk_' ELSE 'uq_' END AS pfx, CAST(k.type AS VARCHAR(2)) AS kind
      FROM sys.key_constraints k JOIN t ON t.object_id = k.parent_object_id
    UNION ALL
    SELECT t.tbl, CAST(f.name AS NVARCHAR(128)) COLLATE Latin1_General_100_BIN2, 'fk_', 'F'
      FROM sys.foreign_keys f JOIN t ON t.object_id = f.parent_object_id
    UNION ALL
    SELECT t.tbl, CAST(c.name AS NVARCHAR(128)) COLLATE Latin1_General_100_BIN2, 'ck_', 'C'
      FROM sys.check_constraints c JOIN t ON t.object_id = c.parent_object_id
    UNION ALL
    SELECT t.tbl, CAST(d.name AS NVARCHAR(128)) COLLATE Latin1_General_100_BIN2, 'df_', 'D'
      FROM sys.default_constraints d JOIN t ON t.object_id = d.parent_object_id
    UNION ALL
    SELECT t.tbl, CAST(i.name AS NVARCHAR(128)) COLLATE Latin1_General_100_BIN2,
           CASE WHEN i.is_unique = 1 THEN 'uq_' ELSE 'ix_' END, 'I'
      FROM sys.indexes i JOIN t ON t.object_id = i.object_id
     WHERE i.index_id > 0 AND i.is_primary_key = 0 AND i.is_unique_constraint = 0
)
SELECT 'N1' AS rule_id, tbl AS object_name, 'name must be a-z0-9_, start with a letter, no __ or trailing _' AS problem
  FROM t WHERE tbl LIKE '%[^a-z0-9_]%' OR tbl NOT LIKE '[a-z]%' OR tbl LIKE '%[_][_]%' OR tbl LIKE '%[_]'
UNION ALL SELECT 'N2', tbl, 'table must start with m_ or t_' FROM t WHERE LEFT(tbl, 2) NOT IN ('m_', 't_')
UNION ALL SELECT 'N5', tbl, 'table name longer than 30' FROM t WHERE LEN(tbl) > 30
UNION ALL SELECT 'N1', tbl + '.' + col, 'name must be a-z0-9_, start with a letter, no __ or trailing _'
  FROM cols WHERE col LIKE '%[^a-z0-9_]%' OR col NOT LIKE '[a-z]%' OR col LIKE '%[_][_]%' OR col LIKE '%[_]'
UNION ALL SELECT 'N4', tbl + '.' + col, 'column must start with i_' FROM cols WHERE LEFT(col, 2) <> 'i_'
UNION ALL SELECT 'N5', tbl + '.' + col, 'column name longer than 30' FROM cols WHERE LEN(col) > 30
UNION ALL SELECT 'K1', tbl + '.' + col, 'key column must be VARCHAR, is ' + typ
  FROM cols WHERE is_key = 1 AND typ <> 'varchar'
UNION ALL SELECT 'K2', tbl + '.' + col, 'key column must use Latin1_General_100_BIN2'
  FROM cols WHERE is_key = 1 AND typ = 'varchar' AND ISNULL(collation_name, '') <> 'Latin1_General_100_BIN2'
UNION ALL SELECT 'K3', CAST(f.name AS NVARCHAR(128)) COLLATE Latin1_General_100_BIN2,
       'FK column ' + CAST(pc.name AS NVARCHAR(128)) COLLATE Latin1_General_100_BIN2 + ' differs from '
       + CAST(rc.name AS NVARCHAR(128)) COLLATE Latin1_General_100_BIN2 + ' in type, length or collation'
  FROM sys.foreign_keys f
  JOIN t ON t.object_id = f.parent_object_id
  JOIN sys.foreign_key_columns fkc ON fkc.constraint_object_id = f.object_id
  JOIN sys.columns pc ON pc.object_id = fkc.parent_object_id AND pc.column_id = fkc.parent_column_id
  JOIN sys.columns rc ON rc.object_id = fkc.referenced_object_id AND rc.column_id = fkc.referenced_column_id
 WHERE pc.user_type_id <> rc.user_type_id OR pc.max_length <> rc.max_length
    OR ISNULL(pc.collation_name, '') <> ISNULL(rc.collation_name, '')
UNION ALL SELECT 'K5', CAST(f.name AS NVARCHAR(128)) COLLATE Latin1_General_100_BIN2, 'FK actions must be NO ACTION'
  FROM sys.foreign_keys f JOIN t ON t.object_id = f.parent_object_id
 WHERE f.delete_referential_action <> 0 OR f.update_referential_action <> 0
UNION ALL SELECT 'K6', CAST(f.name AS NVARCHAR(128)) COLLATE Latin1_General_100_BIN2, 'FK columns are not the leading columns of any index'
  FROM sys.foreign_keys f JOIN t ON t.object_id = f.parent_object_id
 WHERE NOT EXISTS (
        SELECT 1 FROM sys.indexes i
         WHERE i.object_id = f.parent_object_id AND i.index_id > 0 AND i.has_filter = 0
           AND NOT EXISTS (
                SELECT 1 FROM sys.foreign_key_columns fkc
                 WHERE fkc.constraint_object_id = f.object_id
                   AND NOT EXISTS (
                        SELECT 1 FROM sys.index_columns ic
                         WHERE ic.object_id = i.object_id AND ic.index_id = i.index_id
                           AND ic.column_id = fkc.parent_column_id
                           AND ic.key_ordinal BETWEEN 1 AND (SELECT COUNT(*) FROM sys.foreign_key_columns x
                                                              WHERE x.constraint_object_id = f.object_id))))
UNION ALL SELECT 'U1', tbl + '.' + col, 'date/time type not allowed: ' + typ
  FROM cols WHERE typ IN ('date', 'datetime', 'datetime2', 'smalldatetime', 'datetimeoffset', 'time', 'timestamp')
UNION ALL SELECT 'U1', tbl + '.' + col, '_at/_date column must be BIGINT, is ' + typ
  FROM cols WHERE (col LIKE '%[_]at' OR col LIKE '%[_]date') AND typ <> 'bigint'
UNION ALL SELECT CASE WHEN a.c = 'i_active' THEN 'D2' ELSE 'D1' END, t.tbl, 'missing ' + a.c
  FROM t CROSS JOIN audit a
 WHERE NOT EXISTS (SELECT 1 FROM cols WHERE cols.object_id = t.object_id AND cols.col = a.c)
UNION ALL SELECT CASE WHEN a.c = 'i_active' THEN 'D2' ELSE 'D1' END, cols.tbl + '.' + cols.col,
       'must be NOT NULL ' + a.kind + ', is ' + cols.typ + CASE WHEN cols.is_nullable = 1 THEN ' NULL' ELSE '' END
  FROM cols JOIN audit a ON a.c = cols.col
 WHERE cols.is_nullable = 1 OR cols.typ <> a.kind
UNION ALL SELECT 'D3', tbl + '.' + col, 'flag must be NOT NULL BIT, is ' + typ + CASE WHEN is_nullable = 1 THEN ' NULL' ELSE '' END
  FROM cols WHERE (col LIKE 'i[_]is[_]%' OR col LIKE 'i[_]has[_]%') AND (typ <> 'bit' OR is_nullable = 1)
UNION ALL SELECT 'D4', tbl + '.' + col, 'floating-point or money type: ' + typ
  FROM cols WHERE typ IN ('float', 'real', 'money', 'smallmoney')
UNION ALL SELECT 'N6', name, 'name must start with ' + pfx + tbl + CASE WHEN kind = 'PK' THEN '' ELSE '_' END
  FROM obj
 WHERE (kind = 'PK' AND name <> pfx + tbl)
    OR (kind <> 'PK' AND LEFT(name, LEN(tbl) + 4) <> pfx + tbl + '_')
UNION ALL SELECT 'N6', name, 'object name longer than 63' FROM obj WHERE LEN(name) > 63
ORDER BY 1, 2;
```
