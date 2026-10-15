# PostgreSQL Implementation

Requires PostgreSQL **12 or later**, and the database must be created with `ENCODING 'UTF8'`. This file maps the logical standard to PostgreSQL. It never changes a logical rule.

## 1. Type mapping

| Logical | PostgreSQL | Notes |
|---|---|---|
| `ID(n)` | `VARCHAR(n) COLLATE "C"` | Byte-order comparison and sorting; `LIKE 'X%'` prefix searches can use the index (K2) |
| `LINENO` | `VARCHAR(6) COLLATE "C"` | |
| `CODE(n)` | `VARCHAR(n) COLLATE "C"` | |
| `TEXT(n)` | `VARCHAR(n)` | Counted in characters; UTF8 database; database default collation |
| `FLAG` | `CHAR(1)` + CHECK `IN ('Y','N')` | Not `BOOLEAN`, so that flags are represented the same way on every DBMS |
| `UNIXTIME`, `UNIXDATE` | `BIGINT` | |
| `TIMEOFDAY` | `INTEGER` + CHECK `BETWEEN 0 AND 86399` | |
| `DURATION` | `BIGINT` + CHECK `>= 0` | |
| `AMOUNT`, `QTY` | `NUMERIC(19,4)` | Never `REAL`, `DOUBLE PRECISION` or `MONEY` |
| `RATE` | `NUMERIC(9,6)` | |
| `COUNT` | `INTEGER` | Never a key |

Do not use `SERIAL`, `BIGSERIAL` or `GENERATED … AS IDENTITY` for key columns (K1).

## 2. Names: always quote
PostgreSQL turns unquoted names into lowercase: `CREATE TABLE M_PRODUCT` creates `m_product`. To keep the uppercase names (N1), **quote every identifier in DDL and in every query**: `"M_PRODUCT"."I_PRODUCT_ID"`. Mixing quoted and unquoted forms creates two different objects or fails with "relation does not exist".

- Configure your ORM or query builder to quote identifiers. Most can quote always, or quote when a name is not lowercase.
- Index and constraint names must be unique in the schema. The `IX_<TABLE>_…` patterns ensure this.
- The name limit is 63 bytes, which matches N6.

## 3. Unix time (seconds)

| Task | Expression |
|---|---|
| Now (start of the transaction) | `FLOOR(EXTRACT(EPOCH FROM CURRENT_TIMESTAMP))::BIGINT` |
| Now (actual clock time) | `FLOOR(EXTRACT(EPOCH FROM clock_timestamp()))::BIGINT` |
| Unix → `timestamptz` | `to_timestamp(u)` |
| `timestamptz` → Unix | `FLOOR(EXTRACT(EPOCH FROM ts))::BIGINT` |
| `DATE` → `_DATE` value | `EXTRACT(EPOCH FROM d::timestamp)::BIGINT` (a `timestamp` without zone is read as UTC) |
| `_DATE` value → `DATE` | `(to_timestamp(u) AT TIME ZONE 'UTC')::date` |
| Start of a local business day | `EXTRACT(EPOCH FROM (d::timestamp AT TIME ZONE 'Asia/Bangkok'))::BIGINT` |

PostgreSQL uses IANA time-zone names such as `Asia/Bangkok`. For a local day, query with `"I_X_AT" >= start AND "I_X_AT" < start + 86400`.

## 4. CHECK patterns
Use regular expressions with `^…$` anchors, which reject spaces and lowercase letters:

```sql
CHECK ("I_CUSTOMER_ID" ~ '^CUS[0-9]{6}$')      -- fixed-format ID
CHECK ("I_DOC_SEQ_ID"  ~ '^[A-Z0-9]+$')        -- any ID
CHECK ("I_SKU"         ~ '^[A-Z0-9_-]+$')      -- code
CHECK ("I_DUE_DATE" % 86400 = 0)               -- calendar date
```

## 5. Example schema (compliant)

This is the same design as the SQL Server example. This example system has no user accounts, so the `_BY` audit columns are omitted (D1).

