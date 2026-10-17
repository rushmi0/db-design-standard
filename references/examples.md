# Examples

The complete DDL for each vendor is in `dbms/<vendor>.md`, section 5. This file shows the logical design, other patterns that go with it, and common mistakes.

## 1. Compliant logical design

```
m_customer  1 ──< t_invoice_h 1 ──< t_invoice_d >── 1 m_product
                                                        1
m_warehouse 1 ──< t_stock_move (from / to) >───────────┘
m_id_seq    (counter for IDs)
```

| Table | Class | Structure | PK | ID example |
|---|---|---|---|---|
| `m_customer` | Master | single | `i_customer_id` | `cus000001` (no date) |
| `m_product` | Master | single | `i_product_id` | `prd2026100001` (monthly) |
| `m_warehouse` | Master | single | `i_warehouse_id` | `wh0001` (no date) |
| `m_id_seq` | Master (configuration) | single | `i_id_seq_id` | `inv20261017` |
| `t_invoice_h` | Transaction | header | `i_invoice_id` | `inv20261017000001` (daily) |
| `t_invoice_d` | Transaction | detail | `(i_invoice_id, i_internal_no)` | `000001` |
| `t_stock_move` | Transaction | standalone | `i_stock_move_id` | `smv20261017000001` (daily) |

Every table also has the six audit columns and `i_active`.

Why each structure:

- **`t_invoice_h/d`:** an invoice has shared data (customer, time, status, total) plus many lines, so H2 applies.
- **`t_stock_move`:** each movement is a complete record on its own, so it is a single table. It references `m_warehouse` twice, so its FK columns carry the role words `from` and `to`.

### Logical table: `t_invoice_d`

| Column | Logical type | Null | Key / check |
|---|---|---|---|
| `i_invoice_id` | `string(20) ascii` | NOT NULL | PK, FK → `t_invoice_h` |
| `i_internal_no` | `string(6) ascii` | NOT NULL | PK; 6 digits, not `000000` |
| `i_product_id` | `string(20) ascii` | NOT NULL | FK → `m_product` |
| `i_desc` | `string(500)` | NOT NULL | snapshot of the product name |
| `i_qty` | `decimal(19,4)` | NOT NULL | `> 0` |
| `i_unit_price` | `decimal(19,4)` | NOT NULL | `>= 0`, snapshot |
| `i_tax_rate` | `decimal(9,6)` | NOT NULL | `0..1`, snapshot |
| `i_line_amt` | `decimal(19,4)` | NOT NULL | |
| `i_created_at` | `timestamp` | NOT NULL | audit |
| `i_created_func_nm` | `string(100) ascii` | NOT NULL | audit |
| `i_created_by` | `string(50) ascii` | NOT NULL | audit |
| `i_updated_at` | `timestamp` | NOT NULL | audit |
| `i_updated_func_nm` | `string(100) ascii` | NOT NULL | audit |
| `i_updated_by` | `string(50) ascii` | NOT NULL | audit |
| `i_active` | `boolean` | NOT NULL | default `true` |

### Sample rows

| i_invoice_id | i_internal_no | i_product_id | i_desc | i_qty | i_unit_price | i_active |
|---|---|---|---|---|---|---|
| inv20261017000001 | 000001 | prd2026100001 | 商品 A | 2.0000 | 100.0000 | true |
| inv20261017000001 | 000002 | prd2026100002 | สินค้า B | 1.0000 | 150.0000 | false |
| inv20261017000002 | 000001 | prd2026100001 | 商品 A | 5.0000 | 95.0000 | true |

- `000001` appears under both invoices. `i_internal_no` is unique only within its invoice (H1).
- Line `000002` of the first invoice was removed: it is inactive, keeps its number, and is left out of the total (D2).
- The second invoice has a different price for the same product because prices are copied at the time of the transaction (H3).
- Descriptions are Thai and Japanese, stored in Unicode columns (L2).

## 2. Other compliant patterns (logical)

