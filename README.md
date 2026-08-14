# OpenCart Persistent Cart & InnoDB Lock Contention Reduction

**OCMOD v1.9 for OpenCart 3.x** — sharply reduces MySQL/MariaDB deadlock frequency (`Error 1213`) on the `oc_cart` table and provides a 30-day persistent cart for guest users.

> **Scope, stated honestly:** this modification lowers how often the cart GC runs and how many rows a single GC transaction touches. It makes deadlocks much rarer. It does **not** make them impossible — under InnoDB REPEATABLE READ, concurrent `DELETE` and `INSERT` on the same range can still deadlock. Pair this with a retry wrapper (see below) if you need crash-free behaviour.

---

## The Problem

In stock OpenCart the `Cart` class constructor (`system/library/cart/cart.php`) runs a wide-range `DELETE` **on every single page view**:

```sql
DELETE FROM oc_cart
WHERE (api_id > '0' OR customer_id = '0')
  AND date_added < DATE_SUB(NOW(), INTERVAL 1 HOUR)
```

Two things go wrong:

- The `OR` prevents any useful index from being chosen, so the statement scans the table.
- Under REPEATABLE READ, a scanning `DELETE` takes next-key (row + gap) locks across everything it visits. A concurrent guest `INSERT` into `oc_cart` blocks behind it, and the two can form a lock cycle.

Result, typically in bursts during traffic peaks or ERP/CRM sync:

```
PHP Warning: mysqli::query(): (40001/1213): Deadlock found when trying to get lock;
try restarting transaction in /system/library/db/mysqli.php on line 25
```

Stock behaviour also gives guests a **one-hour** cart, which is far too short for any considered purchase.

---

## What This Modification Does

### 1. Throttled garbage collection
The GC no longer runs on every request. It fires with a **0.1% probability** (`mt_rand(1, 1000) === 500`), which is ample to keep the table trimmed while removing constant lock contention.

### 2. Split queries with `LIMIT`
The single `OR` statement becomes two narrower ones, each capped:

```php
if (mt_rand(1, 1000) === 500) {
    $this->db->query("DELETE FROM oc_cart WHERE api_id > '0'
        AND date_added < DATE_SUB(NOW(), INTERVAL 720 HOUR) LIMIT 200");
    $this->db->query("DELETE FROM oc_cart WHERE api_id = '0' AND customer_id = '0'
        AND date_added < DATE_SUB(NOW(), INTERVAL 720 HOUR) LIMIT 500");
}
```

`LIMIT` is what actually matters here: it bounds the transaction, so lock duration stays short and any leftover rows are cleaned on subsequent GC triggers. Splitting the `OR` only helps once a suitable index exists — see the installation step below.

### 3. Sliding 30-day TTL (fixed in 1.8)
GC deletes by `date_added`, but stock `add()` and `update()` never modify that column — they only change `quantity`. Without a fix, an actively used cart still dies 30 days after the **first** item was added, and because newer rows carry newer timestamps, the cart is emptied **partially**, which is worse than being emptied outright.

v1.8 refreshes `date_added` in two places, without patching any core SQL:

- inside `set_cart_cookie()`, hooked into `add()`, `update()`, `remove()`, `clear()` — throttled by `AND date_added < DATE_SUB(NOW(), INTERVAL 24 HOUR)`, so it normally matches zero rows and costs one indexed lookup
- in the same `UPDATE` that migrates the session ID, so a returning guest resets the window at zero extra cost

**Precise semantics:** the window is extended **no more than once per 24 hours, and only when the cart is modified** (`add`/`update`/`remove`/`clear`), plus on every successful guest session migration. It is not refreshed by ordinary page views, and not on every cart interaction.

### 4. Session ID validated, not sanitised (fixed in 1.8)
`Session::start()` in OpenCart 3 accepts `/^[a-zA-Z0-9,\-]{22,52}$/` and aborts on anything else. Earlier versions of this OCMOD ran `preg_replace('%[^A-Za-z0-9]%', '', ...)` on the cookie, which would strip `,` and `-` from an otherwise valid ID; the migration `UPDATE` would then match nothing and the customer would see an empty cart.

v1.8 validates the raw value and skips the merge if it does not match:

