# Unix Time Standard (U1–U3)

## 1. Definition (logical, the same on every vendor)

| Property | Rule |
|---|---|
| Value | Whole **seconds** since `1970-01-01T00:00:00Z` (the Unix epoch), the same convention Nostr uses for `created_at` |
| Unit | Seconds only. **Never** milliseconds or microseconds, in the database, the API, or any integration |
| Logical type | `timestamp` (see `logical-types.md`): a signed 64-bit integer on every vendor |
| Time zone | Always UTC. A Unix time has no time zone; local time exists only in what the application displays |
| Precision | One second. Anything finer is dropped, rounding down: `floor(seconds)` |
| Range | Signed 64-bit: 32-bit fails on 2038-01-19, and signed allows dates before 1970. Valid calendar range is year 0001 to 9999, `-62135596800` to `253402300799` |
| Leap seconds | Ignored, as POSIX defines it: every day is exactly 86400 seconds |
| NULL | Only when the event has not happened or the value is unknown, e.g. `i_paid_at` |

Never store a date or time in a vendor date/time type, as a formatted string (`'2026-10-17'`) or in milliseconds. A vendor date/time type may appear only inside a conversion expression or a display view.

**How to recognise the unit.** A present-day value in seconds has 10 digits (`1792200000`, October 2026). A 13-digit value is milliseconds and is a defect.

## 2. Kinds of time values

| Kind | Suffix | Logical type | Example value | Meaning |
|---|---|---|---|---|
| Instant | `_at` | `timestamp` | `1792222215` | 2026-10-17 14:30:15 Bangkok time (07:30:15 UTC) |
| Calendar date | `_date` | `timestamp` holding 00:00:00 **UTC** of the date | `1792195200` | 2026-10-17 |
| Time of day | `_time` | `int`: seconds since local midnight, 0–86399 | `30600` | 08:30 |
| Duration | `_sec` | `long`: seconds, ≥ 0 | `5400` | 1 h 30 min |

### Instants (`_at`)
- Record the moment something happened: `i_created_at`, `i_invoice_at`, `i_paid_at`.
- Convert to the user's time zone only for display.

### Calendar dates (`_date`)
These are birth dates, due dates and holidays: values that mean a day, not a moment. They use the **same unit**, Unix seconds.

- Store the Unix time of 00:00:00 UTC on that date. The value is always divisible by 86400, so add a CHECK `ck_<table>_<col>`: `<col> % 86400 = 0` (Oracle: `MOD(<col>, 86400) = 0`).
- Convert it to a date **in UTC** both ways. **Never** apply a time-zone offset: if you did, 2026-10-17 would show as 2026-10-16 in time zones west of UTC.
- The date is then the same wherever it is read, and dates still sort and compare as numbers.

### Times of day (`_time`) and durations (`_sec`)
- These are not points in time, so they are not Unix time.
- A `_time` value is wall-clock time in the business time zone. CHECK `BETWEEN 0 AND 86399`.
- A `_sec` value is a length of time. CHECK `>= 0`.

## 3. Common operations

| Task | Logical rule |
|---|---|
| Current time | `floor(now_utc - epoch)` in seconds, from the database (see the vendor file) or the application. Pick one source per column and keep to it |
| Store a local date and time entered by a user | Convert from the user's time zone to UTC in the application, then to seconds |
| Query one **local** business day | Range `[start, start + 86400)`, where `start` is local midnight converted to UTC. Bangkok 2026-10-17 is `1792170000 ≤ x < 1792256400`. Calculate the bounds in the application or with the vendor's time-zone functions |
| Query one **calendar date** (`_date`) | Equality: `i_due_date = 1792195200` |
| Add days to a `_date` | `+ 86400 * n`, which is safe because UTC has no daylight-saving changes |
| Add months | Convert to a date in UTC, add the months, convert back. Never add a fixed number of seconds |
| Age, or difference between dates | Calculate on calendar dates in UTC; never divide seconds by 365 days |

Application examples, all producing whole seconds:

| Language | Now | Date → `_date` value |
|---|---|---|
| Python | `int(time.time())` | `calendar.timegm(date(2026,10,17).timetuple())` → `1792195200` |
| JavaScript | `Math.floor(Date.now() / 1000)` (`Date.now()` is milliseconds) | `Date.UTC(2026, 9, 17) / 1000` |
| Java | `Instant.now().getEpochSecond()` | `LocalDate.of(2026,10,17).atStartOfDay(ZoneOffset.UTC).toEpochSecond()` |
| C# | `DateTimeOffset.UtcNow.ToUnixTimeSeconds()` | `new DateTimeOffset(2026,10,17,0,0,0,TimeSpan.Zero).ToUnixTimeSeconds()` |
| Go | `time.Now().Unix()` | `time.Date(2026, 10, 17, 0, 0, 0, 0, time.UTC).Unix()` |

## 4. Rules that connect to other standards

- **K1:** a time value is never part of a PK or FK. For one row per customer per day, use a `uq_` on `(i_customer_id, i_<x>_date)`, not a composite PK containing the date.
- **N4:** time columns keep the `i_` prefix, e.g. `i_created_at`.
- **D1:** `i_created_at` and `i_updated_at` follow this standard exactly.
- **ID date part (K7):** the `yyyymmdd` inside an ID is a readable label in the business time zone. The authoritative time is the `_at` column. They can legitimately disagree: an order made at 01:00 Bangkok time carries the Bangkok date, but its `_at` falls on the previous UTC day.
- **APIs and integrations** exchange times as integer Unix seconds. A JSON field holding a time is a number of seconds, never an ISO string or milliseconds, unless an external contract requires otherwise; convert at that boundary.

## 5. Implementation requirements for each vendor file

Every vendor file states:
1. the physical type for `timestamp` and for `_time` / `_sec` values
2. the expression for the current Unix time in seconds, and whether it rounds down
3. the conversion from Unix time to the engine's date/time type, and back, without overflow after 2038
4. the conversion for `_date` values in UTC
5. how to query a local business day
