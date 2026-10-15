# Naming Conventions

These rules apply to every DBMS. Names are written in uppercase in the design and in the DDL. Each DBMS file explains how to keep the uppercase form; for example, PostgreSQL needs quoted identifiers.

## 1. Characters and length (N1, N5)

- Use only `A–Z`, `0–9` and `_`. A name starts with a letter and has no `__` and no trailing `_`.
- Separate words with `_`. Use English words in the singular: `M_PRODUCT`, not `M_PRODUCTS`.
- Table and column names are at most 30 characters, including the prefix. Constraint, index and other object names are at most 63 characters.
- Use at most 3 words after the prefix where possible. If a name feels long, use an approved abbreviation or a shorter word with the same meaning (`T_STOCK_MOVE` rather than `T_STOCK_MOVEMENT`). Do not invent abbreviations.

### Approved abbreviations

Use these and no others. Use them the same way everywhere.

| Abbreviation | Meaning | Abbreviation | Meaning |
|---|---|---|---|
| `ID` | identifier (string key) | `DESC` | description |
| `NO` | number within a parent, or document number | `ADDR` | address |
| `AMT` | amount of money | `TEL` | telephone |
| `QTY` | quantity | `CCY` | ISO 4217 currency code |
| `SEQ` | counter or sequence | `SEC` | seconds (durations) |
| `HIST` | history table | | |

`DESC`, `NO` and similar words are SQL reserved words or keywords on their own. That is why the `I_` prefix is mandatory: `I_DESC` is always a safe name.

## 2. Tables (N2, N3)

| Pattern | Use | Example |
|---|---|---|
| `M_<ENTITY>` | Master: reference data, business entities, configuration | `M_CUSTOMER`, `M_PAYMENT_METHOD` |
| `T_<EVENT>` | Transaction: events, activities, operational records | `T_STOCK_MOVE` |
| `T_<EVENT>_H` / `T_<EVENT>_D` | Header and detail of one transaction | `T_INVOICE_H`, `T_INVOICE_D` |
| `T_<EVENT>_<CHILD>_D` | A second child collection, or a detail of a detail | `T_INVOICE_PAY_D`, `T_INVOICE_LOT_D` |
| `M_<A>_<B>` | Junction between masters | `M_PRODUCT_CATEGORY` |

Some tables don't obviously fit. Classify them like this:

- **Junction tables** take the class of what they link. Two masters linked together give an `M_` table. A link that records an event gives a `T_` table.
- **Logs, audit history, imports and staging** record events or work, so they are `T_` (`T_LOGIN_LOG`, `T_PRICE_HIST`).
- **Lookup lists, settings and counters** are configuration, so they are `M_` (`M_DOC_SEQ`).
- **A master with line items**, such as a price list with its prices, is still a master. It may use `_H`/`_D` under the same key rules: `M_PRICE_LIST_H`, `M_PRICE_LIST_D`.

## 3. Columns (N4, N5)

Every column starts with `I_`. The name is built as `I_<WORDS>[_<SUFFIX>]`, and the suffix tells you what kind of data the column holds.

| Suffix / form | Meaning | Logical type | Example |
|---|---|---|---|
| `_ID` | String key of an entity | `ID(n)` | `I_CUSTOMER_ID` |
| `I_INTERNAL_NO`, `I_SUB_NO` | Line number within a parent | `LINENO` | `000001` |
| `_NO` (other) | Business number that is not a key | `CODE(n)` | `I_TAX_NO` |
| `_AT` | Instant in Unix seconds | `UNIXTIME` | `I_INVOICE_AT` |
| `_DATE` | Calendar date (00:00 UTC) | `UNIXDATE` | `I_DUE_DATE` |
| `_TIME` | Time of day in seconds (0–86399) | `TIMEOFDAY` | `I_OPEN_TIME` |
| `_SEC` | Duration in seconds | `DURATION` | `I_WAIT_SEC` |
| `_AMT` | Money | `AMOUNT` | `I_TOTAL_AMT` |
| `_QTY` / `I_QTY` | Quantity | `QTY` | `I_QTY` |
| `_PRICE` | Unit price | `AMOUNT` | `I_UNIT_PRICE` |
| `_RATE` | Ratio (0.07 = 7%) | `RATE` | `I_TAX_RATE` |
| `_NAME` | Display name | `TEXT(n)` | `I_CUSTOMER_NAME` |
| `I_IS_…`, `I_HAS_…` | Boolean | `FLAG` | `I_IS_ACTIVE` |
| `I_STATUS` | Lifecycle state | `CODE(20)` | `COMPLETED` |
| `I_DESC`, `I_NOTE` | Free text | `TEXT(n)` | — |

