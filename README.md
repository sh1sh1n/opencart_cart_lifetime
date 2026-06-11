# OpenCart Persistent Cart & InnoDB Deadlock Fix

**OCMOD v1.7 for OpenCart 3.x** — eliminates MySQL/MariaDB deadlocks (`Error 1213`) in the `oc_cart` table while providing reliable 30-day persistent shopping carts for guest users, without generating spike load on the database.

---

## The Problem

In stock OpenCart and most conventional persistent cart extensions, the `Cart` class constructor (`system/library/cart/cart.php`) runs two heavy SQL statements **on every single page view**:

- A `DELETE` to purge old guest carts
- An `UPDATE` to migrate session IDs for returning guests

Under high traffic or during automated ERP/CRM product sync operations, InnoDB's row-level locking turns these into a deadlock cycle:

- **User A** locks a row via `UPDATE` and simultaneously triggers a wide-range `DELETE`
- **User B** fires an `UPDATE` at the same millisecond and blocks behind User A's `DELETE`
- MySQL detects the unresolvable loop, kills one transaction, and the user gets a 500 error

Typical log output:

```
PHP Fatal error: Uncaught Exception: Error: Deadlock found when trying to get lock;
try restarting transaction
UPDATE oc_cart SET session_id = '...' WHERE api_id = '0' AND customer_id = '398'
in /system/library/db/mysqli.php
```

This is especially painful on shared hosting where you cannot adjust `TRANSACTION ISOLATION LEVEL` globally.

---

## The Solution

This OCMOD restructures `cart.php` in four ways:

### 1. Probabilistic garbage collection
The `DELETE` for expired carts no longer runs on every request. It fires with a **0.1% probability** (1 in 1000 page views) — more than sufficient to keep the table clean, while eliminating the constant lock contention.

### 2. Split DELETEs with LIMIT to prevent gap locking
Instead of one wide-range `DELETE` with an `OR` condition (which forces a full table scan and acquires InnoDB gap locks across all visited rows), two narrow queries are used — each targeting a single prefix of the existing composite index:

```php
if (mt_rand(1, 1000) === 500) {
    $this->db->query("DELETE FROM oc_cart WHERE api_id > '0'
        AND date_added < DATE_SUB(NOW(), INTERVAL 720 HOUR) LIMIT 200");
    $this->db->query("DELETE FROM oc_cart WHERE api_id = '0' AND customer_id = '0'
        AND date_added < DATE_SUB(NOW(), INTERVAL 720 HOUR) LIMIT 500");
}
```

`LIMIT` caps the transaction size, keeping lock duration short. Residual rows are cleaned on subsequent GC triggers.

### 3. Guest-only session migration
The `cart_hash` cookie logic runs **exclusively for unauthenticated guests**. Logged-in users bypass it entirely, preventing lock overlap on customer rows.

```php
$is_guest = !$this->customer->isLogged();
if ($is_guest && $cart_hash != $current_session && ...) { ... }
```

### 4. Per-request session marker (concurrency guard)
A session flag `cart_hash_updated` ensures the session migration `UPDATE` fires only once per request lifecycle — even if multiple browser tabs open simultaneously.

```php
if (!isset($this->session->data['cart_hash_updated'])) {
    // run UPDATE, then set the flag
    $this->session->data['cart_hash_updated'] = true;
}
```

### 5. Modern cookie security (PHP 7.3+ array syntax)
All `setcookie()` calls use the array options form with explicit `SameSite=Lax` and `HttpOnly` flags:

```php
setcookie('cart_hash', $current_session, [
    'expires'  => time() + 86400 * 30,
    'path'     => $p['path'],
    'domain'   => $p['domain'],
    'secure'   => $p['secure'],
    'httponly' => true,
    'samesite' => 'Lax'
]);
```

### 6. Cookie refresh on all cart mutations
`set_cart_cookie()` is called after `add()`, `update()`, `remove()`, and `clear()` — ensuring the 30-day TTL resets on every cart interaction.

---

