# Class Architecture

> Directory structure, plugin patterns, and dependency injection.

---

## Directory Structure

```
nvm-inventory/
├── nvm-inventory.php          # Bootstrap only (minimal constants, autoloader, instance)
├── src/                       # All PHP classes (PSR-4 root)
│   ├── Plugin.php             # Main plugin class (singleton, class constants)
│   ├── Admin/
│   │   └── Settings.php
│   ├── Frontend/
│   │   └── Assets.php
│   ├── REST/
│   │   └── Stock_Controller.php
│   ├── Services/
│   │   └── Stock_Service.php
│   ├── Contracts/
│   │   └── Service_Interface.php
│   └── Enums/                 # PHP 8.1+
│       └── Stock_Status.php
├── assets/                    # Static assets (CSS, JS, images)
│   ├── css/
│   │   └── admin.css
│   ├── js/
│   │   ├── admin.js
│   │   └── frontend.js
│   └── images/
├── templates/                 # PHP template files (if needed)
│   └── admin/
│       └── settings-page.php
├── tests/
│   ├── bootstrap.php
│   ├── Unit_Test_Case.php
│   ├── Integration_Test_Case.php
│   └── Unit/
│       └── Services/
│           └── Stock_ServiceTest.php
├── languages/
├── vendor/                    # Composer autoloader
├── composer.json
├── phpunit.xml
├── phpstan.neon
├── build.sh                   # Production build script (Mac/Linux)
└── build.ps1                  # Production build script (Windows)
```

---

## Architecture Rules

- **Always use classes** with `NVM\{Plugin}\` namespace — avoid procedural functions in global scope.
- Main plugin file should **only**: define 2 global constants (FILE, PATH), require autoloader, instantiate `Plugin`.
- Group classes by purpose: `Admin/`, `Frontend/`, `REST/`, `Services/`, `Contracts/`, `Enums/`.
- Static assets live in `assets/` (CSS, JS, images) — never in `src/`.
- PHP templates (if needed) live in `templates/` — never in `src/`.
- Use **dependency injection** for services — singleton only for main `Plugin` class.
- **Constants strategy**: Minimal globals (FILE, PATH) + class constants for everything else.
- **Enums** (PHP 8.1+): Use backed enums for fixed sets (statuses, types). Fall back to class constants if 8.0 is the target.

---

## Main Plugin File Pattern

```php
<?php
/**
 * Plugin Name: NVM Inventory
 * Description: Inventory management for WooCommerce.
 * Version:     1.0.0
 * Author:      Nevma
 * Text Domain: nvm-inventory
 * Requires PHP: 8.0
 *
 * WC requires at least: 8.0
 * WC tested up to: 9.6
 */

declare(strict_types=1);

if ( ! defined( 'ABSPATH' ) ) {
	exit;
}

/**
 * Global constants — ONLY what's needed before autoloader.
 * All other constants live in the Plugin class.
 *
 * Naming: NVM_{SLUG}_{NAME}
 * Slug reference: INV = nvm-inventory (see Section 1 registry)
 */
define( 'NVM_INV_FILE', __FILE__ );
define( 'NVM_INV_PATH', plugin_dir_path( __FILE__ ) );

// Load Composer autoloader.
require_once NVM_INV_PATH . 'vendor/autoload.php';

// Boot the plugin.
NVM\Inventory\Plugin::instance();
```

---

## Plugin Class Pattern

```php
<?php
declare(strict_types=1);

namespace NVM\Inventory;

if ( ! defined( 'ABSPATH' ) ) {
	exit;
}

/**
 * Main plugin class.
 *
 * @since 1.0.0
 */
final class Plugin {

	/**
	 * Plugin version.
	 */
	public const VERSION = '1.0.0';

	/**
	 * Plugin slug for hooks and text domain.
	 */
	public const SLUG = 'nvm-inventory';

	/**
	 * Short prefix for database options, transients, meta keys, nonces.
	 */
	public const PREFIX = 'nvm_inv_';

	/**
	 * Minimum WooCommerce version required.
	 */
	public const WC_MIN_VERSION = '8.0';

	/**
	 * Singleton instance.
	 */
	private static ?self $instance = null;

