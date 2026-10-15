# Examples

The complete, tested DDL for each DBMS is in `dbms/<dbms>.md`, section 5. This file shows the logical design, the patterns that go with it, and common mistakes.

## 1. Compliant logical design

```
M_CUSTOMER  1 ──< T_INVOICE_H 1 ──< T_INVOICE_D >── 1 M_PRODUCT
                                                        1
M_WAREHOUSE 1 ──< T_STOCK_MOVE (FROM / TO) >───────────┘
M_DOC_SEQ   (counter for document IDs)
```

| Table | Class | Structure | PK |
|---|---|---|---|
| `M_CUSTOMER` | Master | single | `I_CUSTOMER_ID` `CUS000001` |
| `M_PRODUCT` | Master, soft delete | single | `I_PRODUCT_ID` `PRD000001` |
| `M_WAREHOUSE` | Master | single | `I_WAREHOUSE_ID` `WH000001` |
| `M_DOC_SEQ` | Master (configuration) | single | `I_DOC_SEQ_ID` `IV20260503` |
| `T_INVOICE_H` | Transaction | header | `I_INVOICE_ID` `IV202605030001` |
| `T_INVOICE_D` | Transaction | detail | `(I_INVOICE_ID, I_INTERNAL_NO)` |
| `T_STOCK_MOVE` | Transaction | standalone | `I_STOCK_MOVE_ID` `SM20260503000001` |

Why each structure:

- **`T_INVOICE_H/D`:** an invoice has shared data (customer, time, status, total) plus many lines, so H2 applies.
- **`T_STOCK_MOVE`:** each movement is a complete record on its own, so it is a single table. It references `M_WAREHOUSE` twice, so its FK columns carry the role words `FROM` and `TO`.

### `T_INVOICE_D` sample rows

| I_INVOICE_ID | I_INTERNAL_NO | I_PRODUCT_ID | I_DESC | I_QTY | I_UNIT_PRICE |
|---|---|---|---|---|---|
| IV202605030001 | 000001 | PRD000001 | Product A | 2.0000 | 100.0000 |
| IV202605030001 | 000002 | PRD000002 | Product B | 1.0000 | 150.0000 |
| IV202605030002 | 000001 | PRD000001 | Product A | 5.0000 | 95.0000 |

`000001` appears under both invoices. `I_INTERNAL_NO` is unique only within its invoice (H1). The second invoice has a different price for the same product because prices are copied at the time of the transaction (H3).

## 2. Other compliant patterns (logical)

**Junction (many-to-many between masters)**
```
M_PRODUCT_CATEGORY
  I_PRODUCT_ID    ID(20)    PK, FK -> M_PRODUCT      (FK_M_PRODUCT_CATEGORY_PRODUCT)
  I_CATEGORY_ID   ID(20)    PK, FK -> M_CATEGORY     (FK_M_PRODUCT_CATEGORY_CATEGORY)
  I_CREATED_AT    UNIXTIME
  I_UPDATED_AT    UNIXTIME
  IX_M_PRODUCT_CATEGORY_CATEGORY_ID on (I_CATEGORY_ID)   -- FK not covered by the PK's leading column
```

**Second child collection and sub-detail**
```
T_INVOICE_PAY_D   PK (I_INVOICE_ID, I_INTERNAL_NO)            FK I_INVOICE_ID -> T_INVOICE_H
T_INVOICE_LOT_D   PK (I_INVOICE_ID, I_INTERNAL_NO, I_SUB_NO)  FK (I_INVOICE_ID, I_INTERNAL_NO) -> T_INVOICE_D
                  name: FK_T_INVOICE_LOT_D_INVOICE_D
```

**One row per customer per day** (a time value is never part of a key)
```
T_DAILY_LIMIT
  I_DAILY_LIMIT_ID  ID(20)    PK
  I_CUSTOMER_ID     ID(20)    FK -> M_CUSTOMER
  I_LIMIT_DATE      UNIXDATE  CHECK % 86400 = 0
  UQ_T_DAILY_LIMIT_CUSTOMER_ID_LIMIT_DATE on (I_CUSTOMER_ID, I_LIMIT_DATE)
```

## 3. Non-compliant examples

Each example below breaks the rules listed after it. These are the mistakes to catch in a review.

### 3.1 Integer keys, a date type, and names in the wrong case
```sql
CREATE TABLE invoice_items (
    id          INT IDENTITY PRIMARY KEY,
    invoice_id  INT NOT NULL REFERENCES invoices(id) ON DELETE CASCADE,
    line_no     INT NOT NULL,
    created     DATETIME NOT NULL
);
```

