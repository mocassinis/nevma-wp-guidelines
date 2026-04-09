# Performance

> Conditional loading, caching, Action Scheduler, query optimization, and WooCommerce-specific performance patterns.

---

## Conditional Loading

Never load assets globally:

```php
public function enqueue_scripts(): void {
    if ( ! is_checkout() ) {
        return;
    }

    // Check if already enqueued (multi-plugin safety).
    if ( wp_script_is( 'nvm-inventory-checkout', 'enqueued' ) ) {
        return;
    }

    wp_enqueue_script(
        'nvm-inventory-checkout',
        Plugin::url( 'assets/js/checkout.js' ),
        [],    // No jQuery dependency for frontend.
        Plugin::VERSION,
        [ 'strategy' => 'defer' ]  // WP 6.3+ script loading strategy.
    );
}
```

---

## Action Scheduler for Heavy Tasks

Any task > 2 seconds must be async:

```php
// Schedule.
as_schedule_single_action(
    time() + 60,
    'nvm/inventory/sync',
    [ 'product_id' => $id ],
    'nvm-inventory'
);

// Handler.
add_action( 'nvm/inventory/sync', [ $this, 'process_sync' ] );
```

### Recurring Tasks

```php
// Schedule recurring task (if not already scheduled).
if ( ! as_has_scheduled_action( 'nvm/inventory/daily_cleanup', [], 'nvm-inventory' ) ) {
    as_schedule_recurring_action(
        time(),
        DAY_IN_SECONDS,
        'nvm/inventory/daily_cleanup',
        [],
        'nvm-inventory'
    );
}
```

### Batch Processing with Action Scheduler

```php
/**
 * Process large datasets in batches via Action Scheduler.
 * Each batch schedules the next one — no memory issues.
 */
public function schedule_batch_sync(): void {
    as_schedule_single_action(
        time(),
        'nvm/inventory/batch_sync',
        [ 'page' => 1 ],
        'nvm-inventory'
    );
}

public function process_batch( int $page ): void {
    $per_page = 50;

    $product_ids = wc_get_products( [
        'limit'  => $per_page,
        'page'   => $page,
        'return' => 'ids',
        'status' => 'publish',
    ] );

    if ( empty( $product_ids ) ) {
        wc_get_logger()->info( 'Batch sync complete.', [ 'source' => Plugin::SLUG ] );
        return;
    }

    foreach ( $product_ids as $product_id ) {
        $this->sync_single_product( $product_id );
    }

    // Schedule next batch.
    as_schedule_single_action(
        time(),
        'nvm/inventory/batch_sync',
        [ 'page' => $page + 1 ],
        'nvm-inventory'
    );
}
```

---

## Transient Caching (Persistent)

For data that should survive across requests:

```php
$cache_key = Plugin::PREFIX . 'expensive_data_' . $product_id;
$data      = get_transient( $cache_key );

if ( false === $data ) {
    $data = $this->expensive_operation( $product_id );
    set_transient( $cache_key, $data, HOUR_IN_SECONDS );
}

return $data;
```

---

## Object Cache (Per-Request)

For data reused multiple times within the same request (avoids repeated `get_post_meta` calls):

```php
$cache_key = Plugin::PREFIX . 'stock_' . $product_id;
$found     = false;
$stock     = wp_cache_get( $cache_key, Plugin::SLUG, false, $found );

if ( ! $found ) {
    $product = wc_get_product( $product_id );
    $stock   = $product?->get_stock_quantity() ?? 0;
    wp_cache_set( $cache_key, $stock, Plugin::SLUG, 300 );
}

return $stock;
```

---

## Cache Invalidation

Always invalidate when data changes:

```php
public function update_stock( int $product_id, int $quantity ): void {
    $product = wc_get_product( $product_id );
    $product?->set_stock_quantity( $quantity );
    $product?->save();

    // Clear all caches for this product.
    delete_transient( Plugin::PREFIX . 'expensive_data_' . $product_id );
    wp_cache_delete( Plugin::PREFIX . 'stock_' . $product_id, Plugin::SLUG );

    /**
     * Fires after stock is updated — let other code clear its caches too.
     *
     * @since 1.0.0
     *
     * @param int $product_id Product ID.
     * @param int $quantity   New stock quantity.
     */
    do_action( 'nvm/inventory/stock_updated', $product_id, $quantity );
}
```

---

## Query Performance Rules

- **Always set `limit`** on `wc_get_orders()` and `wc_get_products()`. Never unbounded.
- **Use `'return' => 'ids'`** when you only need IDs, not full objects.
- **Batch large operations**: Process in chunks of 50-100, not all at once.
- **Avoid `get_posts` in loops**: Prefetch with a single query when possible.

### Example: Batched Processing

```php
public function process_all_products(): void {
    $page = 1;
    $per_page = 50;

    do {
        $product_ids = wc_get_products( [
            'limit'   => $per_page,
            'page'    => $page,
            'return'  => 'ids',
            'status'  => 'publish',
        ] );

        if ( empty( $product_ids ) ) {
            break;
        }

        foreach ( $product_ids as $product_id ) {
            $this->process_single_product( $product_id );
        }

        $page++;

        // Prevent memory issues on large catalogs.
        if ( function_exists( 'wp_cache_flush' ) ) {
            wp_cache_flush();
        }

    } while ( count( $product_ids ) === $per_page );
}
```

---

## WooCommerce-Specific Performance

### Avoid N+1 Queries

