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
| Block interactivity (data-wp-*) | `15-interactivity-api.md` |
| Local testing with Playground | `16-playground.md` |
| PHPDoc standards | `08-documentation.md` |
| Writing unit tests | `09-testing.md` |
| PHPStan configuration | `10-static-analysis.md` |
| Advanced patterns (DTOs, CLI) | `12-advanced-patterns.md` |
| E2E testing (Playwright) | `14-e2e-testing.md` |
| Before commit/PR | `11-checklist.md` |

## Core Rules (Always Apply)

- **PHP 8.0+** minimum with `declare(strict_types=1)` in every file
- **Namespace**: `NVM\{Plugin}\` for all classes
- **Hooks**: `nvm/{plugin-slug}/` prefix
- **Security**: Sanitize input, escape output, verify nonces, check capabilities
- **No jQuery on frontend** - use vanilla JS with `fetch()`
- **Performance first**: Always set query limits, cache expensive operations, use Action Scheduler for heavy tasks (see `06-performance.md`)

## Mandatory Testing (Always Enforce)

**No code is complete until tests pass.** This is non-negotiable.

### Unit Tests (PHPUnit + Brain Monkey)

Run after writing or modifying any service, handler, controller, or business logic:

```bash
composer test          # All tests
composer test:unit     # Unit tests only
```

- Every public method must have tests
- Every bug fix must start with a failing test (see `09-testing.md`)
- Tests must cover happy path AND negative/edge cases
- **Do not commit code with failing tests**

### E2E Tests (Playwright)

Run after completing any user-facing feature or checkout/cart/admin flow:

```bash
npm run test:e2e       # All E2E tests (headless)
npm run test:e2e:headed # With browser visible
```

- Shopper flows: checkout, cart, product pages
- Merchant flows: admin settings, order management
- API tests: REST endpoint validation
- **Ask user before running E2E tests** (they modify data in the test environment)

### Test Execution Workflow

```
Write Code → Run Unit Tests → Run E2E Tests → Security Audit → Performance Review → Commit
```

**Before any commit or PR:**
1. `composer test` — all unit tests must pass
2. `npx playwright test` — all E2E tests must pass (if E2E tests exist)
3. `vendor/bin/phpstan analyse` — static analysis must pass

## Specialized Agents (Auto-Triggered)

Three specialized agents review code automatically. Use them proactively after writing significant code.

### 1. Security Auditor (`wordpress-security-auditor`)

**Trigger after writing:**
- AJAX handlers
- REST API endpoints
- Form processors
- Database queries (raw SQL)
- File upload handlers
- Any code handling `$_GET`, `$_POST`, `$_REQUEST`

**Checks against:** `04-security.md` rules (nonces, capabilities, sanitization, escaping, SQL injection)

### 2. Performance Optimizer (`wordpress-performance-optimizer`)

**Trigger after writing:**
- Database queries or `wc_get_products()`/`wc_get_orders()` calls
- Loops over products, orders, or users
- Report generation or data aggregation
- Any operation that might take > 2 seconds

**Checks against:** `06-performance.md` rules (caching, Action Scheduler, query limits, batching)

### 3. Unit Test Writer (`wp-unit-test-writer`)

**Trigger after completing:**
- Service classes with business logic
- AJAX handlers
- REST controllers
- Enums with methods
- Any class with public methods that can be unit tested

**Follows:** `09-testing.md` patterns (Brain Monkey, negative path testing, data providers)

### Workflow

```
Write Code → Security Audit → Performance Review → Write Tests → Commit
```

## Author Info

When creating WordPress plugins:
- Author: nevma
- Author URI: https://nevma.gr
