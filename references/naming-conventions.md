# Naming Conventions

These rules apply to every vendor. The examples are lowercase, which is the default project case. In an uppercase project, write the same names in uppercase (`M_CUSTOMER`, `I_CUSTOMER_ID`, `PK_M_CUSTOMER`); nothing else changes.

## 1. Characters, case and length (N1, N5)

- Use only letters of the project case, `0–9` and `_`. A name starts with a letter and has no spaces, no `__` and no trailing `_`. Replace spaces with `_`.
- **One case for the whole database.** Never mix `m_customer` and `M_ORDER`, or `i_` and `I_`.
- Use English words in the singular: `m_product`, not `m_products`.
- Table and column names are at most 30 characters, including the prefix. Constraint, index and other object names are at most 63 characters. Both limits fit every supported vendor.
- Use at most 3 words after the prefix where possible. If a name is long, use an approved abbreviation or a shorter word with the same meaning (`t_stock_move` rather than `t_stock_movement`).

### Approved abbreviations

Use these and no others, the same way everywhere. To add one, revise this list first.

| Abbr. | Meaning | Abbr. | Meaning |
|---|---|---|---|
| `id` | identifier (string key) | `nm` | name |
| `no` | number within a parent, or a document number | `desc` | description |
| `amt` | amount of money | `addr` | address |
| `qty` | quantity | `tel` | telephone |
| `seq` | counter or sequence | `ccy` | ISO 4217 currency code |
| `sec` | seconds (durations) | `func` | function (program function) |
| `hist` | history | `usr` | user |
| `ord` | order | `pkg` | package |

`desc`, `no` and similar words are reserved words on their own. The `i_` prefix is what makes `i_desc` and `i_no` safe.

## 2. Tables (N2, N3)

| Pattern | Use | Example |
|---|---|---|
| `m_<entity>` | Master: reference data, business entities, configuration | `m_customer`, `m_payment_method` |
| `t_<event>` | Transaction: events, activities, operational records | `t_stock_move` |
| `t_<event>_h` / `t_<event>_d` | Header and detail of one transaction | `t_invoice_h`, `t_invoice_d` |
| `t_<event>_<child>_d` | A second child collection, or a detail of a detail | `t_invoice_pay_d`, `t_invoice_lot_d` |
| `m_<a>_<b>` | Junction between masters | `m_product_category` |

**Suffixes (N3).** `_h` and `_d` are the only suffixes with a structural meaning. The last word of an ordinary name, as in `t_login_log` or `t_price_hist`, is part of the name, not a suffix. Never end a table that is not a header in `_h`, or one that is not a detail in `_d`.

Tables that don't obviously fit:

- **Junction tables** take the class of what they link. Two masters linked together give an `m_` table. A link that records an event gives a `t_` table.
- **Logs, audit history, imports and staging** record events or work, so they are `t_` (`t_login_log`, `t_price_hist`).
- **Lookup lists, settings and counters** are configuration, so they are `m_` (`m_id_seq`).
- **A master with line items**, such as a price list and its prices, is still a master. It may use `_h`/`_d` under the same key rules: `m_price_list_h`, `m_price_list_d`.

## 3. Columns (N4, N5)

Every column starts with `i_`. The name is `i_<words>[_<suffix>]`, and the suffix tells you what kind of data the column holds.

| Suffix / form | Meaning | Logical type | Example |
|---|---|---|---|
| `_id` | String key of an entity | `string(20) ascii` | `i_customer_id` |
| `i_internal_no`, `i_sub_no` | Line number within a parent | `string(6) ascii` | `000001` |
| `_no` (other) | Business number that is not a key | `string(n) ascii` | `i_tax_no` |
| Code (no suffix) | A code named for what it is | `string(n) ascii` | `i_status`, `i_ccy`, `i_sku` |
| `_at` | Instant in Unix seconds | `timestamp` | `i_invoice_at` |
| `_date` | Calendar date (00:00 UTC, Unix seconds) | `timestamp` | `i_due_date` |
| `_time` | Time of day in seconds (0–86399) | `int` | `i_open_time` |
| `_sec` | Duration in seconds | `long` | `i_wait_sec` |
| `_amt` | Money | `decimal(19,4)` | `i_total_amt` |
| `_qty` / `i_qty` | Quantity | `decimal(19,4)` | `i_qty` |
| `_price` | Unit price | `decimal(19,4)` | `i_unit_price` |
| `_rate` | Ratio (0.07 = 7%) | `decimal(9,6)` | `i_tax_rate` |
| `_nm` | Display name | `string(n)` | `i_customer_nm` |
| `i_is_…`, `i_has_…` | Boolean | `boolean` | `i_is_taxable` |
| `i_active` | Active / soft-delete flag | `boolean` | — |
| `i_status` | Lifecycle state | `string(20) ascii` | `completed` |
| `i_desc`, `i_note` | Free text | `string(n)` | — |

Further column rules:

