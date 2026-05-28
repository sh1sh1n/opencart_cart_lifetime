# OpenCart Persistent Cart & InnoDB Deadlock Fix

An OCMOD modification for OpenCart 3.x that eliminates database mutual locking issues (MySQL/MariaDB Error: 1213 / Deadlock found when trying to get lock) in the oc_cart table, while ensuring reliable long-term cart storage (persistent cart) for guests without causing spike loads on MySQL.

## The Pain

In standard OpenCart installations and conventional persistent cart extensions, the constructor of the Cart class (`system/library/cart/cart.php`) executes heavy SQL statements `sequentially—specifically`, removing old guest carts (DELETE) and updating session IDs (UPDATE)—on every single page view by every user.

As traffic scales or during high-frequency processes like automated ERP/CRM product syncs, InnoDB triggers row-level locking. This initiates a cyclical lock dependency (Deadlock):

*    **User A** locks their specific row via `UPDATE` and simultaneously fires a `DELETE` for outdated records.
*    **User B** executes an `UPDATE` at that exact millisecond but gets trapped under the wide row-range scanning initiated by User A's `DELETE`.
*    MySQL detects the unresolvable deadlock loop, instantly terminates one of the competing transactions, and throws a 500 error or a white screen to the user.

The associated log output typically looks like this:

```tb
PHP Fatal error: Uncaught Exception: Error: Deadlock found when trying to get lock; try restarting transaction
UPDATE oc_cart SET session_id = '...' WHERE api_id = '0' AND customer_id = '398'
in /system/library/db/mysqli.php
```

This architectural bottleneck is especially problematic on shared hosting environments where developers lack administrative privileges to tweak global database variables like the `TRANSACTION ISOLATION LEVEL`.

## The Solution

This OCMOD extension fundamentally alters how the `oc_cart` table handles traffic, removing redundant database query spam and preventing self-induced DDoS patterns:

1.    Optimized Cleanup (`DELETE`): The demanding query that purges guest carts older than 720 hours is no longer executed on every single hit. Instead, it is wrapped in a randomization function, running with a low 0.1% probability (roughly once every 1,000 page views). This frequency is perfectly sufficient to keep the table clean.

2.    Segmented Guest / Customer Logic: The persistent cart cookie tracking mechanism (`cart_hash`) is isolated to run exclusively for unauthenticated guests. The moment a user logs in, guest cookie evaluations bypass completely to prevent row locking overlapping.

3.    Concurrency Protection (Session Marker): Session markers named `cart_hash_updated are applied. Even if a visitor opens 20 browser tabs simultaneously, the database `UPDATE executes strictly on the first request, while the remaining 19 tabs bypass query execution entirely.

## Technical Requirements

*    CMS: OpenCart 3.0.x (including popular distributions like ocStore)
*    Database Engine: MySQL / MariaDB using InnoDB tables
*    PHP Database Driver: MySQLi

## Installation

   1. Download the `opencart_cart_lifetime.ocmod.xml` file from this repository.

   2. Navigate to your OpenCart administrative panel: Extensions -> Extension Installer.

   3. Upload the xml file.

   4. Go to Extensions -> Modifications.

   5. Click the blue Refresh button in the top right corner to rebuild the modification cache.

   6. On the main Dashboard, click the blue gear icon in the top right corner and clear both Theme and SASS caches.

## Performance Impact

*    0% deadlocks originating from the `oc_cart` table in your server error logs.

*    Significant reduction in CPU and I/O overhead on the database server under peak traffic.

*    Seamless user experience: Guests retain items in their shopping carts for up to 30 days even if their server session expires.

## Additional Recommendation

If your database environment frequently suffers from deadlocks across other core functions, it is highly recommended to integrate a query-retry mechanism directly inside your database driver. You can achieve this by modifying the `query()` method in `system/library/db/mysqli.php` to catch error code `1213`, apply a micro-delay using `usleep(100000);`, and retry execution up to 3 times. Paired with this modification, it provides total immunity against lock-related crashes.
