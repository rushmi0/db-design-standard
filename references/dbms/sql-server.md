# Microsoft SQL Server Implementation

Requires SQL Server **2016 or later** (`DATEDIFF_BIG`, `AT TIME ZONE`) or Azure SQL. This file maps the logical standard to SQL Server. It never changes a logical rule.

## 1. Type mapping

| Logical | SQL Server | Notes |
|---|---|---|
| `ID(n)` | `VARCHAR(n) COLLATE Latin1_General_100_BIN2` | Binary collation: exact, case-sensitive comparison (K2) |
| `LINENO` | `VARCHAR(6) COLLATE Latin1_General_100_BIN2` | |
| `CODE(n)` | `VARCHAR(n) COLLATE Latin1_General_100_BIN2` | Rejects lowercase in CHECKs and `IN` lists |
| `TEXT(n)` | `NVARCHAR(n)` (n ≤ 4000), else `NVARCHAR(MAX)` | Unicode, counted in UTF-16 units. Thai and other common scripts are 1 unit per character. Database default collation |
| `FLAG` | `CHAR(1) COLLATE Latin1_General_100_BIN2` + CHECK `IN ('Y','N')` | |
| `UNIXTIME`, `UNIXDATE` | `BIGINT` | |
| `TIMEOFDAY` | `INT` + CHECK `BETWEEN 0 AND 86399` | |
| `DURATION` | `BIGINT` + CHECK `>= 0` | |
| `AMOUNT`, `QTY` | `DECIMAL(19,4)` | Never `MONEY`, `FLOAT` or `REAL` |
| `RATE` | `DECIMAL(9,6)` | |
| `COUNT` | `INT` | Never a key |

Every column that takes part in a PK/FK pair must use exactly the same type, length and collation (K3). SQL Server rejects an FK when they differ.

## 2. Names
- SQL Server stores names in the case they are written, so write all DDL and queries in uppercase. Brackets are never needed, because the `M_`/`T_`/`I_` prefixes avoid reserved words.
- Constraint names must be unique per schema; the `PK_<TABLE>` / `FK_<TABLE>_…` patterns ensure this.
- Use `P_` for procedures, never `SP_`.

## 3. Unix time (seconds)

| Task | Expression |
|---|---|
| Now | `DATEDIFF_BIG(SECOND, '19700101', SYSUTCDATETIME())` (rounds down for dates after 1970) |
| UTC `DATETIME2` → Unix | `DATEDIFF_BIG(SECOND, '19700101', @utc_dt)` |
| Unix → UTC `DATETIME2` (safe after 2038) | `DATEADD(SECOND, CAST(@u % 86400 AS INT), DATEADD(DAY, CAST(@u / 86400 AS INT), CAST('19700101' AS DATETIME2(0))))` |
| `DATE` → `_DATE` value | `DATEDIFF_BIG(SECOND, '19700101', CAST(@d AS DATETIME2(0)))` |
| `_DATE` value → `DATE` | `DATEADD(DAY, CAST(@u / 86400 AS INT), CAST('19700101' AS DATE))` |
| Start of a local business day | `DATEDIFF_BIG(SECOND, CAST('1970-01-01T00:00:00+00:00' AS DATETIMEOFFSET), CAST(@d AS DATETIME2(0)) AT TIME ZONE 'SE Asia Standard Time')` |

- `DATEADD` accepts only `INT` seconds. A plain `DATEADD(SECOND, @u, '19700101')` fails after 2038-01-19, so always split the value into days plus seconds as shown.
- `AT TIME ZONE` uses Windows time-zone names. Bangkok is `'SE Asia Standard Time'`; the list is in `sys.time_zone_info`.
- For a local business day, query with `I_X_AT >= @start AND I_X_AT < @start + 86400`.

## 4. CHECK patterns

SQL Server ignores trailing spaces in `=` comparisons and in `LEN()`. That means `'CUS000001 '` would collide with `'CUS000001'` instead of being rejected, so every ID and CODE check also compares `DATALENGTH` (which counts trailing spaces) with the expected length:

```sql
-- Fixed-format ID: prefix + digits
CHECK (DATALENGTH(I_CUSTOMER_ID) = 9 AND I_CUSTOMER_ID LIKE 'CUS[0-9][0-9][0-9][0-9][0-9][0-9]')
-- Any ID: A-Z, 0-9 only, no spaces, not empty
CHECK (LEN(I_DOC_SEQ_ID) > 0 AND DATALENGTH(I_DOC_SEQ_ID) = LEN(I_DOC_SEQ_ID) AND I_DOC_SEQ_ID NOT LIKE '%[^A-Z0-9]%')
-- Code: A-Z, 0-9, _ and -
CHECK (LEN(I_SKU) > 0 AND DATALENGTH(I_SKU) = LEN(I_SKU) AND I_SKU NOT LIKE '%[^A-Z0-9_-]%')
-- Calendar date
CHECK (I_DUE_DATE % 86400 = 0)
```

