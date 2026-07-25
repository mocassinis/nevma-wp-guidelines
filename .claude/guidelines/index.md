# Guidelines Index

Quick reference for which guideline to load based on your task.

## By Task Type

| What You're Doing | Files to Read |
|-------------------|---------------|
| **Starting a new plugin** | `00-new-plugin-workflow.md` (full walkthrough) |
| **Adding a new class/service** | `03-modern-php.md` |
| **Writing AJAX handler** | `04-security.md` (AJAX section) |
| **Creating REST endpoint** | `04-security.md` (REST section) |
| **Working with forms** | `04-security.md` (nonces, sanitization) |
| **Custom database queries** | `04-security.md` (SQL section), `06-performance.md` |
| **WooCommerce products/orders** | `05-woocommerce.md` |
| **Adding caching** | `06-performance.md` |
| **Background/async tasks** | `06-performance.md` (Action Scheduler) |
| **Frontend JavaScript** | `07-javascript.md` (vanilla JS section) |
| **Admin JavaScript** | `07-javascript.md` (jQuery section) |
| **Block interactivity** | `15-interactivity-api.md` |
| **Local testing/demos** | `16-playground.md` |
| **Writing unit tests** | `09-testing.md` |
| **Integration tests (wp-phpunit)** | `09-testing.md` (Integration section) |
| **Coverage / mutation testing** | `09-testing.md` (Coverage & Infection sections) |
| **E2E tests (Playwright)** | `14-e2e-testing.md` |
| **Accessibility testing** | `14-e2e-testing.md` (axe section) |
| **Setting up PHPStan** | `10-static-analysis.md` |
| **CI/CD, PHPCS, quality gates** | `17-quality-gates.md` |
| **Automation & Scoping** | `13-automation-tooling.md` |
| **Pre-commit review** | `11-checklist.md` |

## By File Count

| File | Lines | Description |
|------|-------|-------------|
| `00-new-plugin-workflow.md` | ~250 | Step-by-step new plugin creation guide |
| `01-technical-setup.md` | ~80 | PHP version, naming conventions, plugin registry |
| `02-architecture.md` | ~400 | Directory structure, Plugin class, composer.json |
| `03-modern-php.md` | ~180 | PHP 8.0/8.1/8.2 features, strict typing |
| `04-security.md` | ~350 | Input/output, nonces, AJAX, REST, SQL |
| `05-woocommerce.md` | ~880 | CRUD, HPOS + block compatibility, gateways (classic + block), checkout fields, shipping, emails, settings, webhooks, logging |
| `06-performance.md` | ~715 | Caching, Action Scheduler, query optimization, HTTP API, profiling |
| `07-javascript.md` | ~180 | Vanilla JS, jQuery admin, data passing |
| `08-documentation.md` | ~40 | PHPDoc standards |
| `09-testing.md` | ~1100 | PHPUnit, Brain Monkey patterns, integration tests (wp-phpunit), coverage gates, mutation testing |
| `10-static-analysis.md` | ~150 | PHPStan level 8, strict/deprecation rules, baseline hygiene, typing WP dynamics |
| `11-checklist.md` | ~145 | Pre-generation verification with mandatory test execution |
| `12-advanced-patterns.md` | ~200 | DTOs, Value Objects, RBAC, Middleware |
| `13-automation-tooling.md` | ~100 | Composer scoping (PHP-Scoper), phpcbf, Prettier |
| `14-e2e-testing.md` | ~870 | Playwright E2E, Store API tests, accessibility (axe), visual regression |
| `15-interactivity-api.md` | ~280 | WordPress Interactivity API, directives, stores |
| `16-playground.md` | ~250 | WordPress Playground CLI, blueprints, testing |
| `17-quality-gates.md` | ~240 | PHPCS + PHPCompatibilityWP, CI pipeline, Plugin Check, QIT, release gate |

## Common Combinations

**Full new plugin**: 01 → 02 → 03 → 11

**Adding secure endpoint**: 04 → 06 → 11

**Adding tested feature**: 03 → 09 → 11