```php
if (preg_match('/^[a-zA-Z0-9,\-]{22,52}$/D', $cart_hash) && $cart_hash !== $current_session) { ... }
```

Validation plus `$this->db->escape()` keeps injection off the table.

### 5. Honest concurrency guard (fixed in 1.8)
Versions up to 1.7 stored `cart_hash_updated` in `$this->session->data` and the README called it a per-request guard. Session data survives the request, so the label was wrong. v1.8 uses a private class property instead. Since stock OpenCart constructs `Cart` once per request, that is genuinely request-scoped — though it still does not synchronise **parallel** HTTP requests, and it is not a deadlock mitigation.

### 6. Cookie hardening with a 7.0 fallback
`SameSite=Lax` plus `HttpOnly` via the PHP 7.3+ array form, falling back to the positional signature on older builds, so the module runs on PHP 7.0+. A `headers_sent()` check prevents warnings if output has already started.

### 7. Quantity aggregation on merge (fixed in 1.9)
Up to 1.8 the migration moved rows verbatim. If the destination session already held the same `product_id` + `recurring_id` + `option`, the customer ended up with two identical lines instead of a summed quantity — stock `add()` folds quantities, a raw `UPDATE ... SET session_id` does not.

v1.9 probes for a collision with one cheap indexed `SELECT COUNT(*)` first. Collisions are rare, so the common path remains a single narrow `UPDATE`. Only when a collision actually exists does it fall through to a transactional three-statement path: fold quantities into the destination rows, drop the folded source rows, then move whatever is left. The probe matters — multi-table `UPDATE`/`DELETE` with a self-join holds locks on two ranges at once, which is exactly the profile this module otherwise avoids.

### 8. Cookie writability as a precondition (fixed in 1.9)
`headers_sent()` is now checked **before** any rows are touched, not just before `setcookie()`. Previously, if headers had already been flushed, the rows were migrated to the new session ID while the `cart_hash` cookie still pointed at the old one. Nothing broke immediately — the live session still resolved the cart — but once that session expired, the rows were orphaned and the cart vanished with no log entry. Now, if the pointer cannot be persisted, the rows stay put and are picked up on a later request.

---

## Technical Features

| Feature | Details |
|---|---|
| Cart lifetime | 30 days from last activity |
| GC probability | 0.1% per request (`mt_rand`) |
| GC bound | `LIMIT 200` (API carts) / `LIMIT 500` (guest carts) |
| TTL refresh | On cart modification (max once per 24h per cart) and on successful session migration |
| Merge behaviour | Quantities aggregated on collision; verbatim move otherwise |
| Cookie failure mode | Migration skipped entirely if headers already sent — rows never orphaned |
| Session ID handling | Validated against OpenCart's own format, never rewritten |
| HTTPS detection | Proxy-aware (`HTTPS` + `X-Forwarded-Proto`) |
| Cookie flags | `HttpOnly`, `SameSite=Lax`, `Secure` (auto-detected) |
| Guest guard | `!$this->customer->isLogged()` |
| PHP compatibility | 7.0+ (array-form `setcookie` used on 7.3+) |

---

## Requirements

- OpenCart 3.0.x (including ocStore and similar distributions)
- MySQL / MariaDB with InnoDB tables
- PHP 7.0+ with the MySQLi driver

---

## Installation

1. Upload `cart_lifetime.ocmod.xml` to the `/system/` directory
2. **Extensions → Modifications → Refresh**
3. Dashboard → gear icon → clear **Theme** and **SASS** caches

> Prefer the Extension Installer? Rename to `install.xml`, zip it, upload via **Extensions → Extension Installer**.

### Required one-time database step

Stock `oc_cart` carries only `PRIMARY (cart_id)` and `KEY cart_id (api_id, customer_id, session_id, product_id, recurring_id)`. `date_added` does not follow a usable prefix of that key — and since nearly every guest row is `api_id = 0, customer_id = 0`, matching on that prefix alone still spans the whole table. Add an index that covers the GC predicate:

```sql
ALTER TABLE oc_cart ADD INDEX idx_cart_gc (api_id, customer_id, date_added);
```

This gives the guest GC query two equality matches followed by a range — the optimal shape. If you also run a large number of API carts, add `ALTER TABLE oc_cart ADD INDEX idx_date_added (date_added);` for the `api_id > 0` variant.

