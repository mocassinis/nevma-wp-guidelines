# Security (Mandatory)

> Input sanitization, output escaping, nonces, capabilities, AJAX, REST, and SQL.

---

## Input Sanitization

```php
$name       = sanitize_text_field( wp_unslash( $_POST['name'] ?? '' ) );
$product_id = absint( $_GET['product_id'] ?? 0 );
$ids        = array_map( 'absint', (array) ( $_POST['ids'] ?? [] ) );
$email      = sanitize_email( wp_unslash( $_POST['email'] ?? '' ) );
$content    = wp_kses_post( wp_unslash( $_POST['content'] ?? '' ) );
```

---

## Output Escaping (At Render Time)

```php
echo esc_html( $text );
echo '<a href="' . esc_url( $url ) . '">' . esc_html( $label ) . '</a>';
echo '<input type="hidden" value="' . esc_attr( $value ) . '">';
```

---

## Nonces (Use Plugin Prefix)

```php
// Form: create — use Plugin::PREFIX for collision avoidance.
wp_nonce_field( Plugin::PREFIX . 'save_settings', Plugin::PREFIX . 'nonce' );

// Form: verify.
if ( ! wp_verify_nonce(
	sanitize_key( $_POST[ Plugin::PREFIX . 'nonce' ] ?? '' ),
	Plugin::PREFIX . 'save_settings'
) ) {
	wp_die( esc_html__( 'Security check failed.', 'nvm-inventory' ) );
}

// AJAX: verify.
check_ajax_referer( Plugin::PREFIX . 'ajax', 'nonce' );
```

---

## Capabilities

Always check before sensitive operations:

```php
if ( ! current_user_can( 'manage_woocommerce' ) ) {
	wp_die( esc_html__( 'Unauthorized.', 'nvm-inventory' ) );
}
```

---

## AJAX Handler Pattern (Mandatory)

Every AJAX handler must follow this pattern — nonce, capability, sanitize, respond, die:

```php
<?php
declare(strict_types=1);

namespace NVM\Inventory\Admin;

use NVM\Inventory\Plugin;
use NVM\Inventory\Services\Stock_Service;

if ( ! defined( 'ABSPATH' ) ) {
	exit;
}

/**
 * AJAX handlers for inventory admin.
 *
 * @since 1.0.0
 */
final class Ajax_Handler {

	public function __construct(
		private readonly Stock_Service $stock_service,
	) {
		add_action( 'wp_ajax_' . Plugin::PREFIX . 'update_stock', [ $this, 'handle_update_stock' ] );
	}

	/**
	 * Handle stock update AJAX request.
	 *
	 * @since 1.0.0
	 */
	public function handle_update_stock(): void {
		// 1. Verify nonce.
		check_ajax_referer( Plugin::PREFIX . 'ajax', 'nonce' );

		// 2. Check capabilities.
		if ( ! current_user_can( 'manage_woocommerce' ) ) {
			wp_send_json_error(
				[ 'message' => __( 'Unauthorized.', 'nvm-inventory' ) ],
				403
			);
		}

		// 3. Sanitize inputs.
		$product_id = absint( $_POST['product_id'] ?? 0 );
		$quantity   = (int) ( $_POST['quantity'] ?? 0 );

		if ( 0 === $product_id ) {
			wp_send_json_error(
				[ 'message' => __( 'Invalid product ID.', 'nvm-inventory' ) ],
				400
			);
		}

		// 4. Process.
		try {
			$result = $this->stock_service->update_stock( $product_id, $quantity );
			wp_send_json_success( $result );
		} catch ( \Exception $e ) {
			wc_get_logger()->error(
				$e->getMessage(),
				[ 'source' => Plugin::SLUG ]
			);
			wp_send_json_error(
				[ 'message' => __( 'Stock update failed.', 'nvm-inventory' ) ],
				500
			);
		}

		// wp_send_json_* calls wp_die() internally — no code after this.
	}
}
```

---

## REST API Security Pattern