```sql
CREATE SEQUENCE "SQ_M_CUSTOMER"  AS INTEGER MINVALUE 1 MAXVALUE 999999 NO CYCLE;
CREATE SEQUENCE "SQ_M_PRODUCT"   AS INTEGER MINVALUE 1 MAXVALUE 999999 NO CYCLE;
CREATE SEQUENCE "SQ_M_WAREHOUSE" AS INTEGER MINVALUE 1 MAXVALUE 999999 NO CYCLE;

-- Counter for document IDs (configuration -> master)
CREATE TABLE "M_DOC_SEQ" (
    "I_DOC_SEQ_ID"  VARCHAR(20) COLLATE "C" NOT NULL,
    "I_LAST_VAL"    INTEGER NOT NULL,
    "I_CREATED_AT"  BIGINT  NOT NULL DEFAULT FLOOR(EXTRACT(EPOCH FROM CURRENT_TIMESTAMP))::BIGINT,
    "I_UPDATED_AT"  BIGINT  NOT NULL DEFAULT FLOOR(EXTRACT(EPOCH FROM CURRENT_TIMESTAMP))::BIGINT,
    CONSTRAINT "PK_M_DOC_SEQ" PRIMARY KEY ("I_DOC_SEQ_ID"),
    CONSTRAINT "CK_M_DOC_SEQ_DOC_SEQ_ID" CHECK ("I_DOC_SEQ_ID" ~ '^[A-Z0-9]+$'),
    CONSTRAINT "CK_M_DOC_SEQ_LAST_VAL" CHECK ("I_LAST_VAL" >= 0)
);

-- Master
CREATE TABLE "M_CUSTOMER" (
    "I_CUSTOMER_ID"    VARCHAR(20) COLLATE "C" NOT NULL,
    "I_CUSTOMER_NAME"  VARCHAR(200) NOT NULL,
    "I_TAX_NO"         VARCHAR(30) COLLATE "C" NULL,  -- NULL = not registered for tax
    "I_IS_ACTIVE"      CHAR(1) NOT NULL DEFAULT 'Y',
    "I_CREATED_AT"     BIGINT NOT NULL DEFAULT FLOOR(EXTRACT(EPOCH FROM CURRENT_TIMESTAMP))::BIGINT,
    "I_UPDATED_AT"     BIGINT NOT NULL DEFAULT FLOOR(EXTRACT(EPOCH FROM CURRENT_TIMESTAMP))::BIGINT,
    CONSTRAINT "PK_M_CUSTOMER" PRIMARY KEY ("I_CUSTOMER_ID"),
    CONSTRAINT "CK_M_CUSTOMER_CUSTOMER_ID" CHECK ("I_CUSTOMER_ID" ~ '^CUS[0-9]{6}$'),
    CONSTRAINT "CK_M_CUSTOMER_TAX_NO" CHECK ("I_TAX_NO" ~ '^[A-Z0-9_-]+$'),
    CONSTRAINT "CK_M_CUSTOMER_IS_ACTIVE" CHECK ("I_IS_ACTIVE" IN ('Y', 'N'))
);

-- Master with soft delete and a changeable business code
CREATE TABLE "M_PRODUCT" (
    "I_PRODUCT_ID"    VARCHAR(20) COLLATE "C" NOT NULL,
    "I_SKU"           VARCHAR(30) COLLATE "C" NOT NULL,
    "I_PRODUCT_NAME"  VARCHAR(200) NOT NULL,
    "I_UNIT_PRICE"    NUMERIC(19,4) NOT NULL,
    "I_CREATED_AT"    BIGINT NOT NULL DEFAULT FLOOR(EXTRACT(EPOCH FROM CURRENT_TIMESTAMP))::BIGINT,
    "I_UPDATED_AT"    BIGINT NOT NULL DEFAULT FLOOR(EXTRACT(EPOCH FROM CURRENT_TIMESTAMP))::BIGINT,
    "I_DELETED_AT"    BIGINT NULL,  -- NULL = active
    CONSTRAINT "PK_M_PRODUCT" PRIMARY KEY ("I_PRODUCT_ID"),
    CONSTRAINT "CK_M_PRODUCT_PRODUCT_ID" CHECK ("I_PRODUCT_ID" ~ '^PRD[0-9]{6}$'),
    CONSTRAINT "CK_M_PRODUCT_SKU" CHECK ("I_SKU" ~ '^[A-Z0-9_-]+$'),
    CONSTRAINT "CK_M_PRODUCT_UNIT_PRICE" CHECK ("I_UNIT_PRICE" >= 0)
);
-- SKU unique among active products only
CREATE UNIQUE INDEX "UQ_M_PRODUCT_SKU" ON "M_PRODUCT" ("I_SKU") WHERE "I_DELETED_AT" IS NULL;

CREATE TABLE "M_WAREHOUSE" (
    "I_WAREHOUSE_ID"    VARCHAR(20) COLLATE "C" NOT NULL,
    "I_WAREHOUSE_NAME"  VARCHAR(200) NOT NULL,
    "I_OPEN_TIME"       INTEGER NOT NULL,  -- seconds after local midnight
    "I_CLOSE_TIME"      INTEGER NOT NULL,
    "I_CREATED_AT"      BIGINT NOT NULL DEFAULT FLOOR(EXTRACT(EPOCH FROM CURRENT_TIMESTAMP))::BIGINT,
    "I_UPDATED_AT"      BIGINT NOT NULL DEFAULT FLOOR(EXTRACT(EPOCH FROM CURRENT_TIMESTAMP))::BIGINT,
    CONSTRAINT "PK_M_WAREHOUSE" PRIMARY KEY ("I_WAREHOUSE_ID"),
    CONSTRAINT "CK_M_WAREHOUSE_WAREHOUSE_ID" CHECK ("I_WAREHOUSE_ID" ~ '^WH[0-9]{6}$'),
    CONSTRAINT "CK_M_WAREHOUSE_OPEN_TIME" CHECK ("I_OPEN_TIME" BETWEEN 0 AND 86399),
    CONSTRAINT "CK_M_WAREHOUSE_CLOSE_TIME" CHECK ("I_CLOSE_TIME" BETWEEN 0 AND 86399)
);

-- Header
CREATE TABLE "T_INVOICE_H" (
    "I_INVOICE_ID"   VARCHAR(20) COLLATE "C" NOT NULL,
    "I_INVOICE_AT"   BIGINT NOT NULL,
    "I_DUE_DATE"     BIGINT NULL,  -- NULL = payable immediately
    "I_CUSTOMER_ID"  VARCHAR(20) COLLATE "C" NOT NULL,
    "I_STATUS"       VARCHAR(20) COLLATE "C" NOT NULL DEFAULT 'DRAFT',
    "I_CCY"          VARCHAR(3)  COLLATE "C" NOT NULL,
    "I_TOTAL_AMT"    NUMERIC(19,4) NOT NULL DEFAULT 0,  -- stored total: sum of lines
    "I_CREATED_AT"   BIGINT NOT NULL DEFAULT FLOOR(EXTRACT(EPOCH FROM CURRENT_TIMESTAMP))::BIGINT,
    "I_UPDATED_AT"   BIGINT NOT NULL DEFAULT FLOOR(EXTRACT(EPOCH FROM CURRENT_TIMESTAMP))::BIGINT,
    CONSTRAINT "PK_T_INVOICE_H" PRIMARY KEY ("I_INVOICE_ID"),
    CONSTRAINT "FK_T_INVOICE_H_CUSTOMER" FOREIGN KEY ("I_CUSTOMER_ID")
        REFERENCES "M_CUSTOMER" ("I_CUSTOMER_ID") ON DELETE NO ACTION ON UPDATE NO ACTION,
    CONSTRAINT "CK_T_INVOICE_H_INVOICE_ID" CHECK ("I_INVOICE_ID" ~ '^IV[0-9]{12}$'),
    CONSTRAINT "CK_T_INVOICE_H_DUE_DATE" CHECK ("I_DUE_DATE" % 86400 = 0),
    CONSTRAINT "CK_T_INVOICE_H_STATUS" CHECK ("I_STATUS" IN ('DRAFT', 'COMPLETED', 'CANCELLED')),
    CONSTRAINT "CK_T_INVOICE_H_CCY" CHECK ("I_CCY" ~ '^[A-Z]{3}$'),
    CONSTRAINT "CK_T_INVOICE_H_TOTAL_AMT" CHECK ("I_TOTAL_AMT" >= 0)
);
-- FK index; also serves "a customer's invoices by date"
CREATE INDEX "IX_T_INVOICE_H_CUSTOMER_ID_INVOICE_AT" ON "T_INVOICE_H" ("I_CUSTOMER_ID", "I_INVOICE_AT");

-- Detail: composite PK (header ID, I_INTERNAL_NO)
CREATE TABLE "T_INVOICE_D" (
    "I_INVOICE_ID"   VARCHAR(20) COLLATE "C" NOT NULL,
    "I_INTERNAL_NO"  VARCHAR(6)  COLLATE "C" NOT NULL,
    "I_PRODUCT_ID"   VARCHAR(20) COLLATE "C" NOT NULL,
    "I_DESC"         VARCHAR(500) NOT NULL,  -- snapshot of product name
    "I_QTY"          NUMERIC(19,4) NOT NULL,
    "I_UNIT_PRICE"   NUMERIC(19,4) NOT NULL, -- snapshot at sale time
    "I_TAX_RATE"     NUMERIC(9,6)  NOT NULL, -- snapshot at sale time
    "I_LINE_AMT"     NUMERIC(19,4) NOT NULL,
    "I_CREATED_AT"   BIGINT NOT NULL DEFAULT FLOOR(EXTRACT(EPOCH FROM CURRENT_TIMESTAMP))::BIGINT,
    "I_UPDATED_AT"   BIGINT NOT NULL DEFAULT FLOOR(EXTRACT(EPOCH FROM CURRENT_TIMESTAMP))::BIGINT,
    CONSTRAINT "PK_T_INVOICE_D" PRIMARY KEY ("I_INVOICE_ID", "I_INTERNAL_NO"),
    CONSTRAINT "FK_T_INVOICE_D_INVOICE_H" FOREIGN KEY ("I_INVOICE_ID")
        REFERENCES "T_INVOICE_H" ("I_INVOICE_ID") ON DELETE NO ACTION ON UPDATE NO ACTION,
    CONSTRAINT "FK_T_INVOICE_D_PRODUCT" FOREIGN KEY ("I_PRODUCT_ID")
        REFERENCES "M_PRODUCT" ("I_PRODUCT_ID") ON DELETE NO ACTION ON UPDATE NO ACTION,
    CONSTRAINT "CK_T_INVOICE_D_INTERNAL_NO" CHECK ("I_INTERNAL_NO" ~ '^[0-9]{6}$' AND "I_INTERNAL_NO" <> '000000'),
    CONSTRAINT "CK_T_INVOICE_D_QTY" CHECK ("I_QTY" > 0),
    CONSTRAINT "CK_T_INVOICE_D_UNIT_PRICE" CHECK ("I_UNIT_PRICE" >= 0),
    CONSTRAINT "CK_T_INVOICE_D_TAX_RATE" CHECK ("I_TAX_RATE" BETWEEN 0 AND 1)
);
-- FK to header is covered by the PK's leading column; FK to product needs its own index
CREATE INDEX "IX_T_INVOICE_D_PRODUCT_ID" ON "T_INVOICE_D" ("I_PRODUCT_ID");

-- Standalone transaction: one record per event, role FKs to the same master
CREATE TABLE "T_STOCK_MOVE" (
    "I_STOCK_MOVE_ID"      VARCHAR(20) COLLATE "C" NOT NULL,
    "I_MOVE_AT"            BIGINT NOT NULL,
    "I_PRODUCT_ID"         VARCHAR(20) COLLATE "C" NOT NULL,
    "I_FROM_WAREHOUSE_ID"  VARCHAR(20) COLLATE "C" NULL,  -- NULL = received from outside
    "I_TO_WAREHOUSE_ID"    VARCHAR(20) COLLATE "C" NULL,  -- NULL = shipped out
    "I_QTY"                NUMERIC(19,4) NOT NULL,
    "I_CREATED_AT"         BIGINT NOT NULL DEFAULT FLOOR(EXTRACT(EPOCH FROM CURRENT_TIMESTAMP))::BIGINT,
    "I_UPDATED_AT"         BIGINT NOT NULL DEFAULT FLOOR(EXTRACT(EPOCH FROM CURRENT_TIMESTAMP))::BIGINT,
    CONSTRAINT "PK_T_STOCK_MOVE" PRIMARY KEY ("I_STOCK_MOVE_ID"),
    CONSTRAINT "FK_T_STOCK_MOVE_PRODUCT" FOREIGN KEY ("I_PRODUCT_ID")
        REFERENCES "M_PRODUCT" ("I_PRODUCT_ID") ON DELETE NO ACTION ON UPDATE NO ACTION,
    CONSTRAINT "FK_T_STOCK_MOVE_FROM_WAREHOUSE" FOREIGN KEY ("I_FROM_WAREHOUSE_ID")
        REFERENCES "M_WAREHOUSE" ("I_WAREHOUSE_ID") ON DELETE NO ACTION ON UPDATE NO ACTION,
    CONSTRAINT "FK_T_STOCK_MOVE_TO_WAREHOUSE" FOREIGN KEY ("I_TO_WAREHOUSE_ID")
        REFERENCES "M_WAREHOUSE" ("I_WAREHOUSE_ID") ON DELETE NO ACTION ON UPDATE NO ACTION,
    CONSTRAINT "CK_T_STOCK_MOVE_STOCK_MOVE_ID" CHECK ("I_STOCK_MOVE_ID" ~ '^SM[0-9]{14}$'),
    CONSTRAINT "CK_T_STOCK_MOVE_QTY" CHECK ("I_QTY" > 0),
    CONSTRAINT "CK_T_STOCK_MOVE_WAREHOUSE" CHECK (
        ("I_FROM_WAREHOUSE_ID" IS NOT NULL OR "I_TO_WAREHOUSE_ID" IS NOT NULL)
        AND ("I_FROM_WAREHOUSE_ID" IS NULL OR "I_TO_WAREHOUSE_ID" IS NULL OR "I_FROM_WAREHOUSE_ID" <> "I_TO_WAREHOUSE_ID"))
);
CREATE INDEX "IX_T_STOCK_MOVE_PRODUCT_ID_MOVE_AT" ON "T_STOCK_MOVE" ("I_PRODUCT_ID", "I_MOVE_AT");
CREATE INDEX "IX_T_STOCK_MOVE_FROM_WAREHOUSE_ID" ON "T_STOCK_MOVE" ("I_FROM_WAREHOUSE_ID");
CREATE INDEX "IX_T_STOCK_MOVE_TO_WAREHOUSE_ID" ON "T_STOCK_MOVE" ("I_TO_WAREHOUSE_ID");
```

