# Testing

> PHPUnit setup, Brain Monkey, test patterns, and coverage requirements.

Testing is **mandatory** for all service classes and business logic.

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

# Run with code coverage.
composer test:coverage
```
