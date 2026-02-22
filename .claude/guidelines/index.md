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
| **Writing tests** | `09-testing.md` |
| **Setting up PHPStan** | `10-static-analysis.md` |
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
| `05-woocommerce.md` | ~90 | CRUD, HPOS, block checkout, logging |
| `06-performance.md` | ~150 | Caching, Action Scheduler, query optimization |
| `07-javascript.md` | ~180 | Vanilla JS, jQuery admin, data passing |
| `08-documentation.md` | ~40 | PHPDoc standards |
| `09-testing.md` | ~550 | PHPUnit setup, test patterns, Brain Monkey |
| `10-static-analysis.md` | ~50 | PHPStan configuration |
| `11-checklist.md` | ~60 | Pre-generation verification |
| `12-advanced-patterns.md` | ~200 | DTOs, Value Objects, RBAC, Middleware |
| `13-automation-tooling.md` | ~100 | Composer scoping (PHP-Scoper), phpcbf, Prettier |

## Common Combinations

**Full new plugin**: 01 → 02 → 03 → 11

**Adding secure endpoint**: 04 → 06 → 11

**Adding tested feature**: 03 → 09 → 11