The ranges `[A-Z]` and `[0-9]` follow code-point order only because the column uses a `_BIN2` collation. Keep that collation on every ID and CODE column.

## 5. Example schema (compliant)

This example system has no user accounts, so the `_BY` audit columns are omitted (D1).

```sql
CREATE SEQUENCE SQ_M_CUSTOMER  AS INT START WITH 1 MINVALUE 1 MAXVALUE 999999 NO CYCLE;
CREATE SEQUENCE SQ_M_PRODUCT   AS INT START WITH 1 MINVALUE 1 MAXVALUE 999999 NO CYCLE;
CREATE SEQUENCE SQ_M_WAREHOUSE AS INT START WITH 1 MINVALUE 1 MAXVALUE 999999 NO CYCLE;

-- Counter for document IDs (configuration -> master)
CREATE TABLE M_DOC_SEQ (
    I_DOC_SEQ_ID  VARCHAR(20) COLLATE Latin1_General_100_BIN2 NOT NULL,
    I_LAST_VAL    INT    NOT NULL,
    I_CREATED_AT  BIGINT NOT NULL CONSTRAINT DF_M_DOC_SEQ_CREATED_AT DEFAULT (DATEDIFF_BIG(SECOND, '19700101', SYSUTCDATETIME())),
    I_UPDATED_AT  BIGINT NOT NULL CONSTRAINT DF_M_DOC_SEQ_UPDATED_AT DEFAULT (DATEDIFF_BIG(SECOND, '19700101', SYSUTCDATETIME())),
    CONSTRAINT PK_M_DOC_SEQ PRIMARY KEY (I_DOC_SEQ_ID),
    CONSTRAINT CK_M_DOC_SEQ_DOC_SEQ_ID CHECK (LEN(I_DOC_SEQ_ID) > 0 AND DATALENGTH(I_DOC_SEQ_ID) = LEN(I_DOC_SEQ_ID) AND I_DOC_SEQ_ID NOT LIKE '%[^A-Z0-9]%'),
    CONSTRAINT CK_M_DOC_SEQ_LAST_VAL CHECK (I_LAST_VAL >= 0)
);

-- Master
CREATE TABLE M_CUSTOMER (
    I_CUSTOMER_ID    VARCHAR(20)   COLLATE Latin1_General_100_BIN2 NOT NULL,
    I_CUSTOMER_NAME  NVARCHAR(200) NOT NULL,
    I_TAX_NO         VARCHAR(30)   COLLATE Latin1_General_100_BIN2 NULL,  -- NULL = not registered for tax
    I_IS_ACTIVE      CHAR(1)       COLLATE Latin1_General_100_BIN2 NOT NULL CONSTRAINT DF_M_CUSTOMER_IS_ACTIVE DEFAULT ('Y'),
    I_CREATED_AT     BIGINT NOT NULL CONSTRAINT DF_M_CUSTOMER_CREATED_AT DEFAULT (DATEDIFF_BIG(SECOND, '19700101', SYSUTCDATETIME())),
    I_UPDATED_AT     BIGINT NOT NULL CONSTRAINT DF_M_CUSTOMER_UPDATED_AT DEFAULT (DATEDIFF_BIG(SECOND, '19700101', SYSUTCDATETIME())),
    CONSTRAINT PK_M_CUSTOMER PRIMARY KEY (I_CUSTOMER_ID),
    CONSTRAINT CK_M_CUSTOMER_CUSTOMER_ID CHECK (DATALENGTH(I_CUSTOMER_ID) = 9 AND I_CUSTOMER_ID LIKE 'CUS[0-9][0-9][0-9][0-9][0-9][0-9]'),
    CONSTRAINT CK_M_CUSTOMER_TAX_NO CHECK (LEN(I_TAX_NO) > 0 AND DATALENGTH(I_TAX_NO) = LEN(I_TAX_NO) AND I_TAX_NO NOT LIKE '%[^A-Z0-9_-]%'),
    CONSTRAINT CK_M_CUSTOMER_IS_ACTIVE CHECK (I_IS_ACTIVE IN ('Y', 'N'))
);

-- Master with soft delete and a changeable business code
CREATE TABLE M_PRODUCT (
    I_PRODUCT_ID    VARCHAR(20)   COLLATE Latin1_General_100_BIN2 NOT NULL,
    I_SKU           VARCHAR(30)   COLLATE Latin1_General_100_BIN2 NOT NULL,
    I_PRODUCT_NAME  NVARCHAR(200) NOT NULL,
    I_UNIT_PRICE    DECIMAL(19,4) NOT NULL,
    I_CREATED_AT    BIGINT NOT NULL CONSTRAINT DF_M_PRODUCT_CREATED_AT DEFAULT (DATEDIFF_BIG(SECOND, '19700101', SYSUTCDATETIME())),
    I_UPDATED_AT    BIGINT NOT NULL CONSTRAINT DF_M_PRODUCT_UPDATED_AT DEFAULT (DATEDIFF_BIG(SECOND, '19700101', SYSUTCDATETIME())),
    I_DELETED_AT    BIGINT NULL,  -- NULL = active
    CONSTRAINT PK_M_PRODUCT PRIMARY KEY (I_PRODUCT_ID),
    CONSTRAINT CK_M_PRODUCT_PRODUCT_ID CHECK (DATALENGTH(I_PRODUCT_ID) = 9 AND I_PRODUCT_ID LIKE 'PRD[0-9][0-9][0-9][0-9][0-9][0-9]'),
    CONSTRAINT CK_M_PRODUCT_SKU CHECK (LEN(I_SKU) > 0 AND DATALENGTH(I_SKU) = LEN(I_SKU) AND I_SKU NOT LIKE '%[^A-Z0-9_-]%'),
    CONSTRAINT CK_M_PRODUCT_UNIT_PRICE CHECK (I_UNIT_PRICE >= 0)
);
-- SKU unique among active products only
CREATE UNIQUE INDEX UQ_M_PRODUCT_SKU ON M_PRODUCT (I_SKU) WHERE I_DELETED_AT IS NULL;

CREATE TABLE M_WAREHOUSE (
    I_WAREHOUSE_ID    VARCHAR(20)   COLLATE Latin1_General_100_BIN2 NOT NULL,
    I_WAREHOUSE_NAME  NVARCHAR(200) NOT NULL,
    I_OPEN_TIME       INT NOT NULL,   -- seconds after local midnight
    I_CLOSE_TIME      INT NOT NULL,
    I_CREATED_AT      BIGINT NOT NULL CONSTRAINT DF_M_WAREHOUSE_CREATED_AT DEFAULT (DATEDIFF_BIG(SECOND, '19700101', SYSUTCDATETIME())),
    I_UPDATED_AT      BIGINT NOT NULL CONSTRAINT DF_M_WAREHOUSE_UPDATED_AT DEFAULT (DATEDIFF_BIG(SECOND, '19700101', SYSUTCDATETIME())),
    CONSTRAINT PK_M_WAREHOUSE PRIMARY KEY (I_WAREHOUSE_ID),
    CONSTRAINT CK_M_WAREHOUSE_WAREHOUSE_ID CHECK (DATALENGTH(I_WAREHOUSE_ID) = 8 AND I_WAREHOUSE_ID LIKE 'WH[0-9][0-9][0-9][0-9][0-9][0-9]'),
    CONSTRAINT CK_M_WAREHOUSE_OPEN_TIME CHECK (I_OPEN_TIME BETWEEN 0 AND 86399),
    CONSTRAINT CK_M_WAREHOUSE_CLOSE_TIME CHECK (I_CLOSE_TIME BETWEEN 0 AND 86399)
);

-- Header
CREATE TABLE T_INVOICE_H (
    I_INVOICE_ID   VARCHAR(20)   COLLATE Latin1_General_100_BIN2 NOT NULL,
    I_INVOICE_AT   BIGINT        NOT NULL,
    I_DUE_DATE     BIGINT        NULL,  -- NULL = payable immediately
    I_CUSTOMER_ID  VARCHAR(20)   COLLATE Latin1_General_100_BIN2 NOT NULL,
    I_STATUS       VARCHAR(20)   COLLATE Latin1_General_100_BIN2 NOT NULL CONSTRAINT DF_T_INVOICE_H_STATUS DEFAULT ('DRAFT'),
    I_CCY          VARCHAR(3)    COLLATE Latin1_General_100_BIN2 NOT NULL,
    I_TOTAL_AMT    DECIMAL(19,4) NOT NULL CONSTRAINT DF_T_INVOICE_H_TOTAL_AMT DEFAULT (0),  -- stored total: sum of lines
    I_CREATED_AT   BIGINT NOT NULL CONSTRAINT DF_T_INVOICE_H_CREATED_AT DEFAULT (DATEDIFF_BIG(SECOND, '19700101', SYSUTCDATETIME())),
    I_UPDATED_AT   BIGINT NOT NULL CONSTRAINT DF_T_INVOICE_H_UPDATED_AT DEFAULT (DATEDIFF_BIG(SECOND, '19700101', SYSUTCDATETIME())),
    CONSTRAINT PK_T_INVOICE_H PRIMARY KEY (I_INVOICE_ID),
    CONSTRAINT FK_T_INVOICE_H_CUSTOMER FOREIGN KEY (I_CUSTOMER_ID)
        REFERENCES M_CUSTOMER (I_CUSTOMER_ID) ON DELETE NO ACTION ON UPDATE NO ACTION,
    CONSTRAINT CK_T_INVOICE_H_INVOICE_ID CHECK (DATALENGTH(I_INVOICE_ID) = 14
        AND I_INVOICE_ID LIKE 'IV[0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9]'),
    CONSTRAINT CK_T_INVOICE_H_DUE_DATE CHECK (I_DUE_DATE % 86400 = 0),
    CONSTRAINT CK_T_INVOICE_H_STATUS CHECK (I_STATUS IN ('DRAFT', 'COMPLETED', 'CANCELLED')),
    CONSTRAINT CK_T_INVOICE_H_CCY CHECK (DATALENGTH(I_CCY) = 3 AND I_CCY LIKE '[A-Z][A-Z][A-Z]'),
    CONSTRAINT CK_T_INVOICE_H_TOTAL_AMT CHECK (I_TOTAL_AMT >= 0)
);
-- FK index; also serves "a customer's invoices by date"
CREATE INDEX IX_T_INVOICE_H_CUSTOMER_ID_INVOICE_AT ON T_INVOICE_H (I_CUSTOMER_ID, I_INVOICE_AT);

-- Detail: composite PK (header ID, I_INTERNAL_NO)
CREATE TABLE T_INVOICE_D (
    I_INVOICE_ID   VARCHAR(20)   COLLATE Latin1_General_100_BIN2 NOT NULL,
    I_INTERNAL_NO  VARCHAR(6)    COLLATE Latin1_General_100_BIN2 NOT NULL,
    I_PRODUCT_ID   VARCHAR(20)   COLLATE Latin1_General_100_BIN2 NOT NULL,
    I_DESC         NVARCHAR(500) NOT NULL,  -- snapshot of product name
    I_QTY          DECIMAL(19,4) NOT NULL,
    I_UNIT_PRICE   DECIMAL(19,4) NOT NULL,  -- snapshot at sale time
    I_TAX_RATE     DECIMAL(9,6)  NOT NULL,  -- snapshot at sale time
    I_LINE_AMT     DECIMAL(19,4) NOT NULL,
    I_CREATED_AT   BIGINT NOT NULL CONSTRAINT DF_T_INVOICE_D_CREATED_AT DEFAULT (DATEDIFF_BIG(SECOND, '19700101', SYSUTCDATETIME())),
    I_UPDATED_AT   BIGINT NOT NULL CONSTRAINT DF_T_INVOICE_D_UPDATED_AT DEFAULT (DATEDIFF_BIG(SECOND, '19700101', SYSUTCDATETIME())),
    CONSTRAINT PK_T_INVOICE_D PRIMARY KEY (I_INVOICE_ID, I_INTERNAL_NO),
    CONSTRAINT FK_T_INVOICE_D_INVOICE_H FOREIGN KEY (I_INVOICE_ID)
        REFERENCES T_INVOICE_H (I_INVOICE_ID) ON DELETE NO ACTION ON UPDATE NO ACTION,
    CONSTRAINT FK_T_INVOICE_D_PRODUCT FOREIGN KEY (I_PRODUCT_ID)
        REFERENCES M_PRODUCT (I_PRODUCT_ID) ON DELETE NO ACTION ON UPDATE NO ACTION,
    CONSTRAINT CK_T_INVOICE_D_INTERNAL_NO CHECK (DATALENGTH(I_INTERNAL_NO) = 6
        AND I_INTERNAL_NO LIKE '[0-9][0-9][0-9][0-9][0-9][0-9]' AND I_INTERNAL_NO <> '000000'),
    CONSTRAINT CK_T_INVOICE_D_QTY CHECK (I_QTY > 0),
    CONSTRAINT CK_T_INVOICE_D_UNIT_PRICE CHECK (I_UNIT_PRICE >= 0),
    CONSTRAINT CK_T_INVOICE_D_TAX_RATE CHECK (I_TAX_RATE BETWEEN 0 AND 1)
);
-- FK to header is covered by the PK's leading column; FK to product needs its own index
CREATE INDEX IX_T_INVOICE_D_PRODUCT_ID ON T_INVOICE_D (I_PRODUCT_ID);

-- Standalone transaction: one record per event, role FKs to the same master
CREATE TABLE T_STOCK_MOVE (
    I_STOCK_MOVE_ID      VARCHAR(20)   COLLATE Latin1_General_100_BIN2 NOT NULL,
    I_MOVE_AT            BIGINT        NOT NULL,
    I_PRODUCT_ID         VARCHAR(20)   COLLATE Latin1_General_100_BIN2 NOT NULL,
    I_FROM_WAREHOUSE_ID  VARCHAR(20)   COLLATE Latin1_General_100_BIN2 NULL,  -- NULL = received from outside
    I_TO_WAREHOUSE_ID    VARCHAR(20)   COLLATE Latin1_General_100_BIN2 NULL,  -- NULL = shipped out
    I_QTY                DECIMAL(19,4) NOT NULL,
    I_CREATED_AT         BIGINT NOT NULL CONSTRAINT DF_T_STOCK_MOVE_CREATED_AT DEFAULT (DATEDIFF_BIG(SECOND, '19700101', SYSUTCDATETIME())),
    I_UPDATED_AT         BIGINT NOT NULL CONSTRAINT DF_T_STOCK_MOVE_UPDATED_AT DEFAULT (DATEDIFF_BIG(SECOND, '19700101', SYSUTCDATETIME())),
    CONSTRAINT PK_T_STOCK_MOVE PRIMARY KEY (I_STOCK_MOVE_ID),
    CONSTRAINT FK_T_STOCK_MOVE_PRODUCT FOREIGN KEY (I_PRODUCT_ID)
        REFERENCES M_PRODUCT (I_PRODUCT_ID) ON DELETE NO ACTION ON UPDATE NO ACTION,
    CONSTRAINT FK_T_STOCK_MOVE_FROM_WAREHOUSE FOREIGN KEY (I_FROM_WAREHOUSE_ID)
        REFERENCES M_WAREHOUSE (I_WAREHOUSE_ID) ON DELETE NO ACTION ON UPDATE NO ACTION,
    CONSTRAINT FK_T_STOCK_MOVE_TO_WAREHOUSE FOREIGN KEY (I_TO_WAREHOUSE_ID)
        REFERENCES M_WAREHOUSE (I_WAREHOUSE_ID) ON DELETE NO ACTION ON UPDATE NO ACTION,
    CONSTRAINT CK_T_STOCK_MOVE_STOCK_MOVE_ID CHECK (DATALENGTH(I_STOCK_MOVE_ID) = 16
        AND I_STOCK_MOVE_ID LIKE 'SM[0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9]'),
    CONSTRAINT CK_T_STOCK_MOVE_QTY CHECK (I_QTY > 0),
    CONSTRAINT CK_T_STOCK_MOVE_WAREHOUSE CHECK (
        (I_FROM_WAREHOUSE_ID IS NOT NULL OR I_TO_WAREHOUSE_ID IS NOT NULL)
        AND (I_FROM_WAREHOUSE_ID IS NULL OR I_TO_WAREHOUSE_ID IS NULL OR I_FROM_WAREHOUSE_ID <> I_TO_WAREHOUSE_ID))
);
CREATE INDEX IX_T_STOCK_MOVE_PRODUCT_ID_MOVE_AT ON T_STOCK_MOVE (I_PRODUCT_ID, I_MOVE_AT);
CREATE INDEX IX_T_STOCK_MOVE_FROM_WAREHOUSE_ID ON T_STOCK_MOVE (I_FROM_WAREHOUSE_ID);
CREATE INDEX IX_T_STOCK_MOVE_TO_WAREHOUSE_ID ON T_STOCK_MOVE (I_TO_WAREHOUSE_ID);
```

