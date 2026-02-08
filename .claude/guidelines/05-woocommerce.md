# WooCommerce

> CRUD classes, HPOS compatibility, block checkout, and logging.

---

## Use CRUD Classes

Never use direct SQL for products/orders:

```php
$product = wc_get_product( $id );
$product?->set_stock_quantity( 50 );
$product?->save();
```

---

## HPOS Compatibility

Never use `wp_posts`/`wp_postmeta` for orders:

```php
$orders = wc_get_orders( [
	'status' => 'processing',
	'limit'  => 50,       // ALWAYS set a limit.
	'return' => 'ids',    // Use 'ids' when you don't need full objects.
] );
```

### Declare HPOS Compatibility

Already included in Plugin class pattern (see `02-architecture.md`):

```php
public function declare_hpos_compatibility(): void {
	if ( class_exists( \Automattic\WooCommerce\Utilities\FeaturesUtil::class ) ) {
		\Automattic\WooCommerce\Utilities\FeaturesUtil::declare_compatibility(
			'custom_order_tables',
			NVM_INV_FILE,
			true
		);
	}
}
```

---

## Block Checkout

Classic hooks (`woocommerce_after_checkout_form`, etc.) may not work with block checkout. Use the Store API `ExtendSchema` for block checkout extensions:

```php
use Automattic\WooCommerce\StoreApi\StoreApi;
use Automattic\WooCommerce\StoreApi\Schemas\V1\CheckoutSchema;

add_action( 'woocommerce_blocks_loaded', function(): void {
	$extend = StoreApi::container()->get( \Automattic\WooCommerce\StoreApi\Schemas\ExtendSchema::class );

	$extend->register_endpoint_data( [
		'endpoint'        => CheckoutSchema::IDENTIFIER,
		'namespace'       => Plugin::SLUG,
		'data_callback'   => fn() => [ 'custom_field' => '' ],
		'schema_callback' => fn() => [
			'custom_field' => [
				'description' => __( 'Custom checkout field', 'nvm-inventory' ),
				'type'        => 'string',
			],
		],
	] );
} );
```

---

## Logging

Use WooCommerce's built-in logger:

```php
wc_get_logger()->info(
	'Stock updated for product {product_id}: {old} → {new}',
	[
		'source'     => Plugin::SLUG,
		'product_id' => $product_id,
		'old'        => $old_stock,
		'new'        => $new_stock,
	]
);
```

Log levels: `debug`, `info`, `notice`, `warning`, `error`, `critical`, `alert`, `emergency`.

Logs appear in **WooCommerce → Status → Logs**.
