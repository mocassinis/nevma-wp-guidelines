# Advanced Engineering Patterns

> Pro-level strategies for high-performance, secure, and scalable WordPress plugins.

---

## 1. Domain Modeling: DTOs & Value Objects

Avoid passing loose associative arrays between methods. Use **Data Transfer Objects (DTOs)** for structured data and **Value Objects** for domain logic.

### Data Transfer Object (DTO)
Use `readonly` classes (PHP 8.2+) to ensure data integrity.

```php
declare(strict_types=1);

namespace NVM\Inventory\DTO;

/**
 * Structured data for a stock update request.
 */
readonly final class Stock_Update_Request {
	public function __construct(
		public int $product_id,
		public int $new_quantity,
		public string $source = 'manual',
		public ?int $user_id = null,
	) {}

	public static function from_request( \WP_REST_Request $request ): self {
		return new self(
			product_id: (int) $request->get_param( 'id' ),
			new_quantity: (int) $request->get_param( 'quantity' ),
			source: 'rest_api',
			user_id: get_current_user_id()
		);
	}
}
```

### Value Object
Encapsulate logic related to a specific value (e.g., a Price or a SKU).

```php
readonly final class SKU {
	public function __construct(
		private string $value
	) {
		if ( strlen( $value ) < 3 ) {
			throw new \InvalidArgumentException( 'SKU must be at least 3 characters.' );
		}
	}

	public function __toString(): string {
		return strtoupper( $this->value );
	}

	public function equals( SKU $other ): bool {
		return strtoupper( $this->value ) === strtoupper( $other->value );
	}
}
```

---

## 2. Advanced Performance: Fragment Caching

For complex admin dashboards or frontend templates, cache specific HTML fragments rather than the whole page.

```php
public function render_dashboard(): void {
	$cache_key = Plugin::PREFIX . 'dashboard_fragment';
	$output    = get_transient( $cache_key );

	if ( false === $output ) {
		ob_start();
		// ... heavy rendering logic ...
		include Plugin::path( 'templates/admin/dashboard.php' );
		$output = ob_get_clean();

		set_transient( $cache_key, $output, HOUR_IN_SECONDS );
	}

	echo $output; // phpcs:ignore WordPress.Security.EscapeOutput.OutputNotEscaped
}
```

---

## 3. Security: Advanced Validation & RBAC

### JSON Schema Validation (REST API)
Leverage WordPress built-in schema validation for complex payloads.

```php
register_rest_route( self::NAMESPACE, '/batch-update', [
	'methods'             => \WP_REST_Server::EDITABLE,
	'callback'            => [ $this, 'batch_update' ],
	'permission_callback' => [ $this, 'check_write_permission' ],
	'args'                => [
		'updates' => [
			'type'     => 'array',
			'required' => true,
			'items'    => [
				'type'       => 'object',
				'properties' => [
					'product_id' => [ 'type' => 'integer', 'minimum' => 1 ],
					'quantity'   => [ 'type' => 'integer' ],
				],
			],
		],
	],
] );
```

### Granular Capabilities (RBAC)
Instead of checking `manage_woocommerce`, create custom capabilities for your plugin.

```php
// During activation:
$admin = get_role( 'administrator' );
$admin?->add_cap( 'nvm_manage_inventory' );
$admin?->add_cap( 'nvm_view_reports' );

// Usage:
if ( ! current_user_can( 'nvm_manage_inventory' ) ) {
	return new \WP_Error( 'rest_forbidden', __( 'No permission.', 'nvm' ), [ 'status' => 403 ] );
}
```

---

## 4. Scalability: Middleware Pattern

Use a pipeline or middleware-like approach for cross-cutting concerns (logging, auth, profiling).

```php
final class Request_Logger_Middleware {
	public function __invoke( \WP_REST_Request $request, callable $next ) {
		$start = microtime( true );
		$response = $next( $request );
		$duration = microtime( true ) - $start;

		wc_get_logger()->info(
			sprintf( 'REST Request: %s %s - %fs', $request->get_method(), $request->get_route(), $duration ),
			[ 'source' => Plugin::SLUG ]
		);

		return $response;
	}
}
```

---

## 5. Reliability: Detailed Logging (Monolog integration)

Standard WordPress/WooCommerce logs are basic. For "powerful" plugins, use structured logging.

```php
/**
 * Log a domain event.
 *
 * @param string $message The event description.
 * @param array<string, mixed> $context Structured data.
 */
public function log_event( string $message, array $context = [] ): void {
	$logger = wc_get_logger();
	$logger->info(
		$message,
		array_merge( $context, [
			'source'    => Plugin::SLUG,
			'user_id'   => get_current_user_id(),
			'timestamp' => time(),
		] )
	);
}
```

---

## 6. CLI Integration (WP-CLI)

Powerful plugins should be manageable from the command line for automation and large-scale operations.

```php
if ( defined( 'WP_CLI' ) && WP_CLI ) {
	\WP_CLI::add_command( 'nvm-inv sync', function( $args, $assoc_args ) {
		$force = isset( $assoc_args['force'] );
		\WP_CLI::log( 'Starting inventory sync...' );

		$service = Plugin::instance()->get_service( Stock_Service::class );
		$count   = $service->sync_all( $force );

		\WP_CLI::success( sprintf( 'Synced %d products.', $count ) );
	} );
}
```