## 6. ID generation

### Master ID from a sequence
```sql
INSERT INTO M_CUSTOMER (I_CUSTOMER_ID, I_CUSTOMER_NAME)
VALUES ('CUS' + RIGHT('000000' + CAST(NEXT VALUE FOR SQ_M_CUSTOMER AS VARCHAR(6)), 6), N'บริษัท ตัวอย่าง จำกัด');
```
`MAXVALUE 999999 NO CYCLE` makes the sequence raise an error when it runs out, so the `RIGHT(...)` formatting can never cut digits off. A sequence may leave gaps after a rollback, which is acceptable for master IDs.

### Document ID with a daily counter
Run this inside the caller's transaction, so the counter and the document commit or roll back together. `CREATE PROCEDURE` must be the only statement in its batch, so put `GO` before and after it in scripts.

```sql
CREATE PROCEDURE P_NEXT_DOC_ID
    @prefix   VARCHAR(4),
    @biz_date DATE,          -- business date in the business time zone
    @digits   TINYINT,       -- 4 for IV, 6 for SM
    @doc_id   VARCHAR(20) OUTPUT
AS
BEGIN
    SET NOCOUNT ON;
    SET XACT_ABORT ON;
    IF @@TRANCOUNT = 0
        THROW 50000, 'P_NEXT_DOC_ID must run inside the caller''s transaction.', 1;

    DECLARE @seq_id VARCHAR(20) = @prefix + CONVERT(CHAR(8), @biz_date, 112);
    DECLARE @now    BIGINT      = DATEDIFF_BIG(SECOND, '19700101', SYSUTCDATETIME());
    DECLARE @n      INT;

    -- UPDLOCK + HOLDLOCK also locks the key when the row does not exist yet,
    -- so two sessions cannot both insert the first row for a day.
    UPDATE M_DOC_SEQ WITH (UPDLOCK, HOLDLOCK)
       SET @n = I_LAST_VAL = I_LAST_VAL + 1,
           I_UPDATED_AT = @now
     WHERE I_DOC_SEQ_ID = @seq_id;

    IF @@ROWCOUNT = 0
    BEGIN
        SET @n = 1;
        INSERT INTO M_DOC_SEQ (I_DOC_SEQ_ID, I_LAST_VAL, I_CREATED_AT, I_UPDATED_AT)
        VALUES (@seq_id, @n, @now, @now);
    END;

    IF @n >= POWER(10, @digits)
        THROW 50001, 'Document counter exhausted for this prefix and date.', 1;

    SET @doc_id = @seq_id + RIGHT(REPLICATE('0', @digits) + CAST(@n AS VARCHAR(10)), @digits);
END;
```