	/**
	 * Service container for injected dependencies.
	 *
	 * @var array<string, object>
	 */
	private array $services = [];

	/**
	 * Get plugin instance.
	 *
	 * @since 1.0.0
	 */
	public static function instance(): self {
		if ( null === self::$instance ) {
			self::$instance = new self();
		}
		return self::$instance;
	}

	/**
	 * Get plugin URL.
	 *
	 * @since 1.0.0
	 */
	public static function url( string $path = '' ): string {
		return plugin_dir_url( NVM_INV_FILE ) . ltrim( $path, '/' );
	}

	/**
	 * Get plugin path.
	 *
	 * @since 1.0.0
	 */
	public static function path( string $path = '' ): string {
		return NVM_INV_PATH . ltrim( $path, '/' );
	}

	/**
	 * Get a registered service.
	 *
	 * @since 1.0.0
	 *
	 * @template T of object
	 * @param class-string<T> $class_name Service class name.
	 * @return T
	 * @throws \RuntimeException If service is not registered.
	 */
	public function get_service( string $class_name ): object {
		if ( ! isset( $this->services[ $class_name ] ) ) {
			throw new \RuntimeException(
				sprintf( 'Service %s is not registered.', $class_name )
			);
		}
		return $this->services[ $class_name ];
	}

	/**
	 * Constructor.
	 */
	private function __construct() {
		$this->register_hooks();
	}

	/** Prevent cloning. */
	private function __clone() {}

	/**
	 * Register WordPress hooks.
	 *
	 * @since 1.0.0
	 */
	private function register_hooks(): void {
		// Declare HPOS compatibility.
		add_action( 'before_woocommerce_init', [ $this, 'declare_hpos_compatibility' ] );

		// Initialize plugin after all plugins loaded.
		add_action( 'plugins_loaded', [ $this, 'init' ] );

		// Register activation/deactivation hooks.
		register_activation_hook( NVM_INV_FILE, [ $this, 'activate' ] );
		register_deactivation_hook( NVM_INV_FILE, [ $this, 'deactivate' ] );
	}

	/**
	 * Declare HPOS compatibility.
	 *
	 * @since 1.0.0
	 */
	public function declare_hpos_compatibility(): void {
		if ( class_exists( \Automattic\WooCommerce\Utilities\FeaturesUtil::class ) ) {
			\Automattic\WooCommerce\Utilities\FeaturesUtil::declare_compatibility(
				'custom_order_tables',
				NVM_INV_FILE,
				true
			);
		}
	}

	/**
	 * Initialize plugin.
	 *
	 * @since 1.0.0
	 */
	public function init(): void {
		if ( ! $this->check_dependencies() ) {
			return;
		}

		load_plugin_textdomain(
			self::SLUG,
			false,
			dirname( plugin_basename( NVM_INV_FILE ) ) . '/languages'
		);

		$this->init_services();
		$this->init_components();

		/**
		 * Fires when plugin is fully loaded.
		 *
		 * @since 1.0.0
		 *
		 * @param Plugin $plugin Plugin instance.
		 */
		do_action( 'nvm/inventory/loaded', $this );
	}

	/**
	 * Check plugin dependencies.
	 *
	 * @since 1.0.0
	 */
	private function check_dependencies(): bool {
		if ( ! class_exists( 'WooCommerce' ) ) {
			add_action( 'admin_notices', [ $this, 'notice_woocommerce_missing' ] );
			return false;
		}

		if ( version_compare( WC_VERSION, self::WC_MIN_VERSION, '<' ) ) {
			add_action( 'admin_notices', [ $this, 'notice_woocommerce_version' ] );
			return false;
		}

		return true;
	}

	/**
	 * Register services (dependency injection).
	 *
	 * @since 1.0.0
	 */
	private function init_services(): void {
		$this->services[ Services\Stock_Service::class ] = new Services\Stock_Service();
	}

	/**
	 * Initialize plugin components that hook into WordPress.
	 *
	 * @since 1.0.0
	 */
	private function init_components(): void {
		if ( is_admin() ) {
			new Admin\Settings(
				$this->get_service( Services\Stock_Service::class )
			);
		}

		// REST API endpoints load on both admin and frontend.
		new REST\Stock_Controller(
			$this->get_service( Services\Stock_Service::class )
		);
	}

