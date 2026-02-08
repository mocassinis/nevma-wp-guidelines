# Performance

> Conditional loading, caching, Action Scheduler, and query optimization.

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