### Creating a Header/Detail document
```sql
SET XACT_ABORT ON;
BEGIN TRANSACTION;

DECLARE @invoice_id VARCHAR(20);
DECLARE @now BIGINT = DATEDIFF_BIG(SECOND, '19700101', SYSUTCDATETIME());
DECLARE @biz_date DATE = CAST(SYSDATETIMEOFFSET() AT TIME ZONE 'SE Asia Standard Time' AS DATE);

EXEC P_NEXT_DOC_ID @prefix = 'IV', @biz_date = @biz_date, @digits = 4, @doc_id = @invoice_id OUTPUT;

INSERT INTO T_INVOICE_H (I_INVOICE_ID, I_INVOICE_AT, I_CUSTOMER_ID, I_CCY)
VALUES (@invoice_id, @now, 'CUS000001', 'THB');

-- The application numbers lines 000001, 000002, ... and copies price, tax and name (H3)
INSERT INTO T_INVOICE_D (I_INVOICE_ID, I_INTERNAL_NO, I_PRODUCT_ID, I_DESC, I_QTY, I_UNIT_PRICE, I_TAX_RATE, I_LINE_AMT)
SELECT @invoice_id, '000001', P.I_PRODUCT_ID, P.I_PRODUCT_NAME, 2, P.I_UNIT_PRICE, 0.07, 2 * P.I_UNIT_PRICE
  FROM M_PRODUCT P
 WHERE P.I_PRODUCT_ID = 'PRD000001' AND P.I_DELETED_AT IS NULL;

UPDATE T_INVOICE_H
   SET I_TOTAL_AMT = (SELECT ISNULL(SUM(I_LINE_AMT), 0) FROM T_INVOICE_D WHERE I_INVOICE_ID = @invoice_id),
       I_UPDATED_AT = @now
 WHERE I_INVOICE_ID = @invoice_id;

COMMIT TRANSACTION;
```