```php
// BAD — N+1: one query per product inside the loop.
$order = wc_get_order( $order_id );
foreach ( $order->get_items() as $item ) {
    $product = $item->get_product(); // Triggers a query each time.
    $sku     = $product?->get_sku();
}

// GOOD — Prefetch all product IDs, then batch-load.
$order       = wc_get_order( $order_id );
$product_ids = [];

foreach ( $order->get_items() as $item ) {
    $product_ids[] = $item->get_product_id();
}

// Prime the WooCommerce object cache.
_prime_post_caches( $product_ids );
wc_get_products( [ 'include' => $product_ids, 'limit' => count( $product_ids ) ] );

// Now these calls hit cache, not the database.
foreach ( $order->get_items() as $item ) {
    $product = $item->get_product();
    $sku     = $product?->get_sku();
}
```

### Product Lookup Tables

WooCommerce maintains lookup tables for faster queries. Use them:

```php
// Use lookup table for price-range queries (faster than meta queries).
$products = wc_get_products( [
    'limit'     => 50,
    'min_price' => 10,
    'max_price' => 100,
    'return'    => 'ids',
] );
```

### Avoid Loading Full Objects When Not Needed

```php
// BAD — loads full WC_Product for each, heavy on memory.
$products = wc_get_products( [ 'limit' => 500 ] );
$names    = array_map( fn( $p ) => $p->get_name(), $products );

// GOOD — use IDs + direct meta access for simple reads.
$product_ids = wc_get_products( [ 'limit' => 500, 'return' => 'ids' ] );
$names       = array_map( fn( int $id ): string => get_the_title( $id ), $product_ids );
```

### Optimize WooCommerce Admin

```php
// Disable WooCommerce status widget on dashboard if not needed.
add_action( 'wp_dashboard_setup', static function(): void {
    if ( ! current_user_can( 'manage_woocommerce' ) ) {
        remove_meta_box( 'woocommerce_dashboard_status', 'dashboard', 'normal' );
    }
} );
```

### Cart Fragment Optimization

```php
// Disable cart fragments AJAX on pages that don't need it.
add_action( 'wp_enqueue_scripts', static function(): void {
    if ( is_front_page() || is_archive() ) {
        wp_dequeue_script( 'wc-cart-fragments' );
    }
}, 100 );
```

---

## Database Query Optimization

### Use `$wpdb->prepare()` with Proper Indexing

```php
// Ensure your custom tables have appropriate indexes.
// Always use prepare() for custom queries.
global $wpdb;
$table = $wpdb->prefix . 'nvm_sync_log';

$results = $wpdb->get_results(
    $wpdb->prepare(
        "SELECT product_id, sync_status, synced_at
         FROM {$table}
         WHERE sync_status = %s
         AND synced_at > %s
         ORDER BY synced_at DESC
         LIMIT %d",
        'failed',
        gmdate( 'Y-m-d H:i:s', strtotime( '-24 hours' ) ),
        100
    )
);
```

### Custom Table Indexes

```php
// When creating custom tables, always add indexes for columns you query.
public function create_tables(): void {
    global $wpdb;
    $table   = $wpdb->prefix . 'nvm_sync_log';
    $charset = $wpdb->get_charset_collate();

    $sql = "CREATE TABLE {$table} (
        id bigint(20) unsigned NOT NULL AUTO_INCREMENT,
        product_id bigint(20) unsigned NOT NULL,
        sync_status varchar(20) NOT NULL DEFAULT 'pending',
        synced_at datetime NOT NULL DEFAULT CURRENT_TIMESTAMP,
        error_message text,
        PRIMARY KEY (id),
        KEY product_id (product_id),
        KEY sync_status_date (sync_status, synced_at)
    ) {$charset};";

    require_once ABSPATH . 'wp-admin/includes/upgrade.php';
    dbDelta( $sql );
}
```

---

## Autoloaded Options

Keep autoloaded options small. Large serialized arrays in autoloaded options slow every page load:

```php
// BAD — large data in autoloaded option.
update_option( 'nvm_inv_all_sync_results', $huge_array, true );

// GOOD — store only config/settings as autoloaded.
update_option( 'nvm_inv_settings', $small_settings, true );
// Store large data as non-autoloaded.
update_option( 'nvm_inv_sync_results', $huge_array, false );
// Or better: use a custom table for large datasets.
```

---

## Performance Monitoring

### Server-Timing Headers (Debug Only)

```php
add_action( 'send_headers', static function(): void {
    if ( ! defined( 'WP_DEBUG' ) || ! WP_DEBUG ) {
        return;
    }

    $start = microtime( true );
    // ... operation ...
    $duration = ( microtime( true ) - $start ) * 1000;

    header( sprintf( 'Server-Timing: nvm-sync;dur=%.2f;desc="NVM Sync"', $duration ) );
} );
```

### Query Monitor Compatibility

```php
// Add custom timing to Query Monitor.
do_action( 'qm/start', 'nvm-inventory-sync' );
$this->run_sync();
do_action( 'qm/stop', 'nvm-inventory-sync' );
```

---

## Performance Checklist

| Rule | Details |
|------|---------|
| **Always set `limit`** | Every `wc_get_orders()`, `wc_get_products()`, `WP_Query` must have a limit |
| **Use `'return' => 'ids'`** | When you only need IDs, never load full objects |
| **Batch > 50 items** | Process in chunks of 50-100, flush cache between batches |
| **Async > 2 seconds** | Any operation over 2 seconds must use Action Scheduler |
| **Cache expensive calls** | Use transients for cross-request, `wp_cache_*` for per-request |
| **Invalidate on write** | Every data write must clear related caches |
| **Conditional loading** | Assets only on pages that need them, with `defer`/`async` |
| **No N+1 queries** | Prefetch IDs, batch-load objects outside loops |
| **Small autoloaded options** | Config only — large data in non-autoloaded options or custom tables |
| **Index custom tables** | Add indexes for every column used in WHERE/ORDER BY |