## 6. ID generation

### Master ID from a sequence
```sql
INSERT INTO "M_CUSTOMER" ("I_CUSTOMER_ID", "I_CUSTOMER_NAME")
VALUES ('CUS' || LPAD(nextval('"SQ_M_CUSTOMER"')::TEXT, 6, '0'), 'บริษัท ตัวอย่าง จำกัด');
```
`LPAD` cuts off digits if a value is too long. `MAXVALUE 999999 NO CYCLE` makes the sequence raise an error first, so that never happens. Note the double quotes inside the string: `nextval('"SQ_M_CUSTOMER"')`.

### Document ID with a daily counter
A function always runs inside the caller's transaction, so the counter rolls back together with the document:

```sql
CREATE FUNCTION "FN_NEXT_DOC_ID"(p_prefix VARCHAR, p_biz_date DATE, p_digits INTEGER)
RETURNS VARCHAR
LANGUAGE plpgsql
AS $$
DECLARE
    v_seq_id VARCHAR(20) := p_prefix || to_char(p_biz_date, 'YYYYMMDD');
    v_now    BIGINT      := FLOOR(EXTRACT(EPOCH FROM clock_timestamp()))::BIGINT;
    v_n      INTEGER;
BEGIN
    -- Atomic insert-or-increment; the row lock is held until the caller commits
    INSERT INTO "M_DOC_SEQ" ("I_DOC_SEQ_ID", "I_LAST_VAL", "I_CREATED_AT", "I_UPDATED_AT")
    VALUES (v_seq_id, 1, v_now, v_now)
    ON CONFLICT ("I_DOC_SEQ_ID") DO UPDATE
        SET "I_LAST_VAL"   = "M_DOC_SEQ"."I_LAST_VAL" + 1,
            "I_UPDATED_AT" = EXCLUDED."I_UPDATED_AT"
    RETURNING "I_LAST_VAL" INTO v_n;

    IF v_n >= power(10, p_digits) THEN
        RAISE EXCEPTION 'Document counter exhausted for %', v_seq_id;
    END IF;

    RETURN v_seq_id || LPAD(v_n::TEXT, p_digits, '0');
END;
$$;
```

