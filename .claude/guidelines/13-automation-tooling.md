# Automation & Tooling

Standardizing code quality and dependency management through automation.

## 1. Composer Scoping (PHP-Scoper)

In WordPress, multiple plugins can load different versions of the same library (e.g., Guzzle). To prevent "dependency hell" and fatal errors, all third-party dependencies must be prefixed (scoped).

### Implementation
- **Tool**: `humbug/php-scoper`
- **Workflow**:
    1. Install dependencies via Composer to a temporary folder (e.g., `vendor-src`).
    2. Run `php-scoper` to prefix all namespaces in `vendor-src` with `NVM\{Plugin}\Vendor`.
    3. Dump the autoloader for the newly prefixed files.

### scoper.inc.php Configuration
Ensure you don't prefix WordPress or WooCommerce core functions:

```php
return [
    'prefix' => 'NVM\MyPlugin\Vendor',
    'expose-global-constants' => true,
    'expose-global-functions' => true,
    'exclude-namespaces' => [
        'WP',
        'Automattic\WooCommerce',
    ],
];
```

## 2. Auto-fixers

Manual formatting is a waste of time. Use automated tools to enforce the standards defined in `01-technical-setup.md`.

### PHP Code Beautifier (phpcbf)
Use `phpcbf` to automatically fix WordPress Coding Standards (WPCS) violations.

- **Command**: `composer fix:php`
- **Configuration**: Use the same `phpcs.xml.dist` used for static analysis.

### Prettier
Use Prettier for JavaScript, CSS, and JSON files to ensure consistent spacing and syntax.

- **Command**: `npm run format`
- **Config**: `.prettierrc`
```json
{
    "semi": true,
    "singleQuote": true,
    "tabWidth": 4,
    "trailingComma": "es5"
}
```

## 3. Composer Scripts

Standardize the execution of these tools in `composer.json`:

```json
"scripts": {
    "lint:php": "phpcs .",
    "fix:php": "phpcbf .",
    "scope": "php-scoper add-prefix --force && composer dump-autoload",
    "test": "phpunit",
    "analyze": "phpstan analyze"
}
```

## 4. Pre-commit Hooks
Whenever possible, use `husky` or `pre-commit` to run `lint:php` and `format` before allowing a commit. This ensures the "11-checklist.md" is partially automated.