### Adding a line to an existing document
```sql
SET XACT_ABORT ON;
BEGIN TRANSACTION;

DECLARE @invoice_id VARCHAR(20) = 'IV202605030001';
DECLARE @locked VARCHAR(20), @next INT;

-- 1. Lock the header first (fixed lock order: header, then details)
SELECT @locked = I_INVOICE_ID
  FROM T_INVOICE_H WITH (UPDLOCK, HOLDLOCK)
 WHERE I_INVOICE_ID = @invoice_id AND I_STATUS = 'DRAFT';
IF @locked IS NULL
    THROW 50002, 'Invoice not found or not in DRAFT status.', 1;

-- 2. Next line number = highest + 1 (zero-padded, so MAX on the string is correct)
SELECT @next = ISNULL(CAST(MAX(I_INTERNAL_NO) AS INT), 0) + 1
  FROM T_INVOICE_D
 WHERE I_INVOICE_ID = @invoice_id;
IF @next > 999999
    THROW 50003, 'Line number limit reached.', 1;

INSERT INTO T_INVOICE_D (I_INVOICE_ID, I_INTERNAL_NO, I_PRODUCT_ID, I_DESC, I_QTY, I_UNIT_PRICE, I_TAX_RATE, I_LINE_AMT)
VALUES (@invoice_id, RIGHT('000000' + CAST(@next AS VARCHAR(6)), 6), 'PRD000002', N'Product B', 1, 150, 0.07, 150);

COMMIT TRANSACTION;
```

