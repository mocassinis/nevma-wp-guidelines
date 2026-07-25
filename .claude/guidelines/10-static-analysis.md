# Static Analysis

> PHPStan configuration, strictness extensions, baseline hygiene, and typing WordPress dynamics.

---

## phpstan.neon

```neon
includes:
	- vendor/szepeviktor/phpstan-wordpress/extension.neon
	- vendor/phpstan/phpstan-strict-rules/rules.neon
	- vendor/phpstan/phpstan-deprecation-rules/rules.neon
	- vendor/phpstan/phpstan-phpunit/extension.neon
	- vendor/phpstan/phpstan-phpunit/rules.neon

parameters:
	level: 8
	paths:
		- src/
		- tests/
	scanFiles:
		# WooCommerce stubs are NOT autodiscovered — wire them explicitly.
		- vendor/php-stubs/woocommerce-stubs/woocommerce-stubs.php
		- vendor/php-stubs/woocommerce-stubs/woocommerce-packages-stubs.php
	bootstrapFiles:
		- tests/bootstrap.php
	reportUnmatchedIgnoredErrors: true
	treatPhpDocTypesAsCertain: false
```

WordPress core stubs come transitively via `szepeviktor/phpstan-wordpress` — only the WooCommerce stubs need explicit `scanFiles`.

---

## Required Dependencies

```json
{
	"require-dev": {
		"phpstan/phpstan": "^2.1",
		"phpstan/phpstan-strict-rules": "^2.0",
		"phpstan/phpstan-deprecation-rules": "^2.0",
		"phpstan/phpstan-phpunit": "^2.0",
		"szepeviktor/phpstan-wordpress": "^2.0",
		"php-stubs/woocommerce-stubs": "^9.0"
	}
}
```

What each extension adds:

| Extension | Catches |
|-----------|---------|
| `phpstan-wordpress` | WP function signatures, hook callback arity, `apply_filters` semantics |
| `phpstan-strict-rules` | Loose comparisons, unhandled booleans in conditions, dynamic property access |
| `phpstan-deprecation-rules` | Calls to `@deprecated` WP/WC APIs — critical for WC version upgrades |
| `phpstan-phpunit` | Wrong assertion argument order, impossible assertions in tests |

---

## Levels

- **New projects: level 8** with strict rules from day one — retrofitting is 10× the cost.
- **Target: level 9/max** once the codebase is stable (`max` adds strictness about mixed).
- **Legacy projects: start at level 6 + baseline**, raise one level at a time.
- Run `composer analyse` before every commit.
- No `@phpstan-ignore` without a trailing comment explaining why.

---

## Baseline Hygiene

For legacy projects only — new projects must not have a baseline.

```bash
# Generate once when adopting PHPStan.
./vendor/bin/phpstan analyse --generate-baseline
```

Rules:

- **The baseline may only shrink, never grow.** New code must be clean; a PR that adds baseline entries is rejected.
- `reportUnmatchedIgnoredErrors: true` makes fixed-but-still-baselined errors fail the run — regenerate to shrink the file when that happens. This is the enforcement mechanism; do not disable it.
- Review the baseline diff in every regeneration commit — a shrinking baseline is progress, a silent regeneration can hide new suppressions.
- Schedule: burn down a few baseline entries whenever touching the file they live in.

---

## Typing WordPress Dynamics

PHPStan sees `get_option()` and `apply_filters()` as `mixed`. Pin the types at the boundary:

### Filtered Values

```php
/** @var int $threshold */
$threshold = apply_filters( 'nvm/inventory/low_stock_threshold', 5 );
```

Better: wrap the filter in a typed accessor so the cast lives in one place:

```php
public function get_low_stock_threshold(): int {
	return (int) apply_filters( 'nvm/inventory/low_stock_threshold', 5 );
}
```

### Settings Arrays (Array Shapes)

```php
/**
 * @return array{low_stock_threshold: int, sync_enabled: bool, api_url: string}
 */
public function get_settings(): array {
	$defaults = [
		'low_stock_threshold' => 5,
		'sync_enabled'        => false,
		'api_url'             => '',
	];

	$stored = get_option( Plugin::PREFIX . 'settings', [] );

	return wp_parse_args( is_array( $stored ) ? $stored : [], $defaults );
}
```

Every consumer of `get_settings()` now gets key-level type checking instead of `mixed`.

### Lists and Maps

```php
/** @return list<\WC_Product> */
/** @return array<int, string>  Map of product ID => SKU. */
/** @param array<string, mixed> $args */
```

Prefer `list<T>` over `array<T>` when the array is sequential — strict rules will catch key misuse.

---

## Running Analysis

```bash
# Run PHPStan.
composer analyse

# Run with increased memory (for large codebases).
./vendor/bin/phpstan analyse --memory-limit=512M

# Clear result cache after changing stubs/config.
./vendor/bin/phpstan clear-result-cache
```
