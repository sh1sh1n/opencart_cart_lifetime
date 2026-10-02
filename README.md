# OpenCart Persistent Guest Cart & Cart GC Contention Fix

**OCMOD v2.0 for OpenCart 3.0.x** — 30-day persistent cart for guests, and a throttled cart garbage collector that no longer runs a table-wide `DELETE` on every page view.

---

## The problem

Stock `Cart::__construct()` (`system/library/cart/cart.php`) runs this on **every request**:

```sql
DELETE FROM oc_cart
WHERE (api_id > '0' OR customer_id = '0')
  AND date_added < DATE_SUB(NOW(), INTERVAL 1 HOUR)
```

The `OR` defeats the indexes, so under REPEATABLE READ the statement takes next-key locks across the table and regularly deadlocks with concurrent cart inserts (`Error 1213`). It also gives guests a one-hour cart.

---

## Contract

| | |
|---|---|
| Guest cart lifetime | 30 days from the last cart activity |
| Activity that extends it | `add()`, `update()`, `remove()`, and returning with the cookie after the session is gone. Ordinary page views inside a live session do not extend it. |
| Pointer | HttpOnly cookie `cart_lifetime` = cart key (32 hex chars). Not a session ID. `Secure` follows OpenCart's own HTTPS detection, `SameSite=Lax` on PHP 7.3+. |
| Cookie vs rows | Both are extended on the same events; the cookie always outlives the rows it points to. |
| Visitors without a cart | No cookie is set. |
| Logged-in customers | Stock behaviour: the cart follows `customer_id`; the guest cart is folded in on login by stock code. No cookie writes. |
| API sessions (admin order editor) | Untouched: stock key, no cookie, stock 1-hour retention. |
| GC | ~0.1% of requests, two bounded `DELETE`s: API rows older than 1 hour (`LIMIT 200`), guest rows older than 30 days (`LIMIT 500`). Customer carts are never deleted (stock). |
| Failures | Housekeeping (GC, lifetime refresh, legacy migration) never fails the page. Errors are logged as `cart_lifetime: <operation> failed (<class>, code <errno>)` — no SQL, cookie values, cart keys or session IDs. A real database outage still surfaces through the stock queries that follow. |

### How the cart is found

Every stock cart query uses `$this->cart_lifetime_key()` where it used the session ID. For a guest it resolves, in order:

1. a valid `cart_lifetime` cookie;
2. a v1.x `cart_hash` cookie (a raw session ID) — its rows are moved once to a key derived from it, then the old cookie is removed;
3. the key remembered in the session;
4. a key derived from the current session ID — derived, not random, so parallel first requests of one session agree on it. Rows written under the session ID before installation are moved to it.

Restoring a cart moves no rows: any number of parallel requests that arrive with the same cookie and different new sessions all resolve to the same key. Migrating a legacy cookie is a single `UPDATE` whose target is derived from the source, so concurrent requests move the rows to the same place and the later ones match nothing. Rows are moved verbatim and never deleted, so no quantity can be lost; the module issues no transactions.

If the legacy migration fails (deadlock, lock wait), the old cookie is kept, the key is not recorded in the session and no new cookie is written, so the next request retries.

---

## Requirements

- OpenCart 3.0.x — anchors verified on 3.0.3.8, 3.0.4.1 and 3.0.5.0
- MySQL / MariaDB, InnoDB
- PHP 7.0+ (tested on 7.4, 8.1, 8.3, 8.5)

---

## Installation

1. **Upgrading from 1.x:** delete the old *Cart Cookie Hash* modification first (same code, the installer refuses duplicates).
2. Upload `cart_lifetime.ocmod.xml` to `/system/`, or rename it to `install.xml`, zip it and use **Extensions → Extension Installer**.
3. **Extensions → Modifications → Refresh**.
4. Check `storage/logs/ocmod.log`: the `system/library/cart/cart.php` section must show no `NOT FOUND`. A skipped PayPal operation is expected where that query does not exist.

Existing `cart_hash` cookies from 1.x are migrated on the visitor's next request. Guest carts sitting on a live session ID at install time are picked up on that session's next request.

### Recommended index

Stock `oc_cart` has no index usable by the GC predicate. Add one and confirm with `EXPLAIN` on your data:

```sql
ALTER TABLE oc_cart ADD INDEX idx_cart_gc (api_id, customer_id, date_added);

EXPLAIN DELETE FROM oc_cart
WHERE api_id = 0 AND customer_id = 0
  AND date_added < DATE_SUB(NOW(), INTERVAL 30 DAY)
LIMIT 500;
```

`LIMIT` bounds the rows deleted, not the rows scanned; without the index a GC run can still scan a large range.

### Optional: GC from cron

Random GC can fall behind on busy stores and run rarely on quiet ones. If you have cron, set `CART_LIFETIME_GC_DIVISOR` to `0` in the XML (constants are at the top of the first operation), refresh modifications, and schedule:

```sql
DELETE FROM oc_cart WHERE api_id > 0 AND date_added < DATE_SUB(NOW(), INTERVAL 1 HOUR) LIMIT 1000;
DELETE FROM oc_cart WHERE api_id = 0 AND customer_id = 0 AND date_added < DATE_SUB(NOW(), INTERVAL 30 DAY) LIMIT 1000;
```