## 7. Unique codes on soft-deleted tables
Use a filtered unique index, `CREATE UNIQUE INDEX UQ_… ON … (…) WHERE I_DELETED_AT IS NULL`, as on `M_PRODUCT` above. Every session that writes to the table needs `SET ANSI_NULLS ON` and `QUOTED_IDENTIFIER ON`, which are the defaults.

## 8. Performance notes
- **The PK is clustered by default.** Prefix + date + counter IDs increase in insert order, so new rows go at the end of the index. Random UUID strings would scatter inserts and fragment the index.
- **Indexes are not created for FKs automatically.** Create the `IX_` indexes yourself (K6).
- **Send ID parameters as `VARCHAR`, not `NVARCHAR`.** Many drivers send strings as `NVARCHAR` by default. Comparing an `NVARCHAR` parameter with a `VARCHAR` ID column forces a conversion that can turn an index seek into a full scan. Set the parameter type explicitly:
  - .NET: `SqlDbType.VarChar`
  - JDBC: `sendStringParametersAsUnicode=false` for ID-heavy workloads, after checking that it doesn't affect Unicode text columns
- **Consider `READ_COMMITTED_SNAPSHOT ON`** so readers don't block writers. Writers still lock as shown above.
- **Range queries on `_AT` integers use the index.** Never wrap the column in `DATEADD` or other functions inside a `WHERE`.