Verify the plan on your own data rather than trusting the recommendation:

```sql
EXPLAIN DELETE FROM oc_cart
WHERE api_id = 0 AND customer_id = 0
  AND date_added < DATE_SUB(NOW(), INTERVAL 720 HOUR)
LIMIT 500;
```

### Optional: isolation level

If you control the server, `READ COMMITTED` removes gap locking entirely and is the single most effective change against this class of deadlock:

```ini
transaction_isolation = READ-COMMITTED
binlog_format         = ROW
```

---

## Verification

Confirm exactly four call sites were injected (a plain `grep set_cart_cookie` returns **five** lines — the method declaration counts):

```bash
grep -c '\$this->set_cart_cookie();' \
  /path/to/storage/modification/system/library/cart/cart.php
```

Expected output: `4`

---

## Recommended Companion Change

Deadlocks elsewhere in the codebase are best handled with a retry in `system/library/db/mysqli.php`:

```php
// In query() — catch error 1213 and retry
$retries = 3;
while ($retries--) {
    $result = mysqli_query($this->connection, $sql);
    if ($result !== false || mysqli_errno($this->connection) !== 1213) break;
    usleep(100000); // 100ms backoff
}
```

This OCMOD reduces how often 1213 occurs; the retry is what keeps it from surfacing to the customer.

---

## Known Limitations

- The aggregation path assumes at most one row per `product_id` + `recurring_id` + `option` within a session, which is what stock `add()` guarantees. If a third-party extension has inserted duplicates inside a single session, the fold updates the destination once and then deletes all matching source rows, losing the surplus quantity.
- The collision path issues `START TRANSACTION` / `COMMIT` through the DB adapter. If a deadlock is thrown mid-transaction the connection tears down and the whole merge rolls back, which is the safe outcome — but it depends on the retry wrapper below to avoid surfacing as a fatal.
- The private-property guard does not coordinate parallel HTTP requests. Two simultaneous requests can both attempt the migration; the second simply matches zero rows.
- Deadlocks are made rare, not impossible. See the scope note at the top.

---

## Changelog

| Version | Changes |
|---|---|
| **1.9** | **Fix:** merge now aggregates quantities when the destination session already holds the same `product_id` + `recurring_id` + `option`, instead of producing duplicate cart lines — gated behind a cheap collision probe so the common path stays a single `UPDATE`. **Fix:** `headers_sent()` promoted to a precondition of the whole migration; previously rows could be moved while the `cart_hash` cookie could not be updated, orphaning the cart once the session expired. `write_cart_cookie()` now returns `bool`. Corrected the TTL description — the window extends at most once per 24h on cart modification, not on every interaction. |
| **1.8** | **Fix:** active carts no longer expire 30 days after the first add — `date_added` is now refreshed on cart activity and on session migration (TTL is sliding, throttled to once per 24h). **Fix:** session ID is validated against OpenCart's `/^[a-zA-Z0-9,\-]{22,52}$/` instead of having characters stripped, which corrupted IDs containing `,` or `-`. **Fix:** replaced the session-stored `cart_hash_updated` flag with a private property; the previous one persisted beyond the request despite being documented as request-scoped. Added PHP 7.0–7.2 `setcookie` fallback and a `headers_sent()` guard. Corrected the index recommendation (`idx_cart_gc` composite) and the install-verification command (4 calls, not 5 matches). Documentation claims about eliminating deadlocks and avoiding full scans toned down to what the code actually delivers. |
| **1.7** | Split GC into two DELETEs with `LIMIT 200/500`; PHP 7.3+ array `setcookie` with `SameSite=Lax`; `mt_rand` for the GC trigger |
| **1.6** | `&amp;` XML escape fix; exact full signatures for hook insertion; `$_COOKIE` updated immediately after migration; guest-only guard on `set_cart_cookie()` |
| **1.5** | DRY refactor — `get_cookie_params()` helper; cookie path/domain fallback |
| **1.4** | `get_cookie_params()` introduced |
| **1.3** | Proxy-aware HTTPS detection via `HTTP_X_FORWARDED_PROTO` |
| **1.2** | Added `update()` hook; `trim="true"` on search operations |
| **1.1** | Cookie TTL extended to 30 days; GC interval extended to 720 hours |
| **1.0** | Initial release |
