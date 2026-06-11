# OpenCart Persistent Cart & InnoDB Deadlock Fix

**OCMOD v1.6 for OpenCart 3.x** — eliminates MySQL/MariaDB deadlocks (`Error 1213`) in the `oc_cart` table while providing reliable 30-day persistent shopping carts for guest users, without generating spike load on the database.

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

```php
if (rand(1, 1000) === 500) {
    $this->db->query("DELETE FROM oc_cart WHERE ... date_added < DATE_SUB(NOW(), INTERVAL 720 HOUR)");
}
```

### 2. Guest-only session migration
The `cart_hash` cookie logic runs **exclusively for unauthenticated guests**. Logged-in users bypass it entirely, preventing lock overlap on customer rows.

```php
$is_guest = empty($this->session->data['customer_id']);
if ($is_guest && $cart_hash != $current_session && ...) { ... }
```

### 3. Per-request session marker (concurrency guard)
A session flag `cart_hash_updated` ensures the session migration `UPDATE` fires only once per request lifecycle — even if 20 browser tabs open simultaneously.

```php
if (!isset($this->session->data['cart_hash_updated'])) {
    // run UPDATE, then set the flag
    $this->session->data['cart_hash_updated'] = true;
}
```

### 4. Cookie refresh on all cart mutations
`set_cart_cookie()` is called after `add()`, `update()`, `remove()`, and `clear()` — ensuring the 30-day TTL resets on every cart interaction.

---

## Technical Features

| Feature | Details |
|---|---|
| Cart lifetime | 30 days (`86400 * 30`) |
| GC probability | 0.1% per request |
| HTTPS detection | Proxy-aware (`HTTPS` + `X-Forwarded-Proto`) |
| Cookie path/domain | Inherited from `session.cookie_path` / `session.cookie_domain` with `/` fallback |
| Hooks | `add()`, `update()`, `remove()`, `clear()` |
| Guest guard | `empty($this->session->data['customer_id'])` |
| DRY config | Single `get_cookie_params()` method for all `setcookie()` calls |
| XML validity | All special characters properly escaped (`&amp;`) |

---

## Requirements

- OpenCart 3.0.x (including ocStore and similar distributions)
- MySQL / MariaDB with InnoDB tables
- PHP MySQLi driver

---

## Installation

1. Download `opencart_cart_lifetime.ocmod.xml`
2. Upload the file via FTP to the `/system/` directory of your OpenCart installation
3. In your OpenCart admin panel go to **Extensions → Modifications**
4. Click the **Refresh** button (top right) to rebuild the modification cache
5. On the Dashboard, click the gear icon and clear both **Theme** and **SASS** caches

> **Prefer the Extension Installer?** Rename the file to `install.xml`, pack it into a `.zip` archive, and upload it via **Extensions → Extension Installer**.

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
| **1.6** | `regex="true"` for reliable hook insertion after `{`; `&amp;` XML escape fix (resolves admin 500 on modification refresh); `$_COOKIE` superglobal updated immediately after session migration |
| **1.5** | DRY refactor — `get_cookie_params()` helper; session cookie path/domain fallback logic |
| **1.4** | `get_cookie_params()` introduced to eliminate duplicate cookie config |
| **1.3** | Proxy-aware HTTPS detection via `HTTP_X_FORWARDED_PROTO` |
| **1.2** | Added `update()` hook; `trim="true"` on search operations |
| **1.1** | Cookie TTL extended to 30 days; GC interval extended to 720 hours |
| **1.0** | Initial release |
