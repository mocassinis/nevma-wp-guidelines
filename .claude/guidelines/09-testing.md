# Testing

> PHPUnit setup, Brain Monkey, test patterns, integration tests (wp-phpunit), coverage enforcement, and mutation testing.

Testing is **mandatory** for all service classes and business logic.

---

## Test-Driven Development (TDD)

**Write tests first, then implement the function.**

### The TDD Cycle

```
1. RED    → Write a failing test for the desired behavior
2. GREEN  → Write the minimum code to make the test pass
3. REFACTOR → Clean up the code while keeping tests green
```

### Workflow

1. **Define the interface** — Decide on method signature, parameters, and return type
2. **Write the test** — Test the expected behavior (test will fail initially)
3. **Run the test** — Confirm it fails for the right reason
4. **Implement the function** — Write just enough code to pass
5. **Run the test** — Confirm it passes
6. **Refactor** — Improve code quality without changing behavior
7. **Repeat** — Add more test cases for edge cases and error conditions

### Example: TDD for a Discount Calculator

**Step 1: Write the test first**

```php
public function test_apply_discount_with_20_percent_returns_correct_price(): void {
    $calculator = new Price_Calculator();

    $result = $calculator->apply_discount( 100.00, 20.0 );

    $this->assertSame( 80.00, $result );
}
```

