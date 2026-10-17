# Logical Types and Vendor Mapping (L1, L2)

Design every column with a logical type. The logical schema never names a vendor type. This file is the type-mapping specification for every supported vendor; each `dbms/<vendor>.md` file repeats its own column with usage notes.

## 1. Logical types

| Logical type | Meaning | Typical columns |
|---|---|---|
| `string(n)` | Unicode text, at most `n` **characters** | names, descriptions, addresses, notes |
| `string(n) ascii` | ASCII-only string, compared in binary | IDs, `i_internal_no`, codes, statuses, currency, audit `_by` / `_func_nm` |
| `int` | 32-bit signed integer | `_time` (seconds of day), counters |
| `long` | 64-bit signed integer | `_sec` durations, large counters |
| `boolean` | `true` / `false` | `i_active`, `i_is_…`, `i_has_…` |
| `decimal(p,s)` | Exact decimal, `p` digits with `s` after the point | money, quantities, rates |
| `timestamp` | Unix time: whole seconds since `1970-01-01T00:00:00Z`, signed 64-bit | `_at`, `_date` |

**Which strings are `ascii` (L2).** Only values the system generates or that follow a fixed code list: IDs and key columns (K1), line numbers, business codes (SKU, tax number), statuses, ISO codes, and the technical audit columns `_by` and `_func_nm`. Everything a person types or reads as language is `string(n)`, Unicode. When in doubt, use `string(n)`.

`timestamp` is a `long` with a fixed meaning. It is listed separately so that every time column says what it holds; its physical type is always the vendor's 64-bit integer.

## 2. Standard sizes

Use one of these sizes; don't pick a different width for each column.

| Use | Logical type |
|---|---|
| Entity ID (PK and FK) | `string(20) ascii` |
| `i_internal_no`, `i_sub_no` | `string(6) ascii` |
| Status | `string(20) ascii` |
| Currency (ISO 4217) | `string(3) ascii` |
| Business code (SKU, tax number) | `string(30) ascii` |
| `i_created_by`, `i_updated_by` | `string(50) ascii` |
| `i_created_func_nm`, `i_updated_func_nm` | `string(100) ascii` |
| Name | `string(200)` |
| Description, address | `string(500)` |
| Note | `string(2000)` |
| Email | `string(254)` |
| Money, quantity | `decimal(19,4)` |
| Rate (tax, discount, exchange) | `decimal(9,6)` |

## 3. Mapping

| Logical | SQL Server | PostgreSQL | MySQL / MariaDB | Oracle | SQLite (STRICT) |
|---|---|---|---|---|---|
| `string(n)` | `NVARCHAR(n)`; `NVARCHAR(MAX)` above 4000 | `VARCHAR(n)` in a `UTF8` database | `VARCHAR(n)` with `CHARACTER SET utf8mb4` | `VARCHAR2(n CHAR)` in an `AL32UTF8` database; see the note for `n` > 1000 | `TEXT` + CHECK `length(col) <= n` |
| `string(n) ascii` | `VARCHAR(n) COLLATE Latin1_General_100_BIN2` | `VARCHAR(n) COLLATE "C"` | `VARCHAR(n) CHARACTER SET ascii COLLATE ascii_bin` | `VARCHAR2(n BYTE)` | `TEXT` + CHECK on length and characters |
| `int` | `INT` | `INTEGER` | `INT` | `NUMBER(10)` | `INTEGER` |
| `long` | `BIGINT` | `BIGINT` | `BIGINT` | `NUMBER(19)` | `INTEGER` |
| `boolean` | `BIT` | `BOOLEAN` | `BOOLEAN` (stored as `TINYINT(1)`) + CHECK `IN (0,1)` | `NUMBER(1)` + CHECK `IN (0,1)`; `BOOLEAN` on 23ai or later | `INTEGER` + CHECK `IN (0,1)` |
| `decimal(p,s)` | `DECIMAL(p,s)` | `NUMERIC(p,s)` | `DECIMAL(p,s)` | `NUMBER(p,s)` | `INTEGER` holding the value × 10^s (see §5) |
| `timestamp` | `BIGINT` | `BIGINT` | `BIGINT` | `NUMBER(19)` | `INTEGER` |

Boolean literals: `1`/`0` on SQL Server, MySQL, Oracle and SQLite; `true`/`false` on PostgreSQL (MySQL also accepts `TRUE`/`FALSE`).

## 4. Rules that apply on every vendor

- **Same physical type on both sides of an FK (K3).** Copy the PK column's definition, including collation and character set. Some vendors reject a mismatch; others silently skip the index.
- **No vendor date/time types (U1).** `DATE`, `DATETIME`, `TIMESTAMP`, `TIMESTAMPTZ`, `INTERVAL` and similar never appear as column types. The SQL `TIMESTAMP` type is unrelated to the logical type `timestamp`.
- **No floating point or money types (D4).** `FLOAT`, `REAL`, `DOUBLE`, `BINARY_DOUBLE`, `MONEY` and `SMALLMONEY` are never used for business numbers.
- **No database-generated keys (K1).** No `IDENTITY`, `SERIAL`, `AUTO_INCREMENT` or SQLite `INTEGER PRIMARY KEY` row IDs.
- **Character lengths.** `n` counts characters. Each vendor file states how its type gets character semantics.

## 5. Vendor conflicts and their compatible solutions

These are the places where a vendor cannot implement a logical type directly. The logical type stays the same; only the physical form changes.

| Vendor | Conflict | Solution |
|---|---|---|
| SQLite | No exact decimal type. `NUMERIC`/`DECIMAL` store fractional values as 8-byte floating point | Store `decimal(p,s)` as `INTEGER` scaled by 10^s: `150.0000` with scale 4 is stored as `1500000`. The data-access layer converts on read and write. `SUM` stays exact; multiplication must rescale. The largest value is 922,337,203,685,477.5807 at scale 4, below `decimal(19,4)`'s full range |
| SQLite | Declared types and lengths are not enforced | Use `STRICT` tables and a CHECK on `length()` for every string |
| Oracle | No `BOOLEAN` column type before 23ai | `NUMBER(1)` + CHECK `IN (0,1)`; `1` is true |
| Oracle | `VARCHAR2` holds at most 4000 bytes, which is about 1000 characters of 4-byte UTF-8 (1333 for Thai) | For `string(n)` with `n` > 1000: `VARCHAR2(n CHAR)` if the database uses `MAX_STRING_SIZE = EXTENDED`, otherwise `CLOB` with CHECK `LENGTH(col) <= n` |
| Oracle | `''` is stored as NULL | Never store empty strings (D5); optional text is nullable |
| MySQL / MariaDB | `BOOLEAN` is `TINYINT(1)` and accepts any small integer | CHECK `IN (0,1)` |
| SQL Server | `NVARCHAR` above 4000 characters needs `MAX`, which can't be an index key | Index only columns up to 4000 characters, which covers every standard size |