	/**
	 * Plugin activation.
	 *
	 * @since 1.0.0
	 */
	public function activate(): void {
		// Store version for upgrade routines.
		update_option( self::PREFIX . 'version', self::VERSION );

		/**
		 * Fires on plugin activation.
		 *
		 * @since 1.0.0
		 */
		do_action( 'nvm/inventory/activated' );
	}

	/**
	 * Plugin deactivation.
	 *
	 * @since 1.0.0
	 */
	public function deactivate(): void {
		// Clean up scheduled actions.
		as_unschedule_all_actions( 'nvm/inventory/sync', [], 'nvm-inventory' );

		/**
		 * Fires on plugin deactivation.
		 *
		 * @since 1.0.0
		 */
		do_action( 'nvm/inventory/deactivated' );
	}

	/**
	 * Admin notice: WooCommerce missing.
	 *
	 * @since 1.0.0
	 */
	public function notice_woocommerce_missing(): void {
		?>
		<div class="notice notice-error">
			<p>
				<?php
				printf(
					/* translators: %s: Plugin name */
					esc_html__( '%s requires WooCommerce to be installed and active.', 'nvm-inventory' ),
					'<strong>NVM Inventory</strong>'
				);
				?>
			</p>
		</div>
		<?php
	}

	/**
	 * Admin notice: WooCommerce version too low.
	 *
	 * @since 1.0.0
	 */
	public function notice_woocommerce_version(): void {
		?>
		<div class="notice notice-error">
			<p>
				<?php
				printf(
					/* translators: %1$s: Plugin name, %2$s: Required WC version */
					esc_html__( '%1$s requires WooCommerce %2$s or higher.', 'nvm-inventory' ),
					'<strong>NVM Inventory</strong>',
					esc_html( self::WC_MIN_VERSION )
				);
				?>
			</p>
		</div>
		<?php
	}
}
```

---

## composer.json

```json
{
	"name": "nevma/nvm-inventory",
	"description": "Inventory management for WooCommerce",
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
			"NVM\\Inventory\\": "src/"
		}
	},
	"autoload-dev": {
		"psr-4": {
			"NVM\\Inventory\\Tests\\": "tests/"
		}
	},
	"config": {
		"optimize-autoloader": true,
		"sort-packages": true,
		"allow-plugins": {
			"dealerdirect/phpcodesniffer-composer-installer": true
		}
	},
	"scripts": {
		"test": "phpunit",
		"test:unit": "phpunit --testsuite unit",
		"test:integration": "phpunit --testsuite integration",
		"test:coverage": "phpunit --coverage-html coverage",
		"analyse": "phpstan analyse --memory-limit=512M",
		"build": "composer install --no-dev --optimize-autoloader --classmap-authoritative"
	}
}
```

---

## Usage Examples

```php
<?php
use NVM\Inventory\Plugin;

// Access class constants.
$version = Plugin::VERSION;        // '1.0.0'
$slug    = Plugin::SLUG;           // 'nvm-inventory'
$prefix  = Plugin::PREFIX;         // 'nvm_inv_'

// Get paths with optional appended path.
$css_url  = Plugin::url( 'assets/css/admin.css' );
$template = Plugin::path( 'templates/admin/settings.php' );

// Database options use the prefix.
$settings = get_option( Plugin::PREFIX . 'settings', [] );
update_option( Plugin::PREFIX . 'last_sync', time() );

// Transients use the prefix.
$cache_key = Plugin::PREFIX . 'products_cache';
$data      = get_transient( $cache_key );

// Hooks use the full slug.
do_action( 'nvm/inventory/stock_updated', $product_id, $new_stock );
$filtered = apply_filters( 'nvm/inventory/api_response', $response );

// Services via DI container.
$stock_service = Plugin::instance()->get_service( Services\Stock_Service::class );
```

---

## Build Script

Create `build.sh` in the plugin root to generate a production-ready distribution folder.

```bash
#!/bin/bash
#
# Build script for NVM Inventory plugin.
# Creates a clean dist/ folder with only production files.
#

set -e