**Step 2: Run test — it fails** (class doesn't exist yet)

**Step 3: Implement the minimum code**

```php
class Price_Calculator {
    public function apply_discount( float $price, float $percent ): float {
        return $price - ( $price * $percent / 100 );
    }
}
```

**Step 4: Run test — it passes**

**Step 5: Add edge case tests**

```php
public function test_apply_discount_rejects_negative_price(): void {
    $this->expectException( \InvalidArgumentException::class );

    $calculator = new Price_Calculator();
    $calculator->apply_discount( -100.00, 10.0 );
}
```

**Step 6: Implement validation**

```php
public function apply_discount( float $price, float $percent ): float {
    if ( $price < 0 ) {
        throw new \InvalidArgumentException( 'Price cannot be negative.' );
    }

    return $price - ( $price * $percent / 100 );
}
```

### Benefits of TDD

- **Clear requirements** — Tests define expected behavior before coding
- **Better design** — Forces you to think about interfaces first
- **Confidence** — Know immediately when something breaks
- **Documentation** — Tests serve as living documentation

---

## Bug Fix Workflow (Test-First)

**Never fix a bug directly. Always write a failing test first.**

### The Bug Fix Cycle

```
1. REPRODUCE → Write a test that reproduces the bug (test fails)
2. VERIFY    → Run the test to confirm it fails for the right reason
3. FIX       → Write the minimum code to fix the bug
4. CONFIRM   → Run the test to confirm it passes
5. REVIEW    → Check for related edge cases, add more tests if needed
```

### Why Test-First for Bugs?

- **Proves the bug exists** — The failing test documents the exact issue
- **Prevents regression** — The test ensures the bug never returns
- **Defines "fixed"** — Clear success criteria for the fix
- **Documents behavior** — Future developers understand what went wrong

### Example: Bug Fix Workflow

**Bug Report:** "Discount calculator returns wrong value for 100% discount"

**Step 1: Write the failing test**

```php
public function test_apply_discount_with_100_percent_returns_zero(): void {
    $calculator = new Price_Calculator();

    $result = $calculator->apply_discount( 50.00, 100.0 );

    $this->assertSame( 0.00, $result ); // Bug: currently returns -50.00
}
```

**Step 2: Run test — it fails** (confirms the bug exists)

```
FAILED: Expected 0.00, got -50.00
```

**Step 3: Fix the bug**

```php
public function apply_discount( float $price, float $percent ): float {
    if ( $percent >= 100.0 ) {
        return 0.00; // Fix: cap at zero
    }

    return $price - ( $price * $percent / 100 );
}
```

**Step 4: Run test — it passes**

**Step 5: Add related edge cases**

```php
public function test_apply_discount_over_100_percent_returns_zero(): void {
    $calculator = new Price_Calculator();

    $result = $calculator->apply_discount( 50.00, 150.0 );

    $this->assertSame( 0.00, $result );
}
```

### Bug Fix Test Naming

Use descriptive names that document the bug:

```
test_{method}_bug_{issue_number}_{description}
test_{method}_{scenario_that_caused_bug}
```

Examples:
- `test_apply_discount_bug_123_100_percent_returns_zero()`
- `test_calculate_total_with_empty_cart_returns_zero()`
- `test_sync_handles_deleted_product_gracefully()`

---

## Directory Structure

```
nvm-inventory/
├── src/
│   └── Services/
│       └── Stock_Service.php
├── tests/
│   ├── bootstrap.php                # Test setup
│   ├── Unit_Test_Case.php           # Base class for unit tests
│   ├── Integration_Test_Case.php    # Base class for integration tests
│   ├── Unit/
│   │   ├── Services/
│   │   │   └── Stock_ServiceTest.php
│   │   ├── REST/
│   │   │   └── Stock_ControllerTest.php
│   │   └── Enums/
│   │       └── Stock_StatusTest.php
│   └── Integration/
│       └── Services/
│           └── Stock_Service_IntegrationTest.php
├── composer.json
└── phpunit.xml
```

---

## phpunit.xml Configuration

```xml
<?xml version="1.0"?>
<phpunit
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:noNamespaceSchemaLocation="https://schema.phpunit.de/10.5/phpunit.xsd"
	bootstrap="tests/bootstrap.php"
	colors="true"
	beStrictAboutTestsThatDoNotTestAnything="true"
	failOnWarning="true"
	failOnRisky="true"
>
	<testsuites>
		<testsuite name="unit">
			<directory suffix="Test.php">tests/Unit</directory>
		</testsuite>
		<testsuite name="integration">
			<directory suffix="Test.php">tests/Integration</directory>
		</testsuite>
	</testsuites>

	<source>
		<include>
			<directory suffix=".php">src</directory>
		</include>
	</source>

	<coverage>
		<report>
			<html outputDirectory="coverage"/>
		</report>
	</coverage>
</phpunit>
```

---

## tests/bootstrap.php

```php
<?php
declare(strict_types=1);

/**
 * PHPUnit bootstrap file.
 *
 * @package NVM\Inventory\Tests
 */

// Load Composer autoloader (includes test classes via autoload-dev).
require_once dirname( __DIR__ ) . '/vendor/autoload.php';

// Define WordPress constants used in plugin code.
if ( ! defined( 'ABSPATH' ) ) {
	define( 'ABSPATH', sys_get_temp_dir() . '/wordpress/' );
}

if ( ! defined( 'NVM_INV_FILE' ) ) {
	define( 'NVM_INV_FILE', dirname( __DIR__ ) . '/nvm-inventory.php' );
}

if ( ! defined( 'NVM_INV_PATH' ) ) {
	define( 'NVM_INV_PATH', dirname( __DIR__ ) . '/' );
}

if ( ! defined( 'HOUR_IN_SECONDS' ) ) {
	define( 'HOUR_IN_SECONDS', 3600 );
}

if ( ! defined( 'DAY_IN_SECONDS' ) ) {
	define( 'DAY_IN_SECONDS', 86400 );
}

if ( ! defined( 'ARRAY_A' ) ) {
	define( 'ARRAY_A', 'ARRAY_A' );
}
```

---

## Unit Test Case Base Class

```php
<?php
declare(strict_types=1);

namespace NVM\Inventory\Tests;

use Brain\Monkey;
use Brain\Monkey\Functions;
use Mockery\Adapter\Phpunit\MockeryPHPUnitIntegration;
use PHPUnit\Framework\TestCase;

/**
 * Base class for all unit tests.
 *
 * Provides Brain Monkey setup and common WordPress function stubs.
 *
 * @since 1.0.0
 */
abstract class Unit_Test_Case extends TestCase {

	use MockeryPHPUnitIntegration;

	protected function setUp(): void {
		parent::setUp();
		Monkey\setUp();
		$this->setup_common_wp_functions();
	}

	protected function tearDown(): void {
		Monkey\tearDown();
		parent::tearDown();
	}

	/**
	 * Stub commonly used WordPress functions.
	 */
	protected function setup_common_wp_functions(): void {
		// Escaping functions — pass through.
		Functions\stubs( [
			'esc_html'            => static fn( string $t ): string => $t,
			'esc_attr'            => static fn( string $t ): string => $t,
			'esc_url'             => static fn( string $u ): string => $u,
			'wp_kses_post'        => static fn( string $s ): string => $s,
			'__'                  => static fn( string $t ): string => $t,
			'esc_html__'          => static fn( string $t ): string => $t,
			'sanitize_text_field' => static fn( string $s ): string => trim( $s ),
			'sanitize_key'        => static fn( string $s ): string => strtolower( preg_replace( '/[^a-zA-Z0-9_\-]/', '', $s ) ),
			'absint'              => static fn( $v ): int => abs( (int) $v ),
			'wp_unslash'          => static fn( $v ) => is_string( $v ) ? stripslashes( $v ) : $v,
			'wp_json_encode'      => static fn( $v, int $flags = 0 ): string => json_encode( $v, $flags ),
		] );

		// Hook functions — no-op by default.
		Functions\stubs( [
			'add_action'    => '__return_true',
			'add_filter'    => '__return_true',
			'do_action'     => '__return_null',
			'apply_filters' => static fn( string $tag, $value ) => $value,
		] );

		Functions\when( 'current_user_can' )->justReturn( true );
		Functions\when( 'plugin_dir_url' )->justReturn( 'https://example.com/wp-content/plugins/nvm-inventory/' );
		Functions\when( 'plugin_dir_path' )->justReturn( NVM_INV_PATH );
		Functions\when( 'plugin_basename' )->justReturn( 'nvm-inventory/nvm-inventory.php' );
	}

	/**
	 * Create a mock WC_Product with common methods.
	 *
	 * @param array<string, mixed> $props Product properties.
	 * @return \Mockery\MockInterface&\WC_Product
	 */
	protected function create_product_mock( array $props = [] ): \Mockery\MockInterface {
		$defaults = [
			'id'             => 123,
			'name'           => 'Test Product',
			'stock_quantity' => 10,
			'sku'            => 'TEST-001',
			'status'         => 'publish',
			'type'           => 'simple',
		];

		$props = array_merge( $defaults, $props );

		$product = \Mockery::mock( 'WC_Product' );
		$product->shouldReceive( 'get_id' )->andReturn( $props['id'] );
		$product->shouldReceive( 'get_name' )->andReturn( $props['name'] );
		$product->shouldReceive( 'get_stock_quantity' )->andReturn( $props['stock_quantity'] );
		$product->shouldReceive( 'get_sku' )->andReturn( $props['sku'] );
		$product->shouldReceive( 'get_status' )->andReturn( $props['status'] );
		$product->shouldReceive( 'get_type' )->andReturn( $props['type'] );
		$product->shouldReceive( 'save' )->andReturnNull();
		$product->shouldReceive( 'set_stock_quantity' )->andReturnNull();

		return $product;
	}
}
```

---

## Test Naming Convention

```
test_{method}_{scenario}_{expected_result}
```

Examples:
- `test_apply_discount_with_20_percent_returns_correct_price()`
- `test_is_low_stock_returns_true_when_below_threshold()`
- `test_get_stock_throws_exception_for_invalid_product()`
- `test_update_stock_calls_cache_invalidation()`

---

## Test Patterns

### Pattern 1: Testing Pure Logic (No WordPress)

```php
public function test_apply_discount_with_20_percent_returns_correct_price(): void {
	$calculator = new Price_Calculator();

	$result = $calculator->apply_discount( 100.00, 20.0 );

	$this->assertSame( 80.00, $result );
}
```

### Pattern 2: Testing with WordPress Options

```php
public function test_uses_threshold_from_settings(): void {
	Functions\when( 'get_option' )->justReturn( [ 'low_stock_threshold' => 15 ] );

	$service = new Stock_Service();

	$result = $service->get_low_stock_threshold();

	$this->assertSame( 15, $result );
}
```

### Pattern 3: Testing Caching (Transient Hit & Miss)

```php
public function test_returns_cached_data_when_available(): void {
	$cached_data = [ 'stock' => 50, 'is_low' => false ];

	Functions\when( 'get_transient' )->justReturn( $cached_data );

	$service = new Stock_Service();
	$result  = $service->get_stock_summary( 123 );

	$this->assertSame( $cached_data, $result );
}

public function test_sets_cache_when_fetching_fresh_data(): void {
	Functions\when( 'get_transient' )->justReturn( false );

	$product = $this->create_product_mock( [ 'stock_quantity' => 25 ] );
	Functions\when( 'wc_get_product' )->justReturn( $product );
	Functions\when( 'get_option' )->justReturn( [] );

	Functions\expect( 'set_transient' )
		->once()
		->with( 'nvm_inv_stock_123', \Mockery::type( 'array' ), HOUR_IN_SECONDS )
		->andReturn( true );

	$service = new Stock_Service();
	$service->get_stock_summary( 123 );
}

public function test_invalidates_cache_on_stock_update(): void {
	$product = $this->create_product_mock();
	Functions\when( 'wc_get_product' )->justReturn( $product );

	Functions\expect( 'delete_transient' )
		->once()
		->with( 'nvm_inv_expensive_data_123' )
		->andReturn( true );

	Functions\expect( 'wp_cache_delete' )
		->once()
		->with( 'nvm_inv_stock_123', 'nvm-inventory' )
		->andReturn( true );

	$service = new Stock_Service();
	$service->update_stock( 123, 50 );
}
```

### Pattern 4: Testing Exceptions

```php
public function test_throws_exception_for_negative_price(): void {
	$this->expectException( \InvalidArgumentException::class );
	$this->expectExceptionMessage( 'Price cannot be negative.' );

	$calculator = new Price_Calculator();
	$calculator->apply_discount( -100.00, 10.0 );
}
```

### Pattern 5: Data Providers (Multiple Scenarios)

```php
public static function discount_scenarios(): array {
	return [
		'10% off €100'     => [ 100.00, 10.0, 90.00 ],
		'25% off €80'      => [ 80.00, 25.0, 60.00 ],
		'50% off €50'      => [ 50.00, 50.0, 25.00 ],
		'no discount'      => [ 100.00, 0.0, 100.00 ],
		'100% off'         => [ 100.00, 100.0, 0.00 ],
		'zero price'       => [ 0.00, 50.0, 0.00 ],
	];
}

#[\PHPUnit\Framework\Attributes\DataProvider('discount_scenarios')]
public function test_apply_discount_scenarios( float $price, float $discount, float $expected ): void {
	$calculator = new Price_Calculator();

	$result = $calculator->apply_discount( $price, $discount );

	$this->assertSame( $expected, $result );
}
```

### Pattern 6: Testing WooCommerce Product Interactions

```php
public function test_get_stock_returns_quantity_from_product(): void {
	$product = $this->create_product_mock( [ 'stock_quantity' => 42 ] );
	Functions\when( 'wc_get_product' )->justReturn( $product );
	Functions\when( 'get_transient' )->justReturn( false );
	Functions\when( 'set_transient' )->justReturn( true );
	Functions\when( 'get_option' )->justReturn( [] );

	$service = new Stock_Service();
	$result  = $service->get_stock_summary( 123 );

	$this->assertSame( 42, $result['quantity'] );
}

public function test_get_stock_returns_null_for_nonexistent_product(): void {
	Functions\when( 'wc_get_product' )->justReturn( false );
	Functions\when( 'get_transient' )->justReturn( false );

	$this->expectException( \InvalidArgumentException::class );

	$service = new Stock_Service();
	$service->get_stock_summary( 999 );
}
```

### Pattern 7: Testing Hook Registration

```php
use Brain\Monkey\Actions;
use Brain\Monkey\Filters;

public function test_registers_stock_updated_hook(): void {
	Actions\expectDone( 'nvm/inventory/stock_updated' )
		->once()
		->with( 123, 50 );

	$product = $this->create_product_mock();
	Functions\when( 'wc_get_product' )->justReturn( $product );
	Functions\when( 'delete_transient' )->justReturn( true );
	Functions\when( 'wp_cache_delete' )->justReturn( true );

	$service = new Stock_Service();
	$service->update_stock( 123, 50 );
}
```

### Pattern 8: Testing Enums (PHP 8.1+)

```php
public function test_stock_status_label(): void {
	$this->assertSame( 'In Stock', Stock_Status::IN_STOCK->label() );
	$this->assertSame( 'Out of Stock', Stock_Status::OUT_OF_STOCK->label() );
}

public function test_stock_status_purchasable(): void {
	$this->assertTrue( Stock_Status::IN_STOCK->is_purchasable() );
	$this->assertTrue( Stock_Status::LOW_STOCK->is_purchasable() );
	$this->assertFalse( Stock_Status::OUT_OF_STOCK->is_purchasable() );
}

public function test_stock_status_from_string(): void {
	$status = Stock_Status::from( 'instock' );
	$this->assertSame( Stock_Status::IN_STOCK, $status );
}

public function test_stock_status_tryfrom_returns_null_for_invalid(): void {
	$status = Stock_Status::tryFrom( 'invalid' );
	$this->assertNull( $status );
}
```

### Pattern 9: Testing AJAX Handler (Security Path)

```php
public function test_ajax_handler_rejects_missing_nonce(): void {
	Functions\expect( 'check_ajax_referer' )
		->once()
		->with( 'nvm_inv_ajax', 'nonce' )
		->andThrow( new \WPDieException( 'Invalid nonce.' ) );

	$this->expectException( \WPDieException::class );

	$handler = new Ajax_Handler( new Stock_Service() );
	$handler->handle_update_stock();
}

public function test_ajax_handler_rejects_unauthorized_user(): void {
	Functions\when( 'check_ajax_referer' )->justReturn( true );
	Functions\when( 'current_user_can' )->justReturn( false );

	Functions\expect( 'wp_send_json_error' )
		->once()
		->with( \Mockery::type( 'array' ), 403 );

	$handler = new Ajax_Handler( new Stock_Service() );
	$handler->handle_update_stock();
}
```

### Pattern 10: Testing Activation/Deactivation

```php
public function test_activate_stores_version(): void {
	Functions\expect( 'update_option' )
		->once()
		->with( 'nvm_inv_version', '1.0.0' )
		->andReturn( true );

	Functions\when( 'register_activation_hook' )->justReturn( true );
	Functions\when( 'register_deactivation_hook' )->justReturn( true );

	$plugin = Plugin::instance();
	$plugin->activate();
}

public function test_deactivate_cleans_scheduled_actions(): void {
	Functions\expect( 'as_unschedule_all_actions' )
		->once()
		->with( 'nvm/inventory/sync', [], 'nvm-inventory' );

	$plugin = Plugin::instance();
	$plugin->deactivate();
}
```

### Pattern 11: Testing Filters

Pattern 7 covers actions — filters use `Filters\expectApplied`:

```php
use Brain\Monkey\Filters;

public function test_low_stock_threshold_filter_is_applied(): void {
	Filters\expectApplied( 'nvm/inventory/low_stock_threshold' )
		->once()
		->with( 5 )
		->andReturn( 10 );

	Functions\when( 'get_option' )->justReturn( [] );

	$service = new Stock_Service();

	// Filtered value must be used, not the default.
	$this->assertSame( 10, $service->get_low_stock_threshold() );
}
```

### Pattern 12: Mocking HTTP Calls

Never let unit tests hit the network. Test **all three** outcomes: success, `WP_Error` (network failure), and non-2xx response.

```php
public function test_push_stock_succeeds_on_200(): void {
	$response = [
		'response' => [ 'code' => 200 ],
		'body'     => '{"ok":true}',
	];

	Functions\expect( 'wp_remote_post' )
		->once()
		->with( 'https://api.example.com/stock', \Mockery::type( 'array' ) )
		->andReturn( $response );

	Functions\when( 'is_wp_error' )->justReturn( false );
	Functions\when( 'wp_remote_retrieve_response_code' )->justReturn( 200 );
	Functions\when( 'wp_remote_retrieve_body' )->justReturn( '{"ok":true}' );

	$client = new Api_Client();

	$this->assertTrue( $client->push_stock( 123, 50 ) );
}

public function test_push_stock_handles_network_failure(): void {
	$error = \Mockery::mock( 'WP_Error' );
	$error->shouldReceive( 'get_error_message' )->andReturn( 'cURL error 28: timed out' );

	Functions\when( 'wp_remote_post' )->justReturn( $error );
	Functions\when( 'is_wp_error' )->alias( static fn( $thing ): bool => $thing === $error );

	$client = new Api_Client();

	$this->assertFalse( $client->push_stock( 123, 50 ) );
}

public function test_push_stock_handles_http_500(): void {
	Functions\when( 'wp_remote_post' )->justReturn( [ 'response' => [ 'code' => 500 ], 'body' => '' ] );
	Functions\when( 'is_wp_error' )->justReturn( false );
	Functions\when( 'wp_remote_retrieve_response_code' )->justReturn( 500 );

	$client = new Api_Client();

	$this->assertFalse( $client->push_stock( 123, 50 ) );
}
```

In **integration** tests, short-circuit real requests with the `pre_http_request` filter instead of stubbing functions.

### Pattern 13: Testing Action Scheduler

`06-performance.md` mandates Action Scheduler for heavy work — test the scheduling contract:

```php
public function test_queues_async_sync_job(): void {
	Functions\expect( 'as_enqueue_async_action' )
		->once()
		->with( 'nvm/inventory/sync_product', [ 'product_id' => 123 ], 'nvm-inventory' );

	$service = new Sync_Service();
	$service->queue_product_sync( 123 );
}

public function test_does_not_double_schedule_recurring_job(): void {
	Functions\when( 'as_next_scheduled_action' )->justReturn( 1735689600 ); // Already scheduled.
	Functions\expect( 'as_schedule_recurring_action' )->never();

	$service = new Sync_Service();
	$service->maybe_schedule_daily_sync();
}
```

### Pattern 14: Mocking `$wpdb` (Custom Tables)

```php
public function test_get_log_entries_prepares_query(): void {
	global $wpdb;

	$wpdb         = \Mockery::mock( 'wpdb' );
	$wpdb->prefix = 'wp_';

	$wpdb->shouldReceive( 'prepare' )
		->once()
		->with( \Mockery::pattern( '/FROM wp_nvm_inventory_log WHERE product_id = %d/' ), 123, 50 )
		->andReturn( 'prepared-sql' );

	$wpdb->shouldReceive( 'get_results' )
		->once()
		->with( 'prepared-sql', ARRAY_A )
		->andReturn( [ [ 'id' => '1' ] ] );

	$repo = new Log_Repository();

	$this->assertCount( 1, $repo->get_entries( 123 ) );
}
```

Requirements: define `ARRAY_A` in `tests/bootstrap.php` (`define( 'ARRAY_A', 'ARRAY_A' );`) and reset `$wpdb = null;` in `tearDown()` so tests stay isolated. Asserting the `prepare()` call **is** the SQL-injection test.

### Pattern 15: Time-Dependent Logic

Never call `time()`/`new \DateTimeImmutable( 'now' )` directly in services — inject a clock so expiry/scheduling logic is deterministic:

```php
interface Clock {
	public function now(): int;
}

final class System_Clock implements Clock {
	public function now(): int {
		return time();
	}
}

// In the test — frozen clock, no sleep(), no flaky boundaries.
$frozen = new class() implements Clock {
	public function now(): int {
		return 1_700_000_000;
	}
};

$service = new Token_Service( $frozen );

$this->assertTrue( $service->is_expired( 1_700_000_000 - HOUR_IN_SECONDS - 1 ) );
$this->assertFalse( $service->is_expired( 1_700_000_000 - 10 ) );
```

Brain Monkey cannot stub PHP built-ins like `time()` in the global namespace — the clock interface is the reliable pattern.

---

## Negative Path Testing (Mandatory Checklist)

Every service class must include tests for these edge cases:

| Input Type | Test Scenarios |
|------------|----------------|
| **Integer** | 0, -1, PHP_INT_MAX, very large numbers |
| **Float** | 0.00, -0.01, NAN (if applicable), very small decimals |
| **String** | Empty string `''`, whitespace-only `'  '`, UTF-8/Greek text `'Τεστ'`, HTML `'<script>'`, very long strings (1000+ chars) |
| **Array** | Empty `[]`, single item, very large (1000+ items), nested |
| **Null** | Explicitly `null` where union types allow it |
| **Product ID** | 0, nonexistent ID, deleted product, wrong post type |
| **Duplicate** | Submitting the same operation twice in sequence |

```php
public static function invalid_product_ids(): array {
	return [
		'zero'     => [ 0 ],
		'negative' => [ -1 ],
		'max_int'  => [ PHP_INT_MAX ],
	];
}

#[\PHPUnit\Framework\Attributes\DataProvider('invalid_product_ids')]
public function test_rejects_invalid_product_id( int $id ): void {
	Functions\when( 'wc_get_product' )->justReturn( false );
	Functions\when( 'get_transient' )->justReturn( false );

	$this->expectException( \InvalidArgumentException::class );

	$service = new Stock_Service();
	$service->get_stock_summary( $id );
}
```

---

## Common Assertions

```php
// Equality.
$this->assertSame( 100, $result );         // Strict type + value (preferred).
$this->assertEquals( 100, $result );       // Loose comparison (use sparingly).

// Boolean.
$this->assertTrue( $result );
$this->assertFalse( $result );

// Null.
$this->assertNull( $result );
$this->assertNotNull( $result );

// Arrays.
$this->assertIsArray( $result );
$this->assertArrayHasKey( 'key', $result );
$this->assertCount( 3, $result );
$this->assertEmpty( $result );

// Strings.
$this->assertStringContainsString( 'needle', $result );
$this->assertStringStartsWith( 'Hello', $result );
$this->assertMatchesRegularExpression( '/^\d+$/', $result );

// Types.
$this->assertIsInt( $result );
$this->assertIsFloat( $result );
$this->assertInstanceOf( My_Class::class, $result );

// Exceptions.
$this->expectException( \InvalidArgumentException::class );
$this->expectExceptionMessage( 'Specific message' );
```

---

## What to Test (Mandatory)

| Component | Test Coverage Required |
|-----------|------------------------|
| Service classes | All public methods |
| Calculations | All edge cases (zero, negative, max values) |
| Validation | Valid and invalid inputs (see negative path checklist) |
| Caching logic | Cache hit, cache miss, cache invalidation |
| Settings retrieval | Default values, custom values |
| Data transformations | Input/output mapping |
| AJAX handlers | Nonce verification, capability checks, error responses |
| REST controllers | Permission callbacks, argument validation, error responses |
| Enums | All cases, labels, from/tryFrom |
| Hook firing | Correct hook name, correct arguments |

## What NOT to Unit Test

| Component | Why | How to Test Instead |
|-----------|-----|---------------------|
| WordPress hook registration order | Integration test territory | Integration tests or manual |
| Direct database queries | Use WooCommerce CRUD instead | Integration tests with WP test suite |
| Admin UI rendering | Manual or E2E testing | Browser tests (Playwright/Cypress) |
| Third-party API calls | Mock the HTTP client | Unit test with mocked `wp_remote_get` |
| Private methods directly | Test through public interface | Test the public methods that call them |

---

## Integration Tests (wp-phpunit)

Everything the table above defers — hook wiring, real CRUD round-trips, `$wpdb` against real tables — belongs in `tests/Integration/` running against a real WordPress + WooCommerce.

### Setup

```bash
composer require --dev wp-phpunit/wp-phpunit yoast/phpunit-polyfills
```

Integration tests need MySQL — run them inside wp-env's tests environment (see below), never against a live site.

### tests/bootstrap-integration.php

```php
<?php
declare(strict_types=1);

$_tests_dir = getenv( 'WP_TESTS_DIR' ) ?: dirname( __DIR__ ) . '/vendor/wp-phpunit/wp-phpunit';

require_once $_tests_dir . '/includes/functions.php';

// Load WooCommerce and the plugin before the WP test suite boots.
tests_add_filter( 'muplugins_loaded', static function (): void {
	require getenv( 'WC_PLUGIN_FILE' ) ?: WP_PLUGIN_DIR . '/woocommerce/woocommerce.php';
	require dirname( __DIR__ ) . '/nvm-inventory.php';
} );

require $_tests_dir . '/includes/bootstrap.php';
```

Use a second phpunit config (`phpunit-integration.xml`) pointing `bootstrap` at this file and the testsuite at `tests/Integration`.

### Integration Test Case Base Class

```php
<?php
declare(strict_types=1);

namespace NVM\Inventory\Tests;

abstract class Integration_Test_Case extends \WP_UnitTestCase {

	/**
	 * Create a real product in the test database (rolled back after each test).
	 *
	 * @param array<string, mixed> $props Product properties.
	 */
	protected function create_simple_product( array $props = [] ): \WC_Product_Simple {
		$product = new \WC_Product_Simple();
		$product->set_name( $props['name'] ?? 'Integration Test Product' );
		$product->set_regular_price( $props['price'] ?? '19.99' );
		$product->set_manage_stock( true );
		$product->set_stock_quantity( $props['stock'] ?? 10 );
		$product->save();

		return $product;
	}
}
```

`WP_UnitTestCase` wraps every test in a DB transaction and rolls it back — tests stay isolated without manual cleanup.

### Example: Real Round-Trip + Hook Wiring

```php
public function test_update_stock_persists_and_fires_hook(): void {
	$product = $this->create_simple_product( [ 'stock' => 10 ] );
	$fired   = did_action( 'nvm/inventory/stock_updated' );

	( new Stock_Service() )->update_stock( $product->get_id(), 50 );

	// Re-read from the database — not the in-memory object.
	$this->assertSame( 50, wc_get_product( $product->get_id() )->get_stock_quantity() );
	$this->assertSame( $fired + 1, did_action( 'nvm/inventory/stock_updated' ) );
}
```

### Running

```bash
npx wp-env start
npx wp-env run tests-cli --env-cwd=wp-content/plugins/nvm-inventory \
    vendor/bin/phpunit -c phpunit-integration.xml
```

### What Belongs Where

| Test type | Speed | Use for |
|-----------|-------|---------|
| Unit (Brain Monkey) | ms | Business logic, calculations, branches, error paths |
| Integration (wp-phpunit) | seconds | Hook wiring, CRUD round-trips, `$wpdb` schema/queries, meta persistence |
| E2E (Playwright) | minutes | User-visible flows only (see `14-e2e-testing.md`) |

Don't duplicate: a branch tested at unit level does not need an integration test — integration tests verify the *wiring*, not the logic again.

---

## Coverage Enforcement

An HTML report nobody reads is not a gate. Enforce a threshold in CI:

```bash
composer require --dev rregeer/phpunit-coverage-check
```

```json
{
	"scripts": {
		"test:coverage": "XDEBUG_MODE=coverage phpunit --testsuite unit --coverage-html coverage --coverage-clover coverage.xml",
		"coverage:check": "coverage-check coverage.xml 80"
	}
}
```

Rules:

- **80% line coverage minimum on `src/Services/`** (business logic). Glue code (Plugin class, hook registration) is exempt — it's integration-test territory.
- The threshold may only go **up** as the project matures.
- **Coverage ≠ correctness.** A line executed by a test with a weak assertion counts as covered. That gap is what mutation testing measures — see next section.

---

## Mutation Testing (Infection)

Infection mutates the code (`>=` → `>`, `+` → `-`, removes method calls) and re-runs the tests. A mutant that survives means behavior no test asserts — the true measure of test quality, beyond coverage.

```bash
composer require --dev infection/infection
```

### infection.json5

```json5
{
	"$schema": "vendor/infection/infection/resources/schema.json",
	"source": {
		"directories": [ "src/Services" ]   // Scope to business logic — keeps runs fast.
	},
	"timeout": 10,
	"logs": {
		"text": "infection.log"
	},
	"mutators": {
		"@default": true
	},
	"minMsi": 70,
	"minCoveredMsi": 80
}
```

### Running

```bash
XDEBUG_MODE=coverage vendor/bin/infection --threads=max --show-mutations
```

- **MSI ≥ 70** (all code in scope) and **covered MSI ≥ 80** (code that has tests must have *meaningful* tests) — CI fails below either.
- A surviving mutant = missing assertion. Read `infection.log`, add the test, don't lower the threshold.
- Run on services only; mutating hook-registration glue produces noise, not signal.

---

## Test Anti-Patterns

| Anti-pattern | Symptom | Fix |
|--------------|---------|-----|
| **Over-mocking** | Test breaks on every refactor but passes when real code is broken | Mock only true boundaries (WP functions, HTTP, DB) — never the class under test or its value objects |
| **Testing the mock** | All assertions check what the mock returned | Assert on the *system under test's* output/behavior |
| **Implementation-detail assertions** | `->once()` / exact call-order expectations on incidental internals | Use `Functions\when()` (stub) unless the call *is* the contract (e.g. `set_transient`, `as_enqueue_async_action`) |
| **Assertion-free tests** | Test passes because nothing is checked | `beStrictAboutTestsThatDoNotTestAnything` (already in phpunit.xml) + code review |
| **Shared state** | Tests pass alone, fail in suite (or vice versa) | No static state between tests; reset globals (`$wpdb`) in `tearDown()` |
| **Logic in tests** | `if`/`foreach`/`try-catch` inside a test body | Split scenarios into data providers; let exceptions bubble to `expectException` |
| **Sleeping** | `sleep()`/`usleep()` for timing behavior | Inject a clock (Pattern 15) |

Rule of thumb: `Functions\expect()` for **outbound contracts** (cache writes, scheduling, hooks fired), `Functions\when()` for **ambient environment** (options, escaping, current user). If reversing them wouldn't change what the test proves, the expectation is noise.

---

## Running Tests

```bash
# Run all tests.
composer test

# Run only unit tests.
composer test:unit

# Run only integration tests.
composer test:integration

# Run with detailed output.
./vendor/bin/phpunit --testdox

# Run specific test.
./vendor/bin/phpunit --filter test_apply_discount

# Run specific test class.
./vendor/bin/phpunit --filter Price_CalculatorTest

# Run with code coverage + enforce threshold.
composer test:coverage
composer coverage:check

# Run mutation testing.
XDEBUG_MODE=coverage vendor/bin/infection --threads=max

# Run integration tests inside wp-env.
npx wp-env run tests-cli --env-cwd=wp-content/plugins/nvm-inventory \
    vendor/bin/phpunit -c phpunit-integration.xml
```
