# Documentation

> PHPDoc standards for WordPress/WooCommerce code.

---

## Method Documentation

```php
/**
 * Short description.
 *
 * Longer description if needed.
 *
 * @since 1.0.0
 *
 * @param int    $product_id Product ID.
 * @param string $context    Optional. Context for the operation. Default 'view'.
 * @return bool True on success, false on failure.
 *
 * @throws \InvalidArgumentException If product ID is invalid.
 */
public function update_stock( int $product_id, string $context = 'view' ): bool {
	// implementation
}
```

---

## Required Tags

| Tag | When to Use |
|-----|-------------|
| `@since` | Every public method (required) |
| `@param` | Every parameter with type and description |
| `@return` | Every method with return type |
| `@throws` | If method can throw exceptions |
| `@hooked` | For hook callbacks: `@hooked woocommerce_init` |
| `@requires` | For PHP 8.1+ features: `@requires PHP 8.1` |

---

## Class Documentation

```php
/**
 * Stock management service.
 *
 * Handles stock queries, updates, and synchronization
 * with external inventory systems.
 *
 * @since 1.0.0
 * @requires PHP 8.1
 */
final class Stock_Service {
```

---

## Hook Documentation

```php
/**
 * Fires after stock is updated.
 *
 * @since 1.0.0
 *
 * @param int $product_id Product ID.
 * @param int $quantity   New stock quantity.
 */
do_action( 'nvm/inventory/stock_updated', $product_id, $quantity );
```

---

## Complex Types in PHPDoc

When PHP's type system can't express the type:

```php
/**
 * Get stock data for multiple products.
 *
 * @param int[] $product_ids Product IDs.
 * @return array<int, array{quantity: int, status: string, sku: string}>
 */
public function get_bulk_stock( array $product_ids ): array {
```