### Creating a Header/Detail document
The application normally runs these statements in one transaction and keeps the returned ID in a variable. The `DO` block shows the same steps in plain SQL:

```sql
DO $$
DECLARE
    v_id  VARCHAR(20) := "FN_NEXT_DOC_ID"('IV', (CURRENT_TIMESTAMP AT TIME ZONE 'Asia/Bangkok')::date, 4);
    v_now BIGINT      := FLOOR(EXTRACT(EPOCH FROM CURRENT_TIMESTAMP))::BIGINT;
BEGIN
    INSERT INTO "T_INVOICE_H" ("I_INVOICE_ID", "I_INVOICE_AT", "I_CUSTOMER_ID", "I_CCY")
    VALUES (v_id, v_now, 'CUS000001', 'THB');

    -- The application numbers lines 000001, 000002, ... and copies price, tax and name (H3)
    INSERT INTO "T_INVOICE_D" ("I_INVOICE_ID", "I_INTERNAL_NO", "I_PRODUCT_ID", "I_DESC",
                               "I_QTY", "I_UNIT_PRICE", "I_TAX_RATE", "I_LINE_AMT")
    SELECT v_id, '000001', p."I_PRODUCT_ID", p."I_PRODUCT_NAME",
           2, p."I_UNIT_PRICE", 0.07, 2 * p."I_UNIT_PRICE"
      FROM "M_PRODUCT" p
     WHERE p."I_PRODUCT_ID" = 'PRD000001' AND p."I_DELETED_AT" IS NULL;

    -- Recalculate the stored total in the same transaction
    UPDATE "T_INVOICE_H"
       SET "I_TOTAL_AMT"  = COALESCE((SELECT SUM("I_LINE_AMT") FROM "T_INVOICE_D" WHERE "I_INVOICE_ID" = v_id), 0),
           "I_UPDATED_AT" = v_now
     WHERE "I_INVOICE_ID" = v_id;
END;
$$;
```