| Problem | Rule | Fix |
|---|---|---|
| Lowercase, plural, no `T_` prefix or `_D` suffix | N1, N2, N3 | `T_INVOICE_D` |
| Columns without `I_` | N4 | `I_INVOICE_ID`, `I_INTERNAL_NO`, `I_CREATED_AT` |
| Integer surrogate PK `id` | K1, H1 | Composite PK `(I_INVOICE_ID, I_INTERNAL_NO)`; drop `id` |
| Integer FK `invoice_id` | K1, K3 | `VARCHAR(20)` with binary collation, identical to the header PK |
| Integer `line_no` | K1, H1 | `I_INTERNAL_NO VARCHAR(6)`, `000001` format |
| `ON DELETE CASCADE` | K5 | `NO ACTION`; cancel through the header's status |
| `DATETIME` column | U1 | `I_CREATED_AT BIGINT` in Unix seconds |
| No `I_UPDATED_AT` | D1 | Add it |
| No snapshot columns | H3 | Add `I_DESC`, `I_UNIT_PRICE`, `I_TAX_RATE` |

### 3.2 Header/Detail keyed by a single detail ID
```sql
CREATE TABLE T_ORDER_D (
    I_ORDER_D_ID  VARCHAR(20) PRIMARY KEY,
    I_ORDER_ID    VARCHAR(20) NOT NULL REFERENCES T_ORDER_H (I_ORDER_ID),
    ...
);
```
- **H1:** the detail PK must be `(I_ORDER_ID, I_INTERNAL_NO)`, not its own single ID.
- **Fix:** replace `I_ORDER_D_ID` with `I_INTERNAL_NO VARCHAR(6)`, then make the PK composite. The FK index is then the PK's leading column.

### 3.3 Mismatched FK definition and collation
```sql
-- M_CUSTOMER.I_CUSTOMER_ID is VARCHAR(20) with a binary collation
I_CUSTOMER_ID NVARCHAR(30) NOT NULL REFERENCES M_CUSTOMER (I_CUSTOMER_ID)
```
- **K3:** the type, length and collation all differ from the PK.
  - SQL Server rejects the FK.
  - Where the DBMS accepts it, joins compare different types and stop using the index.
- **Fix:** copy the PK definition exactly: `VARCHAR(20) COLLATE <binary>`.

### 3.4 Unix time misuse
```sql
I_ORDER_DATE  VARCHAR(10)  -- '2026-05-03'
I_PAID_AT     INT          -- 32-bit
I_SHIP_AT     BIGINT       -- milliseconds: 1777793415000
I_BIRTH_DATE  BIGINT       -- 1777741200 (Bangkok midnight)
```

| Column | Rule | Problem and fix |
|---|---|---|
| `I_ORDER_DATE` | U1, U2 | A string date. Use `BIGINT` holding UTC midnight: `1777766400` |
| `I_PAID_AT` | U1 | 32-bit, so it fails in 2038. Use `BIGINT` |
| `I_SHIP_AT` | U1 | Milliseconds. Store seconds: `1777793415` |
| `I_BIRTH_DATE` | U2 | Local midnight, which is not divisible by 86400 and fails the CHECK. Use UTC midnight `1777766400` and add `CK_…_BIRTH_DATE` |

### 3.5 A timestamp in the key
```sql
CONSTRAINT PK_T_DAILY_LIMIT PRIMARY KEY (I_CUSTOMER_ID, I_LIMIT_DATE)
```
- **K1:** `I_LIMIT_DATE` is numeric.
- **Fix:** see "One row per customer per day" in section 2: use a string PK plus a `UQ_` on `(I_CUSTOMER_ID, I_LIMIT_DATE)`.

### 3.6 Names that are too long or unclear
```sql
CREATE TABLE T_CUSTOMER_PURCHASE_ORDER_HEADER (...);    -- 32 characters, spells out "HEADER"
I_CUST_PUR_ORD_TOT_AMT                                  -- invented abbreviations
I_INVOICE_STATUS  -- in T_INVOICE_H
```
- **N5 and N3:**
  - Use `T_PURCHASE_ORDER_H`, or `T_PO_H` if `PO` is added to the approved abbreviations.
  - Use `I_TOTAL_AMT` and `I_STATUS`. Don't repeat the table name, and don't invent abbreviations.

### 3.7 Unnecessary Header/Detail
```
T_LOGIN_H (I_LOGIN_ID, I_USER_ID, I_LOGIN_AT)
T_LOGIN_D (I_LOGIN_ID, I_INTERNAL_NO, I_IP_ADDR)   -- always exactly one row
```
- **H2:** there is no one-to-many relationship here.
- **Fix:** use one table, `T_LOGIN_LOG (I_LOGIN_LOG_ID, I_USER_ID, I_LOGIN_AT, I_IP_ADDR)`.

### 3.8 Floating-point money and three-state booleans
```sql
I_TOTAL_AMT  FLOAT
I_IS_PAID    BIT NULL      -- NULL / 0 / 1
```
- **D2:** use `DECIMAL(19,4)`.
- **D3:** use `CHAR(1)` with `'Y'`/`'N'`, NOT NULL. If "unknown" is a real state, it belongs in `I_STATUS`.