**Junction (many-to-many between masters)**
```
m_product_category
  i_product_id    string(20) ascii   PK, FK -> m_product      (fk_m_product_category_product)
  i_category_id   string(20) ascii   PK, FK -> m_category     (fk_m_product_category_category)
  + six audit columns, i_active
  ix_m_product_category_category_id on (i_category_id)   -- FK not covered by the PK's leading column
```

**Second child collection and sub-detail**
```
t_invoice_pay_d   PK (i_invoice_id, i_internal_no)            FK i_invoice_id -> t_invoice_h
t_invoice_lot_d   PK (i_invoice_id, i_internal_no, i_sub_no)  FK (i_invoice_id, i_internal_no) -> t_invoice_d
                  name: fk_t_invoice_lot_d_invoice_d
```

**One row per customer per day** (a time value is never part of a key)
```
t_daily_limit
  i_daily_limit_id  string(20) ascii   PK  (dl + yyyymmdd + 6 digits)
  i_customer_id     string(20) ascii   FK -> m_customer
  i_limit_date      timestamp          CHECK % 86400 = 0
  uq_t_daily_limit_customer_id_limit_date on (i_customer_id, i_limit_date)
```

**User master and audit columns**
```
m_usr
  i_usr_id   string(20) ascii   PK  usr000001
  i_usr_nm   string(200)
  ...
-- Other tables store 'usr000001' in i_created_by / i_updated_by, with no FK,
-- because batch jobs write process names such as 'batch_import' there.
```

## 3. Non-compliant examples

Each example below breaks the rules listed after it. These are the mistakes to catch in a review.

### 3.1 Integer keys, a date type, and no audit or active columns
```sql
CREATE TABLE InvoiceItems (
    id          INT IDENTITY PRIMARY KEY,
    invoice_id  INT NOT NULL REFERENCES Invoices(id) ON DELETE CASCADE,
    line_no     INT NOT NULL,
    created     DATETIME NOT NULL
);
```

| Problem | Rule | Fix |
|---|---|---|
| Mixed case, plural, no `t_` prefix or `_d` suffix | N1, N2, N3 | `t_invoice_d` |
| Columns without `i_` | N4 | `i_invoice_id`, `i_internal_no`, `i_created_at` |
| Integer surrogate PK `id` | K1, H1 | Composite PK `(i_invoice_id, i_internal_no)`; drop `id` |
| Integer FK `invoice_id` | K1, K3 | `string(20) ascii`, identical to the header PK |
| Integer `line_no` | K1, H1 | `i_internal_no string(6) ascii`, `000001` format |
| `ON DELETE CASCADE` | K5 | `NO ACTION`; remove through `i_active` |
| `DATETIME` column | U1 | `i_created_at timestamp` in Unix seconds |
| Five audit columns and `i_active` missing | D1, D2 | Add them |
| No snapshot columns | H3 | Add `i_desc`, `i_unit_price`, `i_tax_rate` |

### 3.2 Header/Detail keyed by a single detail ID
```sql
CREATE TABLE t_order_d (
    i_order_d_id  VARCHAR(20) PRIMARY KEY,
    i_order_id    VARCHAR(20) NOT NULL REFERENCES t_order_h (i_order_id),
    ...
);
```
- **H1:** the detail PK must be `(i_order_id, i_internal_no)`, not its own single ID.
- **Fix:** replace `i_order_d_id` with `i_internal_no string(6) ascii`, then make the PK composite. The FK index is then the PK's leading column.

### 3.3 IDs that break the format
```
i_customer_id = 'CUS-000001'        -- uppercase and '-'
i_order_id    = 'ord2026-10-17-1'   -- separators, no fixed width
i_order_id    = '8f14e45f-ceea-...' -- random UUID with no documented reason
```
- **K2, K7:** IDs are lowercase `a–z0–9` in the `<abbr><date><sequence>` format: `cus000001`, `ord20261017000001`.
- A UUID needs a documented reason (offline or multi-site creation) and is then written in lowercase hex without dashes.