### Adding a line to an existing document
```sql
BEGIN;

-- 1. Lock the header first (fixed lock order: header, then details)
SELECT "I_INVOICE_ID"
  FROM "T_INVOICE_H"
 WHERE "I_INVOICE_ID" = 'IV202605030001' AND "I_STATUS" = 'DRAFT'
   FOR UPDATE;
-- The application stops if no row is returned.

-- 2. Next line number = highest + 1 (zero-padded, so MAX on the string is correct)
INSERT INTO "T_INVOICE_D" ("I_INVOICE_ID", "I_INTERNAL_NO", "I_PRODUCT_ID", "I_DESC",
                           "I_QTY", "I_UNIT_PRICE", "I_TAX_RATE", "I_LINE_AMT")
SELECT 'IV202605030001',
       LPAD((COALESCE(MAX("I_INTERNAL_NO")::INTEGER, 0) + 1)::TEXT, 6, '0'),
       'PRD000002', 'Product B', 1, 150, 0.07, 150
  FROM "T_INVOICE_D"
 WHERE "I_INVOICE_ID" = 'IV202605030001';
-- Above 999999 the result has 7 digits, so VARCHAR(6) rejects it instead of cutting it off.

COMMIT;
```

## 7. Unique codes on soft-deleted tables
Use a partial unique index: `CREATE UNIQUE INDEX "UQ_…" ON … (…) WHERE "I_DELETED_AT" IS NULL`.