## Technical Features

| Feature | Details |
|---|---|
| Cart lifetime | 30 days (`86400 * 30`) |
| GC probability | 0.1% per request (`mt_rand`) |
| GC query style | Two split DELETEs with LIMIT — no `OR`, no full scan |
| HTTPS detection | Proxy-aware (`HTTPS` + `X-Forwarded-Proto`) |
| Cookie path/domain | Inherited from `session.cookie_path` / `session.cookie_domain` with `/` fallback |
| Cookie flags | `HttpOnly`, `SameSite=Lax`, `Secure` (auto-detected) |
| Hooks | `add()`, `update()`, `remove()`, `clear()` |
| Guest guard | `!$this->customer->isLogged()` |
| DRY config | Single `get_cookie_params()` method for all `setcookie()` calls |
| PHP compatibility | 7.3+ (array-form `setcookie`) |
| XML validity | All special characters properly escaped (`&amp;`) |

---

## Requirements

- OpenCart 3.0.x (including ocStore and similar distributions)
- MySQL / MariaDB with InnoDB tables
- PHP 7.3+ with MySQLi driver

---

## Installation

1. Download `opencart_cart_lifetime.ocmod.xml`
2. Upload the file via FTP to the `/system/` directory of your OpenCart installation
3. In your OpenCart admin panel go to **Extensions → Modifications**
4. Click the **Refresh** button (top right) to rebuild the modification cache
5. On the Dashboard, click the gear icon and clear both **Theme** and **SASS** caches

> **Prefer the Extension Installer?** Rename the file to `install.xml`, pack it into a `.zip` archive, and upload it via **Extensions → Extension Installer**.

### One-time database step (required)

Add an index on `date_added` to ensure GC queries use an index instead of scanning the whole table:

```sql
ALTER TABLE oc_cart ADD INDEX idx_date_added (date_added);
```

Run this once per database where the OCMOD is installed. Without it the `LIMIT` still caps lock duration, but the query will be slower on large tables.

---

## Verification

After refreshing modifications, confirm all four hooks were inserted correctly:

```bash
grep -n "set_cart_cookie" /path/to/storage/modification/system/library/cart/cart.php
```

Expected output — exactly **4 lines**, one inside each of `add()`, `update()`, `remove()`, `clear()`.

---

## Additional Recommendation

If your environment has deadlocks in other parts of the codebase, add a retry mechanism to `system/library/db/mysqli.php`:

```php
// In the query() method — catch error 1213 and retry up to 3 times
$retries = 3;
while ($retries--) {
    $result = mysqli_query($this->connection, $sql);
    if ($result !== false || mysqli_errno($this->connection) !== 1213) break;
    usleep(100000); // 100ms backoff
}
```

Combined with this OCMOD, it provides complete protection against lock-related crashes across the entire application.

---

## Changelog

| Version | Changes |
|---|---|
| **1.7** | Split GC into two separate DELETEs (eliminates `OR` full-scan); added `LIMIT 200/500` per query to cap transaction size and prevent InnoDB gap locking; switched all `setcookie()` calls to PHP 7.3+ array syntax with explicit `SameSite=Lax` and `HttpOnly`; `mt_rand` instead of `rand` for GC trigger |
| **1.6** | `&amp;` XML escape fix (resolves admin 500 on modification refresh); exact full signatures with `{` for hook insertion — no `regex`, no `trim` on hooks; `$_COOKIE` superglobal updated immediately after session migration; guest-only guard on `set_cart_cookie()` |
| **1.5** | DRY refactor — `get_cookie_params()` helper; session cookie path/domain fallback logic |
| **1.4** | `get_cookie_params()` introduced to eliminate duplicate cookie config |
| **1.3** | Proxy-aware HTTPS detection via `HTTP_X_FORWARDED_PROTO` |
| **1.2** | Added `update()` hook; `trim="true"` on search operations |
| **1.1** | Cookie TTL extended to 30 days; GC interval extended to 720 hours |
| **1.0** | Initial release |
