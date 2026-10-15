# Unix Time Standard (U1–U3)

## 1. Definition (logical, the same on every DBMS)

| Property | Rule |
|---|---|
| Value | Whole **seconds** since `1970-01-01T00:00:00Z` (the Unix epoch) |
| Time zone | Always UTC. A Unix time has no time zone; local time exists only in the application's display |
| Precision | One second. Anything finer is dropped, rounding down: `floor(seconds)` |
| Storage | Signed 64-bit integer (`BIGINT`). 32-bit fails on 2038-01-19, and signed allows dates before 1970 |
| Valid range | Year 0001 to 9999, which is `-62135596800` to `253402300799`. A CHECK on the range is optional |
| Leap seconds | Ignored, as POSIX defines it: every day is exactly 86400 seconds |
| NULL | Only when the event has not happened or the value is unknown, e.g. `I_DELETED_AT`, `I_PAID_AT` |

Never store a date or time as a DBMS date/time type, a formatted string (`'2026-05-03'`) or milliseconds. A DBMS date/time type may appear only inside a conversion expression or a view used for display.

## 2. Kinds of time values

| Kind | Suffix | Stored as | Example value | Meaning |
|---|---|---|---|---|
| Instant | `_AT` | `UNIXTIME` | `1777793415` | 2026-05-03 14:30:15 Bangkok time (07:30:15 UTC) |
| Calendar date | `_DATE` | `UNIXDATE`: Unix time of 00:00:00 **UTC** on that date | `1777766400` | 2026-05-03 |
| Time of day | `_TIME` | Integer seconds since local midnight, 0–86399 | `30600` | 08:30 |
| Duration | `_SEC` | Integer seconds, ≥ 0 | `5400` | 1 h 30 min |

### Instants (`_AT`)
- Record the moment something happened: `I_CREATED_AT`, `I_INVOICE_AT`, `I_PAID_AT`.
- Convert to the user's time zone only for display.

### Calendar dates (`_DATE`)
These are birth dates, due dates and holidays: values that mean a day, not a moment.

- Store the Unix time of 00:00:00 UTC on that date. The value is always divisible by 86400, so add a CHECK: `CK_<TABLE>_<COL>`: `<col> % 86400 = 0`.
- Convert it to a date **in UTC** both ways. **Never** apply a time-zone offset. If you did, 2026-05-03 would show as 2026-05-02 in time zones west of UTC.
- This keeps the date the same no matter where it is read, and dates still sort and compare as numbers.

### Times of day (`_TIME`) and durations (`_SEC`)
- These are not points in time, so they are not Unix time.
- A `_TIME` value is wall-clock time in the business time zone. Constrain it with CHECK `BETWEEN 0 AND 86399`.
- A `_SEC` value is a length of time. Constrain it with CHECK `>= 0`.

## 3. Common operations

| Task | Logical rule |
|---|---|
| Current time | `floor(now_utc - epoch)` in seconds, from the DB (see the DBMS file) or from the application. Pick one source per column and keep to it |
| Store a local date and time from the user | Convert from the user's time zone to UTC in the application, then to seconds |
| Query one **local** business day | Use the range `[start, start + 86400)`, where `start` is local midnight converted to UTC. Bangkok 2026-05-03 is `1777741200 ≤ x < 1777827600`. Calculate the bounds in the application or with the DB's time-zone functions |
| Query one **calendar date** (`_DATE`) | Equality: `I_DUE_DATE = 1777766400` |
| Add days to a `_DATE` | `+ 86400 * n`, which is safe because UTC has no daylight-saving changes |
| Add months | Convert to a date in UTC, add the months, convert back. Never add a fixed number of seconds |
| Age, or difference between dates | Calculate on calendar dates in UTC in the application or SQL; never divide seconds by 365 days |

Application examples, all producing whole seconds:

| Language | Now | Date → `_DATE` value |
|---|---|---|
| Python | `int(time.time())` | `calendar.timegm(date(2026,5,3).timetuple())` → `1777766400` |
| JavaScript | `Math.floor(Date.now() / 1000)` | `Date.UTC(2026, 4, 3) / 1000` |
| Java | `Instant.now().getEpochSecond()` | `LocalDate.of(2026,5,3).atStartOfDay(ZoneOffset.UTC).toEpochSecond()` |
| C# | `DateTimeOffset.UtcNow.ToUnixTimeSeconds()` | `new DateTimeOffset(2026,5,3,0,0,0,TimeSpan.Zero).ToUnixTimeSeconds()` |

## 4. Rules that connect to other standards

- **K1:** a time value is never part of a PK or FK. To give each customer one row per day, use a `UQ_` on `(I_CUSTOMER_ID, I_<X>_DATE)`, not a composite PK containing the date.
- **N4:** time columns keep the `I_` prefix, e.g. `I_CREATED_AT`.
- **ID date part:** the `YYYYMMDD` inside a document ID is a readable label in the business time zone. The authoritative time is the `_AT` column. They can legitimately disagree: an invoice made at 01:00 Bangkok time carries the Bangkok date, but its `_AT` falls on the previous UTC day.

## 5. Implementation requirements for each DBMS file

Every DBMS file must state:
1. the physical type for `UNIXTIME`, `UNIXDATE`, `TIMEOFDAY` and `DURATION`
2. the expression for the current Unix time in seconds, and whether it rounds down
3. the conversion from Unix time to the engine's date/time type, and back, without overflow after 2038
4. the conversion for `_DATE` values in UTC
5. how to query a local business day