### 3.4 Mismatched FK definition
```sql
-- m_customer.i_customer_id is the ASCII ID type with a binary collation
i_customer_id NVARCHAR(30) NOT NULL REFERENCES m_customer (i_customer_id)
```
- **K3:** the type, length and collation all differ from the PK.
  - SQL Server and MySQL reject the FK.
  - Where the vendor accepts it, joins compare different types and stop using the index.
- **Fix:** copy the PK definition exactly.

### 3.5 Unix time misuse
```sql
i_order_date  VARCHAR(10)  -- '2026-10-17'
i_paid_at     INT          -- 32-bit
i_ship_at     BIGINT       -- milliseconds: 1792222215000
i_birth_date  BIGINT       -- 1792170000 (Bangkok midnight)
```

| Column | Rule | Problem and fix |
|---|---|---|
| `i_order_date` | U1, U2 | A string date. Use `timestamp` holding UTC midnight: `1792195200` |
| `i_paid_at` | U1 | 32-bit, so it fails in 2038. Use `timestamp` (64-bit) |
| `i_ship_at` | U1 | Milliseconds (13 digits). Store seconds: `1792222215` |
| `i_birth_date` | U2 | Local midnight, which is not divisible by 86400 and fails the CHECK. Use UTC midnight `1792195200` and add `ck_…_birth_date` |

### 3.6 A timestamp in the key
```sql
CONSTRAINT pk_t_daily_limit PRIMARY KEY (i_customer_id, i_limit_date)
```
- **K1:** `i_limit_date` is a number.
- **Fix:** see "One row per customer per day" in section 2: use a string PK plus a `uq_` on `(i_customer_id, i_limit_date)`.

### 3.7 Soft delete done the old way
```sql
i_deleted_at  BIGINT NULL      -- second delete marker
i_is_deleted  CHAR(1)          -- 'Y'/'N'
DELETE FROM m_customer WHERE i_customer_id = 'cus000001';
```
- **D2:** the only active/soft-delete marker is `i_active boolean NOT NULL DEFAULT true`. Drop `i_deleted_at` and `i_is_deleted`.
- **D2:** normal operation never deletes rows. Replace the `DELETE` with an update that sets `i_active` to false and the three `updated` audit columns.
- **D3:** a flag is never `CHAR(1)` with `'Y'`/`'N'`; it uses the vendor's boolean mapping.

### 3.8 Text that can't hold Thai or Japanese
```sql
i_customer_nm  VARCHAR(200)                            -- SQL Server
i_product_nm   VARCHAR(200) CHARACTER SET latin1       -- MySQL
i_desc         VARCHAR2(500 BYTE)                      -- Oracle
```
- **L2:** human text is Unicode on every vendor: `NVARCHAR(200)`, `utf8mb4`, `VARCHAR2(500 CHAR)` in an `AL32UTF8` database. On SQL Server and MySQL these columns silently store `?` for Thai.

### 3.9 Names that are too long or unclear
```sql
CREATE TABLE t_customer_purchase_order_header (...);    -- 32 characters, spells out "header"
i_cust_pur_ord_tot_amt                                  -- invented abbreviations
i_invoice_status  -- in t_invoice_h
i_customer_name   -- name, not nm
```
- **N3, N5:**
  - Use `t_purchase_ord_h`.
  - Use `i_total_amt`, `i_status` and `i_customer_nm`. Don't repeat the table name, and don't invent abbreviations.

### 3.10 Unnecessary Header/Detail
```
t_login_h (i_login_id, i_usr_id, i_login_at)
t_login_d (i_login_id, i_internal_no, i_ip_addr)   -- always exactly one row
```
- **H2:** there is no one-to-many relationship here.
- **Fix:** use one table, `t_login_log (i_login_log_id, i_usr_id, i_login_at, i_ip_addr, …audit, i_active)`.

### 3.11 Floating-point money and three-state booleans
```sql
i_total_amt  FLOAT
i_is_paid    BIT NULL      -- NULL / 0 / 1
```
- **D4:** use `decimal(19,4)`.
- **D3:** a boolean is NOT NULL. If "unknown" is a real state, it belongs in `i_status`.