Further column rules:

- **PK column:** `I_<ENTITY>_ID`, where `ENTITY` is the table name without `M_`/`T_` and without `_H`/`_D`. So `T_INVOICE_H` has `I_INVOICE_ID`, and `M_PAYMENT_METHOD` has `I_PAYMENT_METHOD_ID`.
- **FK column:** the same name as the PK it references (`I_CUSTOMER_ID` in `T_INVOICE_H`). When one table references the same parent twice, or the meaning differs, put a role word in front: `I_FROM_WAREHOUSE_ID`, `I_TO_WAREHOUSE_ID`.
- **Don't repeat the table name** in non-key columns unless the name would be unclear without it. `M_CUSTOMER.I_CUSTOMER_NAME` is fine because `I_NAME` would be ambiguous in joins and views. Avoid `T_INVOICE_H.I_INVOICE_STATUS`; use `I_STATUS`.
- **Audit columns** are always named `I_CREATED_AT`, `I_CREATED_BY`, `I_UPDATED_AT`, `I_UPDATED_BY`, and `I_DELETED_AT` for soft delete.

## 4. Constraints, indexes and other objects (N6)

`<TABLE>` is the full table name. `<COLS>` is the column names without `I_`, joined by `_`. `<PARENT>` is the referenced table name without its `M_`/`T_` prefix.

| Object | Pattern | Example |
|---|---|---|
| Primary key | `PK_<TABLE>` | `PK_T_INVOICE_D` |
| Foreign key | `FK_<TABLE>_<PARENT>`; when the same parent is referenced twice, `FK_<TABLE>_<ROLE>_<PARENT>` | `FK_T_INVOICE_D_INVOICE_H`, `FK_T_STOCK_MOVE_FROM_WAREHOUSE` |
| Unique | `UQ_<TABLE>_<COLS>` | `UQ_M_PRODUCT_SKU` |
| Index | `IX_<TABLE>_<COLS>` | `IX_T_INVOICE_H_CUSTOMER_ID` |
| Check | `CK_<TABLE>_<COLS>`, or `CK_<TABLE>_<RULE>` when one check covers several columns | `CK_T_INVOICE_D_QTY`, `CK_T_STOCK_MOVE_WAREHOUSE` |
| Default (SQL Server) | `DF_<TABLE>_<COLS>` | `DF_M_CUSTOMER_CREATED_AT` |
| Sequence | `SQ_<TABLE>` | `SQ_M_CUSTOMER` |
| View | `V_<NAME>` | `V_INVOICE_SUMMARY` |
| Trigger | `TR_<TABLE>_<EVENT>` (`INS`, `UPD`, `DEL`) | `TR_T_INVOICE_H_UPD` |
| Procedure | `P_<VERB>_<NOUN>` | `P_NEXT_DOC_ID` |
| Function | `FN_<NAME>` | `FN_NEXT_DOC_ID` |

Do not start SQL Server procedures with `SP_`: SQL Server checks the `master` database first for names beginning `sp_`. Object names include the table name because some DBMSs require constraint and index names to be unique across the whole schema.

If a name would exceed 63 characters, shorten `<COLS>` with approved abbreviations or keep only the first column. Never truncate a name in the middle of a word.

## 5. Quick check

- [ ] Uppercase, `A–Z0-9_`, starts with a letter, no `__` or trailing `_`
- [ ] Table has `M_` or `T_`, chosen by purpose; header and detail have `_H`/`_D`
- [ ] Every column starts with `I_` and has the right suffix for its type
- [ ] Table and column names ≤ 30, object names ≤ 63; only approved abbreviations
- [ ] FK columns have the same name as the referenced PK, or a role word plus that name
- [ ] Constraint and index names follow section 4
