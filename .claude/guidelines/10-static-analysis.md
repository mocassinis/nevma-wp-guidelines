# Static Analysis

> PHPStan configuration for WordPress/WooCommerce projects.

---

## phpstan.neon

```neon
includes:
	- vendor/szepeviktor/phpstan-wordpress/extension.neon

parameters:
	level: 6
	paths:
		- src/
	scanDirectories:
		- vendor/php-stubs/
	bootstrapFiles:
		- tests/bootstrap.php
	ignoreErrors:
		# WordPress global functions that PHPStan can't fully resolve.
		- '#Function wc_get_product invoked with#'
	treatPhpDocTypesAsCertain: false
```

---

## Rules

- **Minimum level**: 6 (strict property types, return types).
- **Target level**: 8 for new projects.
- Run `composer analyse` before every commit.
- No `@phpstan-ignore` without a comment explaining why.

---

## Running Analysis

```bash
# Run PHPStan.
composer analyse

# Run with increased memory (for large codebases).
./vendor/bin/phpstan analyse --memory-limit=512M

# Generate baseline (for existing projects).
./vendor/bin/phpstan analyse --generate-baseline
```

---

## Required Dependencies

In `composer.json`:

```json
{
	"require-dev": {
		"phpstan/phpstan": "^2.1",
		"szepeviktor/phpstan-wordpress": "^2.0",
		"php-stubs/woocommerce-stubs": "^9.0"
	}
}
```
