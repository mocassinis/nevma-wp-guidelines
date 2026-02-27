# New Plugin Workflow

> Step-by-step guide for creating a new WordPress/WooCommerce plugin.

---

## Step 1: Register Plugin Slug

**Read**: `01-technical-setup.md`

Before writing any code, register the plugin in the slug registry to avoid collisions.

| Decision | Example |
|----------|---------|
| Plugin folder name | `nvm-inventory` |
| Short slug (3-5 chars) | `INV` |
| Namespace | `NVM\Inventory` |
| Global prefix | `NVM_INV_` |
| Class prefix | `nvm_inv_` |

Add to the registry table in `01-technical-setup.md`.

---

## Step 2: Create Directory Structure

**Read**: `02-architecture.md`

Create the folder structure:

```bash
mkdir -p nvm-{plugin-name}/{src/{Admin,Frontend,REST,Services,Contracts},assets/{css,js,images},templates/admin,tests/Unit,languages}
```

Result:
```
nvm-{plugin-name}/
├── nvm-{plugin-name}.php
├── src/
│   ├── Plugin.php
│   ├── Admin/
│   ├── Frontend/
│   ├── REST/
│   ├── Services/
│   └── Contracts/
├── assets/
│   ├── css/
│   ├── js/
│   └── images/
├── templates/
├── tests/
├── languages/
├── composer.json
├── phpunit.xml
├── phpstan.neon
├── build.sh
└── build.ps1
```

---

## Step 3: Create composer.json

**Read**: `02-architecture.md` (composer.json section)

```json
{
	"name": "nevma/nvm-{plugin-name}",
	"description": "{Description}",
	"type": "wordpress-plugin",
	"license": "GPL-2.0-or-later",
	"require": {
		"php": ">=8.0"
	},
	"require-dev": {
		"phpunit/phpunit": "^10.5",
		"brain/monkey": "^2.6",
		"yoast/phpunit-polyfills": "^3.0",
		"mockery/mockery": "^1.6",
		"phpstan/phpstan": "^2.1",
		"szepeviktor/phpstan-wordpress": "^2.0",
		"php-stubs/woocommerce-stubs": "^9.0"
	},
	"autoload": {
		"psr-4": {
			"NVM\\{Plugin_Namespace}\\": "src/"
		}
	},
	"autoload-dev": {
		"psr-4": {
			"NVM\\{Plugin_Namespace}\\Tests\\": "tests/"
		}
	},
	"scripts": {
		"test": "phpunit",
		"test:unit": "phpunit --testsuite unit",
		"analyse": "phpstan analyse --memory-limit=512M",
		"build": "composer install --no-dev --optimize-autoloader --classmap-authoritative"
	}
}
```

Run: `composer install`

---

## Step 4: Create Main Plugin File

**Read**: `02-architecture.md` (Main Plugin File Pattern)

Create `nvm-{plugin-name}.php`:

```php
<?php
/**
 * Plugin Name: NVM {Plugin Name}
 * Description: {Description}
 * Version:     1.0.0
 * Author:      nevma
 * Author URI:  https://nevma.gr
 * Text Domain: nvm-{plugin-name}
 * Requires PHP: 8.0
 *
 * WC requires at least: 8.0
 * WC tested up to: 9.6
 */

declare(strict_types=1);

if ( ! defined( 'ABSPATH' ) ) {
	exit;
}

define( 'NVM_{SLUG}_FILE', __FILE__ );
define( 'NVM_{SLUG}_PATH', plugin_dir_path( __FILE__ ) );

require_once NVM_{SLUG}_PATH . 'vendor/autoload.php';

NVM\{Namespace}\Plugin::instance();
```

---

## Step 5: Create Plugin Class

**Read**: `02-architecture.md` (Plugin Class Pattern)

