# Nevma WordPress/WooCommerce Guidelines

Strict coding standards for AI-generated WordPress/WooCommerce code.

## Quick Start

Before starting any task, read the relevant guideline from `.claude/guidelines/`:

| Task | Read |
|------|------|
| **New plugin setup** | `00-new-plugin-workflow.md` (complete walkthrough) |
| Writing PHP classes | `03-modern-php.md` |
| Security (AJAX, REST, forms, SQL) | `04-security.md` |
| WooCommerce integration | `05-woocommerce.md` |
| Caching, async tasks, queries | `06-performance.md` |
| Frontend/Admin JavaScript | `07-javascript.md` |
| PHPDoc standards | `08-documentation.md` |
| Writing unit tests | `09-testing.md` |
| PHPStan configuration | `10-static-analysis.md` |
| Before commit/PR | `11-checklist.md` |

## Core Rules (Always Apply)

- **PHP 8.0+** minimum with `declare(strict_types=1)` in every file
- **Namespace**: `NVM\{Plugin}\` for all classes
- **Hooks**: `nvm/{plugin-slug}/` prefix
- **Security**: Sanitize input, escape output, verify nonces, check capabilities
- **No jQuery on frontend** - use vanilla JS with `fetch()`

## Author Info

When creating WordPress plugins:
- Author: nevma
- Author URI: https://nevma.gr