Repeat each statement until it affects 0 rows. Lifetimes and the refresh interval are constants in the same place.

---

## Verification

Against the generated file (paths relative to the store root):

```bash
F=storage/modification/system/library/cart/cart.php
grep -c 'cart_lifetime_key()' "$F"        # 13 on stock 3.0.3.8-3.0.5.0
grep -c 'session->getId()' "$F"           # 1 (the module's own resolver)
grep -c 'cart_lifetime_touch();' "$F"     # 4
grep -c 'cart_lifetime_init();' "$F"      # 1
```

A `session->getId()` count above 1 means a fork or another modification builds cart queries differently; those queries would still use the session ID and must be adapted before going live.

### Why there is no `error="abort"`

OCMOD keeps the operations applied before a missing anchor. The operations are ordered so that every such prefix still yields a working class (helpers → query key → GC → lifetime hooks); a missing anchor degrades the feature but cannot cause `Call to undefined method`. `error="abort"` would roll back this modification, but in OpenCart 3.0.x it does so with `break 5`, which also silently skips every modification loaded after this one.

---

## Deadlocks and retries

The module runs single statements in autocommit mode and treats housekeeping failures as non-fatal, so it needs no retry wrapper.

Do **not** add a transparent per-statement retry to `system/library/db/mysqli.php`. On a deadlock InnoDB rolls back the **whole** transaction; replaying only the failed statement inside code that uses `START TRANSACTION` silently drops the statements before it (for example, re-running a `DELETE` after the `UPDATE` that preceded it was rolled back). Retry the whole operation in the code that owns the transaction, and remember that with `MYSQLI_REPORT_STRICT` (the PHP 8.1+ default) failures arrive as `mysqli_sql_exception`, not as a `false` return.

`READ COMMITTED` reduces gap locking but does not remove it (foreign-key and duplicate-key checks still take gap locks). It is a server-wide change; prefer setting it per session or test every application sharing the server.

---

## Compatibility notes

- Guest cart rows are keyed by the cart key, not the session ID. Third-party code that reads or writes `oc_cart` directly with `session_id = <session ID>` (abandoned-cart tools, custom checkouts) will not see guest carts. Find candidates with `grep -rn "cart WHERE" catalog/ | grep session` and adapt them with a guarded call, so they keep working if this modification is removed or not applied:

  ```php
  $this->db->escape(method_exists($this->cart, 'getLifetimeKey') ? $this->cart->getLifetimeKey() : $this->session->getId())
  ```

  An unguarded `$this->cart->getLifetimeKey()` fails with `Call to undefined method` as soon as the modification is gone.
- The stock PayPal models (`payment/paypal`, `module/paypal_smart_button`) do this in `hasProductInCart()`; the modification patches them with the guarded call above, otherwise PayPal "buy now" would add the product a second time. The patch is skipped where the query does not exist (e.g. `payment/paypal` in 3.0.3.8).
- The guard checks that `getLifetimeKey()` exists (operation 1), not that the cart queries were switched to the key (operation 2). If only operation 1 applied — a fork whose `cart.php` contains none of the stock query expressions — the cart would still use the session ID while patched callers use the key, and PayPal "buy now" could add one extra unit. Nothing fails and no rows are lost; the verification counts above catch this state.
- Stock `add()` is a non-atomic `SELECT` + `INSERT`, and `oc_cart` has no unique key, so concurrent adds of the same product can create duplicate rows. That is stock behaviour; this module never folds or deletes rows when moving them, so quantities are preserved.
- A cart key is a bearer token for the guest cart only: it grants no access to the session or to a customer account. On a shared computer the guest cart persists for the next guest, as with any persistent cart.
- ocStore and other forks are not verified; use the verification step above.

---

## Changelog

| Version | Changes |
|---|---|
| **2.0** | Guest carts keyed by a stable cart key instead of the session ID. **Fixes:** parallel restores with different new sessions no longer leave the cookie pointing at an empty cart (rows are never moved on restore); merge transaction, collision probe and fold/delete aggregation removed, so no quantity can be lost on duplicates or deadlocks; cookie expiry is extended together with `date_added`; log lines carry only operation and error number; GC, lifetime refresh and migration errors no longer abort the request; operations ordered so any partial OCMOD application leaves a working class; API carts keep stock 1-hour retention and never get the guest cookie; array-valued cookies no longer raise warnings; stock PayPal `hasProductInCart()` follows the cart key. 1.x `cart_hash` cookies are migrated automatically. README: removed the unsafe per-statement retry recommendation. |
| 1.11 | Pinned `cart_hash` after a failed merge for the rest of the request. |
| 1.10 | Real `setcookie()` result; cookie advanced only after a confirmed merge; explicit `ROLLBACK`. |
| 1.9 | Quantity aggregation on merge; `headers_sent()` as a merge precondition. |
| 1.8 | Sliding TTL; session ID validated instead of stripped; request-scoped guard; PHP 7.0–7.2 cookie fallback. |
| 1.7 | GC split into two bounded `DELETE`s; `SameSite=Lax`. |
| 1.1–1.6 | 30-day cookie and GC interval, `update()` hook, proxy-aware HTTPS, cookie helper, XML fixes. |
| 1.0 | Initial release. |