PLUGIN_SLUG="nvm-inventory"
BUILD_DIR="dist"
PLUGIN_DIR="${BUILD_DIR}/${PLUGIN_SLUG}"

echo "Building ${PLUGIN_SLUG}..."

# Clean previous build.
rm -rf "${BUILD_DIR}"
mkdir -p "${PLUGIN_DIR}"

# Copy production files.
cp -r src "${PLUGIN_DIR}/"
cp -r assets "${PLUGIN_DIR}/"
cp -r templates "${PLUGIN_DIR}/" 2>/dev/null || true
cp -r languages "${PLUGIN_DIR}/" 2>/dev/null || true
cp "${PLUGIN_SLUG}.php" "${PLUGIN_DIR}/"
cp composer.json "${PLUGIN_DIR}/"
cp composer.lock "${PLUGIN_DIR}/" 2>/dev/null || true

# Install production dependencies only.
cd "${PLUGIN_DIR}"
composer install --no-dev --optimize-autoloader --classmap-authoritative --quiet
rm -f composer.json composer.lock
cd - > /dev/null

# Create zip archive.
cd "${BUILD_DIR}"
zip -rq "${PLUGIN_SLUG}.zip" "${PLUGIN_SLUG}"
cd - > /dev/null

echo "Build complete: ${BUILD_DIR}/${PLUGIN_SLUG}.zip"
```

Make it executable (Mac/Linux):

```bash
chmod +x build.sh
```

### build.ps1 (Windows)

Create `build.ps1` for native Windows support:

```powershell
#
# Build script for NVM Inventory plugin (Windows).
# Creates a clean dist/ folder with only production files.
#

$ErrorActionPreference = "Stop"

$PLUGIN_SLUG = "nvm-inventory"
$BUILD_DIR = "dist"
$PLUGIN_DIR = "$BUILD_DIR\$PLUGIN_SLUG"

Write-Host "Building $PLUGIN_SLUG..."

# Clean previous build.
if (Test-Path $BUILD_DIR) {
    Remove-Item -Recurse -Force $BUILD_DIR
}
New-Item -ItemType Directory -Path $PLUGIN_DIR -Force | Out-Null

# Copy production files.
Copy-Item -Recurse "src" "$PLUGIN_DIR\"
Copy-Item -Recurse "assets" "$PLUGIN_DIR\"
if (Test-Path "templates") { Copy-Item -Recurse "templates" "$PLUGIN_DIR\" }
if (Test-Path "languages") { Copy-Item -Recurse "languages" "$PLUGIN_DIR\" }
Copy-Item "$PLUGIN_SLUG.php" "$PLUGIN_DIR\"
Copy-Item "composer.json" "$PLUGIN_DIR\"
if (Test-Path "composer.lock") { Copy-Item "composer.lock" "$PLUGIN_DIR\" }

# Install production dependencies only.
Push-Location $PLUGIN_DIR
composer install --no-dev --optimize-autoloader --classmap-authoritative --quiet
Remove-Item -Force "composer.json", "composer.lock" -ErrorAction SilentlyContinue
Pop-Location

# Create zip archive.
Compress-Archive -Path "$PLUGIN_DIR" -DestinationPath "$BUILD_DIR\$PLUGIN_SLUG.zip" -Force

Write-Host "Build complete: $BUILD_DIR\$PLUGIN_SLUG.zip"
```

Run from PowerShell:

```powershell
.\build.ps1
```

> **Note:** If you get an execution policy error, run: `Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser`

### What Gets Excluded

The script explicitly copies only production files, automatically excluding:

- `tests/` — Unit and integration tests
- `phpunit.xml` — PHPUnit configuration
- `phpstan.neon` — Static analysis configuration
- `.git/` — Version control
- `.github/` — GitHub workflows
- `node_modules/` — Node dependencies
- `.env` — Environment files
- `build.sh` — The build script itself (Mac/Linux)
- `build.ps1` — The build script itself (Windows)
- Dev dependencies in `vendor/`

### Directory Structure After Build

```
dist/
├── nvm-inventory/
│   ├── nvm-inventory.php
│   ├── src/
│   ├── assets/
│   ├── templates/
│   ├── languages/
│   └── vendor/          # Production dependencies only
└── nvm-inventory.zip    # Ready to upload
```