Create `src/Plugin.php` with:
- Class constants: `VERSION`, `SLUG`, `PREFIX`, `WC_MIN_VERSION`
- Singleton pattern
- HPOS compatibility declaration
- Dependency checking
- Service container
- Activation/deactivation hooks

---

## Step 6: Create First Service

**Read**: `03-modern-php.md`

Create a service class in `src/Services/`:

```php
<?php
declare(strict_types=1);

namespace NVM\{Namespace}\Services;

if ( ! defined( 'ABSPATH' ) ) {
	exit;
}

/**
 * {Service description}.
 *
 * @since 1.0.0
 */
final class {Name}_Service {

	public function __construct(
		// Dependencies via constructor promotion
	) {}

	// Public methods with full type hints
}
```

Register in `Plugin::init_services()`.

---

## Step 7: Add Security Layer

**Read**: `04-security.md`

For any user-facing functionality:

- [ ] AJAX handlers follow the mandatory pattern (nonce → capability → sanitize → respond)
- [ ] REST endpoints have proper `permission_callback`
- [ ] All inputs sanitized
- [ ] All outputs escaped
- [ ] Nonces use `Plugin::PREFIX`

---

## Step 8: WooCommerce Integration

**Read**: `05-woocommerce.md`

If integrating with WooCommerce:

- [ ] HPOS compatibility declared
- [ ] Using CRUD classes (`wc_get_product`, `wc_get_order`)
- [ ] No direct SQL for orders/products
- [ ] Block checkout considered if extending checkout
- [ ] Logging uses `wc_get_logger()`

---

## Step 9: Performance Optimization

**Read**: `06-performance.md`

Before shipping:

- [ ] Assets load conditionally (not globally)
- [ ] Heavy tasks use Action Scheduler
- [ ] Expensive operations cached with `Plugin::PREFIX`
- [ ] All queries have `limit` set
- [ ] Cache invalidation on data changes

---

## Step 10: Add JavaScript

**Read**: `07-javascript.md`

- Frontend: Vanilla JS with `fetch()`
- Admin: jQuery acceptable
- Data: `wp_add_inline_script()` for config, `wp_localize_script()` for i18n only

---

## Step 11: Write Tests

**Read**: `09-testing.md`

Create test infrastructure:

1. `tests/bootstrap.php`
2. `tests/Unit_Test_Case.php`
3. `phpunit.xml`

Write tests for:
- [ ] All service class public methods
- [ ] AJAX handlers (security paths)
- [ ] REST controllers (permissions)
- [ ] Edge cases (zero, negative, null, empty)

Run: `composer test`

---

## Step 12: Static Analysis

**Read**: `10-static-analysis.md`

Create `phpstan.neon` and run:

```bash
composer analyse
```

Fix all errors before shipping.

---

## Step 13: Final Checklist

**Read**: `11-checklist.md`

Go through every item before committing.

---

## Quick Commands

```bash
# Install dependencies
composer install

# Run tests
composer test

# Run static analysis
composer analyse

# Build for production (creates dist/ folder with zip)
./build.sh
```

---

## Files to Create (Summary)

| Order | File | Template In |
|-------|------|-------------|
| 1 | `composer.json` | `02-architecture.md` |
| 2 | `nvm-{name}.php` | `02-architecture.md` |
| 3 | `src/Plugin.php` | `02-architecture.md` |
| 4 | `src/Services/*.php` | `03-modern-php.md` |
| 5 | `src/Admin/*.php` | `04-security.md` |
| 6 | `src/REST/*.php` | `04-security.md` |
| 7 | `assets/js/*.js` | `07-javascript.md` |
| 8 | `tests/bootstrap.php` | `09-testing.md` |
| 9 | `tests/Unit_Test_Case.php` | `09-testing.md` |
| 10 | `phpunit.xml` | `09-testing.md` |
| 11 | `phpstan.neon` | `10-static-analysis.md` |
| 12 | `build.sh` | `02-architecture.md` |
| 13 | `build.ps1` | `02-architecture.md` |