## 8. Performance notes
- **Indexes are not created for FKs automatically.** Create the `IX_` indexes yourself (K6).
- **`COLLATE "C"` on IDs and codes** makes comparison and sorting byte-based and fast. IDs only contain ASCII, so the result is the same as any other collation, and prefix `LIKE` searches can use the index.
- **Write `BETWEEN`-style range queries on `_AT` as integer comparisons.** Never wrap the column in `to_timestamp()` inside a `WHERE`.
- **Table rows are not clustered** in PostgreSQL. Sequential IDs still keep the B-tree index compact.

## 9. Catalog audit query

This query lists the violations of the rules that can be checked from the system catalog: N1, N2, N4, N5, N6, K1, K2, K3, K5, K6, U1, D1, D2 and D3. Run it in the schema to check (`SET search_path = <schema>` first). An empty result means every automated check passed. The remaining rules need a manual review; see `../schema-validation.md`.

```sql
WITH t AS (
    SELECT c.oid, c.relname::TEXT AS tbl
      FROM pg_class c JOIN pg_namespace n ON n.oid = c.relnamespace
     WHERE c.relkind IN ('r', 'p') AND n.nspname = current_schema()
), col AS (
    SELECT t.oid, t.tbl, a.attname::TEXT AS col, a.attnum, a.atttypid, a.attcollation, a.attnotnull,
           format_type(a.atttypid, a.atttypmod) AS typ,
           EXISTS (SELECT 1 FROM pg_constraint k
                    WHERE k.conrelid = t.oid AND k.contype IN ('p', 'f') AND a.attnum = ANY (k.conkey)) AS is_key
      FROM t JOIN pg_attribute a ON a.attrelid = t.oid AND a.attnum > 0 AND NOT a.attisdropped
), fk AS (
    SELECT k.*, t.tbl FROM pg_constraint k JOIN t ON t.oid = k.conrelid WHERE k.contype = 'f'
)
SELECT 'N1' AS rule_id, tbl AS object_name, 'name must be uppercase A-Z0-9_, start with a letter, no __ or trailing _' AS problem
  FROM t WHERE tbl !~ '^[A-Z][A-Z0-9]*(_[A-Z0-9]+)*$'
UNION ALL SELECT 'N2', tbl, 'table must start with M_ or T_' FROM t WHERE left(tbl, 2) NOT IN ('M_', 'T_')
UNION ALL SELECT 'N5', tbl, 'table name longer than 30' FROM t WHERE length(tbl) > 30
UNION ALL SELECT 'N1', tbl || '.' || col, 'name must be uppercase A-Z0-9_, start with a letter, no __ or trailing _'
  FROM col WHERE col !~ '^[A-Z][A-Z0-9]*(_[A-Z0-9]+)*$'
UNION ALL SELECT 'N4', tbl || '.' || col, 'column must start with I_' FROM col WHERE left(col, 2) <> 'I_'
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
UNION ALL SELECT 'U1', tbl || '.' || col, '_AT/_DATE column must be BIGINT, is ' || typ
  FROM col WHERE (col LIKE '%\_AT' OR col LIKE '%\_DATE') AND atttypid <> 'int8'::regtype
UNION ALL SELECT 'D1', t.tbl, 'missing ' || v.c
  FROM t CROSS JOIN (VALUES ('I_CREATED_AT'), ('I_UPDATED_AT')) AS v(c)
 WHERE NOT EXISTS (SELECT 1 FROM col WHERE col.oid = t.oid AND col.col = v.c)
UNION ALL SELECT 'D1', tbl || '.' || col, 'audit column must be NOT NULL'
  FROM col WHERE col IN ('I_CREATED_AT', 'I_UPDATED_AT') AND NOT attnotnull
UNION ALL SELECT 'D2', tbl || '.' || col, 'floating-point or money type: ' || typ
  FROM col WHERE atttypid IN ('real'::regtype, 'double precision'::regtype, 'money'::regtype)
UNION ALL SELECT 'D3', tbl || '.' || col, 'flag must be CHAR(1), is ' || typ
  FROM col WHERE (col LIKE 'I\_IS\_%' OR col LIKE 'I\_HAS\_%') AND typ <> 'character(1)'
UNION ALL SELECT 'N6', k.conname, 'constraint name must start with '
       || CASE k.contype WHEN 'p' THEN 'PK_' WHEN 'f' THEN 'FK_' WHEN 'u' THEN 'UQ_' ELSE 'CK_' END || t.tbl
  FROM pg_constraint k JOIN t ON t.oid = k.conrelid
 WHERE k.contype IN ('p', 'f', 'u', 'c')
   AND NOT (CASE k.contype WHEN 'p' THEN k.conname = 'PK_' || t.tbl
                           ELSE left(k.conname, length(t.tbl) + 4)
                                = CASE k.contype WHEN 'f' THEN 'FK_' WHEN 'u' THEN 'UQ_' ELSE 'CK_' END || t.tbl || '_' END)
UNION ALL SELECT 'N6', ic.relname, 'index name must start with IX_' || t.tbl || '_ (or UQ_ when unique)'
  FROM pg_index i JOIN pg_class ic ON ic.oid = i.indexrelid JOIN t ON t.oid = i.indrelid
 WHERE NOT EXISTS (SELECT 1 FROM pg_constraint k WHERE k.conindid = i.indexrelid AND k.contype IN ('p', 'u'))
   AND left(ic.relname, length(t.tbl) + 4) <> (CASE WHEN i.indisunique THEN 'UQ_' ELSE 'IX_' END) || t.tbl || '_'
UNION ALL SELECT 'N6', conname, 'object name longer than 63' FROM pg_constraint k JOIN t ON t.oid = k.conrelid WHERE length(conname) > 63
ORDER BY 1, 2;
```