## 9. Catalog audit query

This query lists the violations of the rules that can be checked from the system catalog: N1, N2, N4, N5, N6, K1, K2, K3, K5, K6, U1, D1, D2 and D3. Run it in the database to check; it covers the caller's default schema. An empty result means every automated check passed. The remaining rules need a manual review; see `../schema-validation.md`.

```sql
WITH t AS (
    SELECT object_id, CAST(name AS NVARCHAR(128)) COLLATE Latin1_General_100_BIN2 AS tbl
      FROM sys.tables
     WHERE is_ms_shipped = 0 AND schema_id = SCHEMA_ID()
), cols AS (
    SELECT t.object_id, t.tbl, CAST(c.name AS NVARCHAR(128)) COLLATE Latin1_General_100_BIN2 AS col,
           c.column_id, c.max_length, c.is_nullable, c.user_type_id, c.collation_name, CAST(ty.name AS NVARCHAR(128)) COLLATE Latin1_General_100_BIN2 AS typ,
           CASE WHEN EXISTS (SELECT 1 FROM sys.index_columns ic
                               JOIN sys.indexes i ON i.object_id = ic.object_id AND i.index_id = ic.index_id
                              WHERE i.is_primary_key = 1 AND ic.object_id = c.object_id AND ic.column_id = c.column_id)
                  OR EXISTS (SELECT 1 FROM sys.foreign_key_columns fkc
                              WHERE fkc.parent_object_id = c.object_id AND fkc.parent_column_id = c.column_id)
                THEN 1 ELSE 0 END AS is_key
      FROM t
      JOIN sys.columns c ON c.object_id = t.object_id
      JOIN sys.types ty ON ty.user_type_id = c.user_type_id
), obj AS (   -- constraints and indexes with their expected prefix
    SELECT t.tbl, CAST(k.name AS NVARCHAR(128)) COLLATE Latin1_General_100_BIN2 AS name,
           CASE k.type WHEN 'PK' THEN 'PK_' ELSE 'UQ_' END AS pfx, k.type AS kind
      FROM sys.key_constraints k JOIN t ON t.object_id = k.parent_object_id
    UNION ALL
    SELECT t.tbl, CAST(f.name AS NVARCHAR(128)) COLLATE Latin1_General_100_BIN2, 'FK_', 'F'
      FROM sys.foreign_keys f JOIN t ON t.object_id = f.parent_object_id
    UNION ALL
    SELECT t.tbl, CAST(c.name AS NVARCHAR(128)) COLLATE Latin1_General_100_BIN2, 'CK_', 'C'
      FROM sys.check_constraints c JOIN t ON t.object_id = c.parent_object_id
    UNION ALL
    SELECT t.tbl, CAST(d.name AS NVARCHAR(128)) COLLATE Latin1_General_100_BIN2, 'DF_', 'D'
      FROM sys.default_constraints d JOIN t ON t.object_id = d.parent_object_id
    UNION ALL
    SELECT t.tbl, CAST(i.name AS NVARCHAR(128)) COLLATE Latin1_General_100_BIN2,
           CASE WHEN i.is_unique = 1 THEN 'UQ_' ELSE 'IX_' END, 'I'
      FROM sys.indexes i JOIN t ON t.object_id = i.object_id
     WHERE i.index_id > 0 AND i.is_primary_key = 0 AND i.is_unique_constraint = 0
)
SELECT 'N1' AS rule_id, tbl AS object_name, 'name must be uppercase A-Z0-9_, start with a letter, no __ or trailing _' AS problem
  FROM t WHERE tbl LIKE '%[^A-Z0-9_]%' OR tbl NOT LIKE '[A-Z]%' OR tbl LIKE '%[_][_]%' OR tbl LIKE '%[_]'
UNION ALL SELECT 'N2', tbl, 'table must start with M_ or T_' FROM t WHERE LEFT(tbl, 2) NOT IN ('M_', 'T_')
UNION ALL SELECT 'N5', tbl, 'table name longer than 30' FROM t WHERE LEN(tbl) > 30
UNION ALL SELECT 'N1', tbl + '.' + col, 'name must be uppercase A-Z0-9_, start with a letter, no __ or trailing _'
  FROM cols WHERE col LIKE '%[^A-Z0-9_]%' OR col NOT LIKE '[A-Z]%' OR col LIKE '%[_][_]%' OR col LIKE '%[_]'
UNION ALL SELECT 'N4', tbl + '.' + col, 'column must start with I_' FROM cols WHERE LEFT(col, 2) <> 'I_'
UNION ALL SELECT 'N5', tbl + '.' + col, 'column name longer than 30' FROM cols WHERE LEN(col) > 30
UNION ALL SELECT 'K1', tbl + '.' + col, 'key column must be VARCHAR, is ' + typ
  FROM cols WHERE is_key = 1 AND typ <> 'varchar'
UNION ALL SELECT 'K2', tbl + '.' + col, 'key column must use Latin1_General_100_BIN2'
  FROM cols WHERE is_key = 1 AND typ = 'varchar' AND ISNULL(collation_name, '') <> 'Latin1_General_100_BIN2'
UNION ALL SELECT 'K3', CAST(f.name AS NVARCHAR(128)) COLLATE Latin1_General_100_BIN2, 'FK column ' + CAST(pc.name AS NVARCHAR(128)) COLLATE Latin1_General_100_BIN2 + ' differs from ' + CAST(rc.name AS NVARCHAR(128)) COLLATE Latin1_General_100_BIN2 + ' in type, length or collation'
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
  FROM cols WHERE typ IN ('date', 'datetime', 'datetime2', 'smalldatetime', 'datetimeoffset', 'time')
UNION ALL SELECT 'U1', tbl + '.' + col, '_AT/_DATE column must be BIGINT, is ' + typ
  FROM cols WHERE (col LIKE '%[_]AT' OR col LIKE '%[_]DATE') AND typ <> 'bigint'
UNION ALL SELECT 'D1', t.tbl, 'missing ' + v.c
  FROM t CROSS JOIN (VALUES ('I_CREATED_AT'), ('I_UPDATED_AT')) AS v(c)
 WHERE NOT EXISTS (SELECT 1 FROM cols WHERE cols.object_id = t.object_id AND cols.col = v.c)
UNION ALL SELECT 'D1', tbl + '.' + col, 'audit column must be NOT NULL'
  FROM cols WHERE col IN ('I_CREATED_AT', 'I_UPDATED_AT') AND is_nullable = 1
UNION ALL SELECT 'D2', tbl + '.' + col, 'floating-point or money type: ' + typ
  FROM cols WHERE typ IN ('float', 'real', 'money', 'smallmoney')
UNION ALL SELECT 'D3', tbl + '.' + col, 'flag must be CHAR(1), is ' + typ
  FROM cols WHERE (col LIKE 'I[_]IS[_]%' OR col LIKE 'I[_]HAS[_]%') AND NOT (typ = 'char' AND max_length = 1)
UNION ALL SELECT 'N6', name, 'name must start with ' + pfx + tbl + CASE WHEN kind = 'PK' THEN '' ELSE '_' END
  FROM obj
 WHERE (kind = 'PK' AND name <> pfx + tbl)
    OR (kind <> 'PK' AND LEFT(name, LEN(tbl) + 4) <> pfx + tbl + '_')
UNION ALL SELECT 'N6', name, 'object name longer than 63' FROM obj WHERE LEN(name) > 63
ORDER BY 1, 2;
```