- **PK column:** `i_<entity>_id`, where `entity` is the table name without `m_`/`t_` and without `_h`/`_d`. So `t_invoice_h` has `i_invoice_id`, and `m_payment_method` has `i_payment_method_id`.
- **FK column:** the same name as the PK it references (`i_customer_id` in `t_invoice_h`). When one table references the same parent twice, or the meaning differs, put a role word in front: `i_from_warehouse_id`, `i_to_warehouse_id`.
- **Don't repeat the table name** in non-key columns unless the name would be unclear without it. `m_customer.i_customer_nm` is fine because `i_nm` would be ambiguous in joins and views. Avoid `t_invoice_h.i_invoice_status`; use `i_status`.
- **Audit and active columns** always have exactly these names: `i_created_at`, `i_created_func_nm`, `i_created_by`, `i_updated_at`, `i_updated_func_nm`, `i_updated_by`, `i_active`.

## 4. Constraints, indexes and other objects (N6)

`<table>` is the full table name. `<cols>` is the column names without `i_`, joined by `_`. `<parent>` is the referenced table name without its `m_`/`t_` prefix.

| Object | Pattern | Example |
|---|---|---|
| Primary key | `pk_<table>` | `pk_t_invoice_d` |
| Foreign key | `fk_<table>_<parent>`; when the same parent is referenced twice, `fk_<table>_<role>_<parent>` | `fk_t_invoice_d_invoice_h`, `fk_t_stock_move_from_warehouse` |
| Unique | `uq_<table>_<cols>` | `uq_m_product_sku` |
| Index | `ix_<table>_<cols>` | `ix_t_invoice_h_customer_id` |
| Check | `ck_<table>_<cols>`, or `ck_<table>_<rule>` when one check covers several columns | `ck_t_invoice_d_qty`, `ck_t_stock_move_warehouse` |
| Default (SQL Server) | `df_<table>_<cols>` | `df_m_customer_created_at` |
| Sequence | `sq_<table>` | `sq_m_customer` |
| View | `v_<name>` | `v_m_customer` (active rows) |
| Trigger | `tr_<table>_<event>` (`ins`, `upd`, `del`) | `tr_t_invoice_h_upd` |
| Procedure | `p_<verb>_<noun>` | `p_next_id` |
| Function | `fn_<name>` | `fn_next_id` |

Do not start SQL Server procedures with `sp_`: SQL Server looks in the `master` database first for those names. Object names include the table name because PostgreSQL, Oracle and SQLite require index and constraint names to be unique across the whole schema.

If a name would exceed 63 characters, shorten `<cols>` with approved abbreviations or keep only the first column. Never cut a word in half.

## 5. Reserved words and quoting (N7)

**Reserved words.** Every table starts with `m_` or `t_`, every column with `i_`, and every other object with one of the prefixes in section 4. No vendor reserves a word with those prefixes, so names built by these rules never collide with a keyword. Check by hand only names that break the pattern: schema or database names, and any approved deviation. Use the vendors' own keyword lists:

| Vendor | Keyword list |
|---|---|
| SQL Server | "Reserved Keywords (Transact-SQL)" in the Microsoft docs |
| PostgreSQL | Appendix C "SQL Key Words" |
| MySQL / MariaDB | "Keywords and Reserved Words" / "Reserved Words" |
| Oracle | `SELECT keyword FROM v$reserved_words WHERE reserved = 'Y'` |
| SQLite | "SQL As Understood By SQLite: SQLite Keywords" |

**Quoting.** Write names unquoted unless the vendor needs quotes to keep the project case:

| Vendor | Lowercase project | Uppercase project |
|---|---|---|
| SQL Server | Unquoted. Names are stored as written | Unquoted |
| PostgreSQL | Unquoted. Unquoted names fold to lowercase | **Quote every name, everywhere**: `"M_CUSTOMER"` |
| MySQL / MariaDB | Unquoted. Lowercase works on every `lower_case_table_names` setting | Unquoted, with `lower_case_table_names = 0` on every server, including Windows and macOS. Otherwise names become lowercase |
| Oracle | Unquoted. Oracle stores unquoted names in uppercase and matches them without regard to case, so `m_customer` and `M_CUSTOMER` are the same name. Never quote, or the name becomes case-sensitive | Unquoted |
| SQLite | Unquoted. Names are matched without regard to case | Unquoted |

When quoting is needed, configure the ORM or query builder to quote every identifier. Mixing quoted and unquoted forms creates two objects or fails with "does not exist".

## 6. Quick check

- [ ] Project case only, `a–z0-9_` (or `A–Z0-9_`), starts with a letter, no spaces, `__` or trailing `_`
- [ ] Table has `m_` or `t_`, chosen by purpose; only headers end in `_h` and only details end in `_d`
- [ ] Every column starts with `i_` and has the right suffix for its type
- [ ] Table and column names ≤ 30, object names ≤ 63; only approved abbreviations
- [ ] All six audit columns and `i_active` have their exact names
- [ ] FK columns have the same name as the referenced PK, or a role word plus that name
- [ ] Constraint and index names follow section 4
- [ ] No name is a reserved word; quoting follows section 5 for every target vendor