```php
<?php
declare(strict_types=1);

namespace NVM\Inventory\REST;

use NVM\Inventory\Plugin;
use NVM\Inventory\Services\Stock_Service;

if ( ! defined( 'ABSPATH' ) ) {
	exit;
}

/**
 * REST API controller for stock data.
 *
 * @since 1.0.0
 */
final class Stock_Controller {

	private const NAMESPACE = 'nvm/inventory/v1';

	public function __construct(
		private readonly Stock_Service $stock_service,
	) {
		add_action( 'rest_api_init', [ $this, 'register_routes' ] );
	}

	/**
	 * Register REST API routes.
	 *
	 * @since 1.0.0
	 */
	public function register_routes(): void {
		register_rest_route( self::NAMESPACE, '/stock/(?P<id>\d+)', [
			'methods'             => \WP_REST_Server::READABLE,
			'callback'            => [ $this, 'get_stock' ],
			'permission_callback' => [ $this, 'check_read_permission' ],
			'args'                => [
				'id' => [
					'required'          => true,
					'type'              => 'integer',
					'sanitize_callback' => 'absint',
					'validate_callback' => static fn( $value ) => $value > 0,
				],
			],
		] );

		register_rest_route( self::NAMESPACE, '/stock/(?P<id>\d+)', [
			'methods'             => \WP_REST_Server::EDITABLE,
			'callback'            => [ $this, 'update_stock' ],
			'permission_callback' => [ $this, 'check_write_permission' ],
			'args'                => [
				'id' => [
					'required'          => true,
					'type'              => 'integer',
					'sanitize_callback' => 'absint',
				],
				'quantity' => [
					'required'          => true,
					'type'              => 'integer',
					'sanitize_callback' => 'intval',
				],
			],
		] );
	}

	/**
	 * Check read permission.
	 *
	 * CRITICAL: Never return true without a check. Always validate capability.
	 */
	public function check_read_permission( \WP_REST_Request $request ): bool {
		return current_user_can( 'read_shop_reports' )
			|| current_user_can( 'manage_woocommerce' );
	}

	/**
	 * Check write permission.
	 */
	public function check_write_permission( \WP_REST_Request $request ): bool {
		return current_user_can( 'manage_woocommerce' );
	}

	/**
	 * Get stock for a product.
	 */
	public function get_stock( \WP_REST_Request $request ): \WP_REST_Response {
		$product_id = $request->get_param( 'id' );

		try {
			$data = $this->stock_service->get_stock_summary( $product_id );
			return new \WP_REST_Response( $data, 200 );
		} catch ( \InvalidArgumentException $e ) {
			return new \WP_REST_Response(
				[ 'message' => $e->getMessage() ],
				404
			);
		}
	}

	/**
	 * Update stock for a product.
	 */
	public function update_stock( \WP_REST_Request $request ): \WP_REST_Response {
		$product_id = $request->get_param( 'id' );
		$quantity   = $request->get_param( 'quantity' );

		try {
			$result = $this->stock_service->update_stock( $product_id, $quantity );
			return new \WP_REST_Response( $result, 200 );
		} catch ( \Exception $e ) {
			wc_get_logger()->error(
				$e->getMessage(),
				[ 'source' => Plugin::SLUG ]
			);
			return new \WP_REST_Response(
				[ 'message' => __( 'Update failed.', 'nvm-inventory' ) ],
				500
			);
		}
	}
}
```

---

## SQL Injection Protection

Avoid direct SQL. When unavoidable (custom tables, reporting), **always** use `$wpdb->prepare()`:

```php
global $wpdb;

$table = $wpdb->prefix . 'nvm_inv_sync_log';

// CORRECT — parameterized.
$results = $wpdb->get_results(
	$wpdb->prepare(
		"SELECT * FROM {$table} WHERE product_id = %d AND status = %s ORDER BY created_at DESC LIMIT %d",
		$product_id,
		$status,
		$limit
	)
);

// NEVER do this:
// $wpdb->get_results( "SELECT * FROM {$table} WHERE product_id = {$product_id}" );
```

### Custom Table Creation

Use `dbDelta` with proper prefix:

```php
public function create_tables(): void {
	global $wpdb;

	$table   = $wpdb->prefix . 'nvm_inv_sync_log';
	$charset = $wpdb->get_charset_collate();

	$sql = "CREATE TABLE {$table} (
		id bigint(20) unsigned NOT NULL AUTO_INCREMENT,
		product_id bigint(20) unsigned NOT NULL,
		status varchar(20) NOT NULL DEFAULT 'pending',
		message text,
		created_at datetime NOT NULL DEFAULT CURRENT_TIMESTAMP,
		PRIMARY KEY (id),
		KEY product_id (product_id),
		KEY status_created (status, created_at)
	) {$charset};";

	require_once ABSPATH . 'wp-admin/includes/upgrade.php';
	dbDelta( $sql );
}
```

---

## File Upload Security

When handling file imports (CSV, XML):

```php
/**
 * Validate and process an uploaded file.
 *
 * @param array<string, mixed> $file $_FILES array entry.
 */
public function process_upload( array $file ): string|false {
	// 1. Validate file type.
	$allowed = [ 'csv' => 'text/csv', 'xml' => 'application/xml' ];
	$filetype = wp_check_filetype( $file['name'], $allowed );

	if ( ! $filetype['ext'] ) {
		return false;
	}

	// 2. Move to safe temp location.
	$upload = wp_handle_upload( $file, [
		'test_form' => false,
		'mimes'     => $allowed,
	] );

	if ( isset( $upload['error'] ) ) {
		return false;
	}

	// 3. Never trust $_FILES['name'] — use the sanitized path.
	return $upload['file'];
}
```

---

## Rate Limiting for Expensive Endpoints

```php
/**
 * Throttle an action using transients.
 *
 * @param string $action Action identifier.
 * @param int    $interval Minimum seconds between executions.
 */
private function throttle( string $action, int $interval = 60 ): bool {
	$key = Plugin::PREFIX . 'throttle_' . $action;

	if ( get_transient( $key ) ) {
		return false; // Still throttled.
	}

	set_transient( $key, time(), $interval );
	return true; // Allowed.
}
```

---

## Direct Access Prevention

Every PHP file starts with:

```php
if ( ! defined( 'ABSPATH' ) ) {
	exit;
}
```
